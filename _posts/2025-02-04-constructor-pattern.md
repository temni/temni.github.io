---
layout: post
title: Building Smarter Constructors in TON Blockchain Contracts
date: 2025-02-04 20:52 +0100
categories: [funC, TON]
tags: [pattern, smart-contracts]
---
# The Constructor Pattern in TON Blockchain Smart Contracts
In blockchains contracts are normally deployed with a pre‐computed stateInit — a cell prepared off‐chain (or sometimes on‐chain) that completely defines the initial storage and code of the contract.
In TON’s FunC language this `stateInit` is used to compute the contract’s address and to “seed” its persistent storage.

However, in some cases it’s beneficial to implement on‑chain initialization logic - a `constructor`pattern - to:
- Initialize certain data on chain (e.g. fill certain fields with TVM calls results)
- Exclude certain fields from `stateInit`.
- Execute some side effect actions (e.g. sending messages)

In this article we describe one approach to implementing a constructor in a FunC contract, discuss the potential pitfalls (including replay protection), 
and show how to design your storage and message handling to ensure that initialization can only occur once.

# Traditional Deployment with stateInit
Normally, when deploying a FunC contract, you prepare a complete data cell (the stateInit) that contains all the initial storage values. 
The deployment mechanism then uses that cell to initialize your contract’s state. For example, an NFT collection contract 
might have a stateInit that simply stores the owner address, the next item index, and a reference to NFT item code.

This approach is simple, safe and efficient when all data can be prepared off‑chain. 
However, it cannot be applied all the time.

# Why Use a Constructor Pattern?
Consider a scenario when the address of your smartcontract must be evaluated quite often.
For example - a collection's element deployed as an explicit individual contract (to avoid unbound state on collection's side).
Such an element might have a significant amount of data and in case you include everything into `stateInit` - 
you will have to fill it completely each time a contract`s address needs to be calculated, 
when practically a collection address and index of an element should be enough.

It is not possible to evaluate an address with one set of fields and send the different (e.g. extended) one - 
according to [TON whitepaper](https://docs.ton.org/assets/files/tblkch-6aaf006b94ee2843a982ebf21d7c1247.pdf) 
section _1.7.3. Initializing smart contracts by constructor messages_:
```
The hash of the code and of the data contained in the constructor message must coincide with the address η of the
smart contract; otherwise, it is rejected.
```
The same chapter gives us the instructions how to initialize contract further:
```
After the code and data of the smart contract are initialized from the
body of the constructor message, the remainder of the constructor message
is processed by a transaction (the creating transaction for smart contract η)
by invoking TVM in a manner similar to that used for processing ordinary
inbound messages
```
That is exactly what we are going to do!

# Implementing the Constructor Pattern
A common approach is to reserve one particular operation code (op) as the constructor call. 
When your contract receives a message with that op, it executes the initialization routine.

Below is an excerpt from a typical FunC constructor implementation:

```func
#include "stdlib.fc";

;; Constructor for on-chain initialization.
() initialize_contract(slice sender, slice msg_data) impure inline {
    ;; Parse current (`virgin`) storage state from get_data()
    slice ds = get_data().begin_parse();
    ;; Expect the storage to contain only a message address (the owner)
    slice owner = ds~load_msg_addr();
    ds.end_parse();  ;; if extra data is present, throw an exception

    ;; Verify that the sender is the stored owner.
    throw_unless(error::wrong_sender(), equal_slice_bits(sender, owner));

    ;; Load additional initialization data from the message body.
    slice contract_b_address = msg_data~load_msg_addr();
    slice contract_c_address = msg_data~load_msg_addr();
    cell dict_1 = msg_data~load_dict();
    cell dict_2 = msg_data~load_dict();

    ;; Replace the state with a new cell.
    set_data(begin_cell()
        .store_uint(seq_no, 64)
        .store_slice(contract_b_address)
        .store_slice(contract_c_address)
        .store_dict(dict_1)
        .store_dict(dict_2)
        .end_cell();
}

() recv_internal(cell in_msg_full, slice in_msg_body) impure {
    if (in_msg_body.slice_empty?()) {
        ;; ignore empty messages
        return ();
    }

    slice cs = in_msg_full.begin_parse();
    int flags = cs~load_uint(4);

    if (flags & 1) {
        ;; ignore all bounced messages
        return ();
    }
    slice sender_address = cs~load_msg_addr();
    int op = in_msg_body~load_uint(32);

    ;; initialization of contract for the first time
    if (op == 1) {
        cs~load_msg_addr(); ;; skip dst
        cs~load_coins(); ;; skip value
        cs~skip_bits(1); ;; skip extracurrency collection
        cs~load_coins(); ;; skip ihr_fee
        cs~load_coins(); ;; skip fwd_fee
        cs~skip_bits(64 + 32); ;; skip created_lt and created_at
        int init_field_flag = cs~load_uint(1); ;; init-field flag (Maybe)
        ;; only init message can be processed
        throw_unless(error::unrecognized_op(), init_field_flag == 1);
        ;; owner check inside
        initialize_contract(sender_address, in_msg_body);
        return ();
    }
    ;; do something else
}
```
Here the check of the `init_field_flag` is redundant though - it can be faked. 

Corresponding `ts` code for deployment of such a contract if following:
```typescript
import {Address, beginCell, Cell, Contract, contractAddress, ContractProvider, Dictionary, Sender, SendMode} from '@ton/core';

export type MyContractConfig = {
  owner: Address
};

export type TLendDeployParam = {
  contract_b_address: Address,
  contract_c_address: Address,
  set_of_addresses_1?: Address[],
  set_of_addresses_2?: Address[],
};

export function configToCell(config: MyContractConfig): Cell {
  return beginCell()
    .storeAddress(config.owner)
    .endCell();
}

function addressesToDict(addresses: Address[]): Dictionary<Address, number> {
  let dict = Dictionary.empty<Address, number>(Dictionary.Keys.Address(), Dictionary.Values.Int(2));
  for (const addr of addresses) {
    dict.set(addr, 1);
  }
  return dict;
}

export class MyContract implements Contract {

    constructor(readonly address: Address,
                readonly init?: { code: Cell; data: Cell }) {
    }

    static createFromAddress(address: Address) {
        return new MyContract(address);
    }

    static createFromConfig(config: MyContractConfig, code: Cell, workchain = 0) {
        const data = configToCell(config);
        const init = {code, data};
        return new MyContract(contractAddress(workchain, init), init);
    }

    async sendDeploy(provider: ContractProvider, via: Sender, param: MyDeployParam, value?: bigint) {
        await provider.internal(via, {
            value: value ?? GAS_AMOUNT_DEPLOY,
            sendMode: SendMode.PAY_GAS_SEPARATELY,
            body: beginCell()
                .storeUint(1, 32)
                .storeUint(0, 64) // query id is irrelevant
                .storeAddress(param.contract_b_address)
                .storeAddress(param.contract_c_address)
                .storeDict(addressesToDict(param.set_of_addresses_1 ?? []))
                .storeDict(addressesToDict(param.set_of_addresses_2 ?? []))
                .endCell(),
        });
    }
}
```

## How It Works
1. State Check and Sender Verification:
   1. The constructor reads the initial state (using `get_data()`), which should be in a “virgin” format containing only the owner address.
   2. It then calls `ds.end_parse()`. If any extra data exists, an exception is thrown. This guarantees that the contract is uninitialized.
   3. The sender is compared to the stored owner; only the proper account may initialize the contract.
2. Processing Constructor Data:
   1. The contract loads additional initialization parameters from msg_data (e.g., NFT address, wallet, collectors, and operators).
   2. It then calls `store_data(...)` to replace the previous state with a new state that indicates the contract is now initialized (here, the first field is a `seq_no` set to 1).
3. One-Time Execution:
   1. Because the new state does not match the original “virgin” format (which started with a message address), any replay of the constructor message will fail when trying to parse the state.
   2. In effect, the constructor can run only once.



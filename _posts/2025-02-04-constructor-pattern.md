---
layout: post
title: Constructor Pattern in TON Smart Contracts
date: 2025-02-04 20:52 +0100
categories: [funC, TON]
tags: [pattern, constructor, deployment]
---
# The Constructor Pattern: Flexible Initialization for TON Contracts

In TON blockchain development, smart contracts are typically deployed with a predefined
`stateInit` cell containing all initial code and data. 
While this works well for static setups, dynamic initialization scenarios might demand adifferent
approach. Let’s explore how the constructor pattern solves this.
---
## Traditional Deployment: The Limits of `stateInit`
The standard deployment flow uses `stateInit` to compute a contract’s address and seed its storage. 
For example, an NFT collection might initialize with:

```typescript
const initialData = beginCell()
  .storeAddress(ownerAddress)
  .storeUint(nextItemIndex, 64)
  .storeRef(nftItemCode)
  .endCell();

const init = { code, initialData };
const collection = new NftCollection(contractAddress(workchain, init), init);
await nftCollection.sendDeploy(provider.sender());
}
```

**Pros:**
- Straightforward address calculation
- Atomic deployment
- Predictable state

**Cons:**
- Requires all data upfront
- No runtime validation and\or additional actions
- Limited to static configurations

## When Constructors Shine
The constructor pattern becomes essential when:
- Address pre-calculation is needed without full initialization data
- Dynamic values (e.g., blockchain timestamps) must be captured
- Complex validation is required during setup
- Side effects (like sending messages) are needed post-deployment

TON’s whitepaper (§1.7.3) explicitly supports this pattern:

> After the code and data of the smart contract are initialized from the
body of the constructor message, the remainder of the constructor message
is processed by a transaction (the creating transaction for smart contract η)
by invoking TVM in a manner similar to that used for processing ordinary
inbound messages

# Implementing the Constructor Pattern
A common approach is to reserve one particular operation code (op) as the constructor call. 
When your contract receives a message with that op, it executes the initialization routine.

Below is an example of a constructor implementation:

```
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
        ;; owner check inside
        initialize_contract(sender_address, in_msg_body);
        return ();
    }
    ;; do something else
}
```

### Key Security Measures:
1. `ds.end_parse()` ensures no residual data exists
2. Sequence number (`store_uint(1, 64)`) blocks re-initialization

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

## Avoiding Common Pitfalls

### Replay Protection
Code can prevent re-execution through state mutation. 
The initial state (`owner_address` in our case) becomes irreversibly transformed to the desired form.
If the post-deployment state matches the `virgin` one - it makes sense to introduce a specific 1-bit flag
to make sure states don't match (so `ds.end_parse()` would raise an exception).

### Unauthorized Constructor Call Protection 
An actor deploying the contract must pass its own address as a part of `stateInit` data.
In our example this is the only value being passed.
Contract should throw an exception in case sender address does not match a persisted one.
It's not required to store this address back if the contract's logic doesn't need it (ownerless contracts).

### Gas Considerations
Constructor transactions might require a budgeting off-chain.
Even if a deployment initiator pays fees separatly it's required to: 
1. Keep some storage deposit on an account's balance.
2. Manage the balance carefully in case of sending messages.

## Testing Strategy
Is derived from the pitfalls:
- Double initialization attempts
- Invalid sender addresses
- Malformed parameter cells
- Balance issues

## Exceptions In Constructor
In our example we throw exceptions to protect the constructor from attacks only.
Even though deploy message is usually bouncable it can only bounce if the account creation failed.

> Throwing exceptions from constructor method doesn't automatically lead to an account destruction.
{: .prompt-tip }

If you need to implement a validation on-chain in a post-deployment constructor and destroy a smart-contract on the 
validation failure - make sure you send a message back to the originator with `mode = 128` and `flag = 32`.



---
layout: post
title: Gas Consumption Tricks in TON FunC Contracts
date: 2025-02-09 19:52 +0100
---
<script type="text/javascript" src="/assets/highlight.min.js"></script>
<script type="text/javascript" src="/assets/func.min.js"></script>
<script type="text/javascript" src="/assets/typescript.min.js"></script>
<link rel="stylesheet" type="text/css" href="/assets/atom-one-dark.css">

<script
  src="https://code.jquery.com/jquery-3.7.1.min.js"
  integrity="sha256-/JqT3SQfawRcv/BIHPThkBvs0OEvtFFmqPF/lYI/Cxo="
  crossorigin="anonymous"></script>


<script type="text/javascript" src="https://cdn.datatables.net/2.2.2/js/dataTables.min.js"></script>
<link rel="stylesheet" type="text/css" href="https://cdn.datatables.net/2.2.2/css/dataTables.dataTables.min.css">

<script type="text/javascript">
  $(document).ready( function () {
    $('#myTable').DataTable({
      paging: false,
      searching: false
    });
    $('#myTable2').DataTable({
      paging: false,
      searching: false
    });
    hljs.highlightAll();
} );
</script>

In TON smart contract development every nanogram counts. 
Over the course of my experiments, I developed five versions of a FunC counter contract—each 
incorporating different gas optimization techniques. 

In this article, I explain these optimizations and share test results that demonstrate their impact 
on annual gas spendings.

## The Five Versions: Evolution of Gas Optimization

### Version 1 – Baseline Implementation
The [baseline contract](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract.fc) stores three 32‑bit counters (and their three initial values) in a single flat storage cell. 
In the main message handler (`recv_internal`), the contract uses three sequential 
if/else conditions (one for each counter) to process messages. 
This implementation works but is not optimized for gas.

<pre><code class="language-func">if (counter_number == 1) {
    ctx_counter_1 += increase_by;
} elseif (counter_number == 2) {
    ctx_counter_2 += increase_by;
} elseif (counter_number == 3) {
    ctx_counter_3 += increase_by;
} else {
    throw(400); ;; incorrect number is passed
}
</code></pre>

### Version 2 – Reordering Conditions and Grouped Storage

[In Version 2](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract_v2.fc), I reordered the conditional branches in recv_internal based on the daily frequency of counter calls:

- **Counter 1 (C1)**: 1,000 calls/day
- **Counter 2 (C2)**: 2,000 calls/day
- **Counter 3 (C3)**: 3,000 calls/day
Since Counter 3 is invoked three times more often than Counter 1, moving its condition block to the top saves gas by avoiding the evaluation of two extra conditions on most calls. In addition, the storage layout was optimized so that the frequently accessed counter values are grouped at the beginning of the storage cell, reducing the parsing cost.

_Test results_: Version 2 saved roughly **57 TONs** per year compared to [Version 1](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract.fc).

<pre><code class="language-func">if (counter_number == 3) {
    ctx_counter_3 += increase_by;
} elseif (counter_number == 2) {
    ctx_counter_2 += increase_by;
} elseif (counter_number == 1) {
    ctx_counter_1 += increase_by;
} else {
    throw(400); ;; incorrect number is passed
}
</code></pre>

### Version 3 – Enhanced Storage Grouping with Sub‑Cells

[Version 3](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract_v3.fc) refines the storage design further by 
splitting the state into multiple sub‑cells. 
The main cell holds the current counter values, while a separate cell holds the static initial values. 

This design allows to parse the necessary sub‑cell only on demand - when `flush` methods are called or _getter_, 
while keeping this cell unparsed for `increment` calls, reducing gas consumption further.

_Test results_: [Version 3](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract_v3.fc) provided an additional annual saving of about **240 TONs** over [Version 2](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract_v2.fc).

<pre><code class="language-func">() load_data() impure {
  var ds = get_data().begin_parse();
  ctx_counter_1 = ds~load_uint(32);
  ctx_counter_2 = ds~load_uint(32);
  ctx_counter_3 = ds~load_uint(32);
  static = ds~load_ref();
  ds.end_parse();
}
</code></pre>

This is a lot! 

> Try to keep a bare minimum of operations on each flow. If the data is unused - try to not even parse it from the storage.
{: .prompt-tip }

### Version 4 – Composite Counters Using Pure Arithmetic

In [Version 4](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract_v4.fc), I reengineered the design to pack the 
three 32‑bit counters into a single 96‑bit composite value:

<pre><code class="language-func">composite = (A << 64) | (B << 32) | C</code></pre>

By updating a counter with pure arithmetic and bit‑masking—without if‑conditions—the overhead 
from multiple conditional branches is eliminated. 

> Note that once you use pure math the ordering of conditions becomes irrelevant.
{: .prompt-tip }

_Test results_: [Version 4](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract_v4.fc) improves efficiency by approximately **72 TONs** per year compared to [Version 3](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract_v3.fc).

<pre><code class="language-func">(int, int, int) get_counters() method_id {
  load_data();
  ;; A is the most significant 32 bits: shift right by 64 bits.
  int A = composite_counters >> 64;
  ;; B is the middle 32 bits: shift right by 32 and mask out all but 32 bits.
  int B = (composite_counters >> 32) & ((1 << 32) - 1);
  ;; C is the least significant 32 bits.
  int C = composite_counters & ((1 << 32) - 1);
  return (A, B, C);
}
</code></pre>

### Version 5 – Final improvements
The final version ([Version 5](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract_v5.fc)) further reduces gas consumption by:

#### Minimizing Callback Overhead
Callback messages now send back only the bare minimum information to the sender, eliminating the transmission of unnecessary data.
<pre><code class="language-func">;; return unspent grams back
send_raw_message(begin_cell()
    .store_uint(0x18, 6)
    .store_slice(sender_address)
    .store_coins(0)
    .store_uint(0, 107)
    .end_cell(),
    64);
</code></pre>

#### Inlining Storage Functions
All storage helper functions (for reading and writing state) are marked inline. 
This allows the compiler to substitute the function body directly into the calling code, 
avoiding the additional gas cost associated with a `CALLREF`.

<pre><code class="language-func">int load_data() impure inline {</code></pre>
<pre><code class="language-func">() save_counters(int composite) impure inline {</code></pre>

[Official documentation](https://docs.ton.org/v3/documentation/smart-contracts/transaction-fees/fees-low-level) says:
>By default, when you have a FunC function, it gets its own id, stored in a separate leaf of id->function dictionary, and when you call it somewhere in the program, a search of the function in dictionary and subsequent jump occur. Such behavior is justified if your function is called from many places in the code and thus jumps allow to decrease the code size (by storing a function body once). However, if the function is only used once or twice, it is often much cheaper to declare this function as inline or inline_ref. inline modificator places the body of the function right into the code of the parent function, while inline_ref places the function code into the reference (jumping to the reference is still much cheaper than searching and jumping to the dictionary entry).

#### Pure math only
Finally, all counters and their initial values are stored within a single insigned integer.
This is possible because we have 6 variables of type uint32, resulting in 192 bits.

> Max length of insigned integer is 256 bits. If you need more - you will have to operate with a slice instead. Slice also has it`s size restrictions - check [corresponding docs](https://docs.ton.org/v3/documentation/smart-contracts/func/docs/types)
{: .prompt-tip }

### Compiler Comparison: FunC vs. Tolk

I also experimented with converting the final [Version 5](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract_v5.fc) using the [convert-func-to-tolk tool](https://github.com/ton-blockchain/convert-func-to-tolk) to compare 
compiler efficiency. 

The Tolk version, with the [compute-asm-ltr pragma enabled by default](https://docs.ton.org/v3/documentation/smart-contracts/tolk/tolk-vs-func/in-detail), performed slightly 
worse—resulting in an additional cost of about **31.536 TON/Year**. 

In other words, while my optimized FunC code achieves significant savings, the Tolk version’s 
default configuration adds overhead that eliminates some of these gains.

## Test Results: Annual Gas Spending Comparison
The following chart summarizes the annual gas consumption (in TONs) and improvements for each version:
<table id="myTable" class="display">
    <thead>
        <tr>
            <th>Version</th>
            <th>Annual Gas Spending (TONs)</th>
            <th>Improvement vs Previous Version (TONs)</th>
            <th>Cumulative Reduction (%)</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>V1</td>
            <td>10,297.38</td>
            <td>—</td>
            <td>0%</td>
        </tr>
        <tr>
            <td>V2</td>
            <td>~10,240.15</td>
            <td>~57.23 less than V1</td>
            <td>~0.56%</td>
        </tr>
        <tr>
            <td>V3</td>
            <td>~9,999.25</td>
            <td>~240.90 less than V2</td>
            <td>~2.34%</td>
        </tr>
        <tr>
            <td>V4</td>
            <td>~9,927.71</td>
            <td>~71.54 less than V3</td>
            <td>~3.32%</td>
        </tr>
        <tr>
            <td>V5</td>
            <td>~8,253.67</td>
            <td>~1,674.04 less than V4</td>
            <td>~19% overall reduction</td>
        </tr>
    </tbody>
</table>
_Note: These numbers are derived from simulation tests and extrapolations based on the daily frequencies of counter operations (C1: 1,000/day; C2: 2,000/day; C3: 3,000/day)._

## Own contract size
Let's also mention contract storage size change over these contract versions. 
It makes sense to keep contracts as minimalistic as possible also because of storage fees - 
TON contracts are required to spend some grams from its balances over the time.

While it's not a purpose of this article, I will show how contract's size changed.

<table id="myTable2" class="display">
  <thead>
    <tr>
      <th>Version</th>
      <th>Bits</th>
      <th>Cells</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>V1</td>
      <td>2604</td>
      <td>11</td>
    </tr>
    <tr>
      <td>V2</td>
      <td>2604</td>
      <td>11</td>
    </tr>
    <tr>
      <td>V3</td>
      <td>2580</td>
      <td>12</td>
    </tr>
    <tr>
      <td>V4</td>
      <td>2244</td>
      <td>11</td>
    </tr>
    <tr>
      <td>V5</td>
      <td>1930</td>
      <td>7</td>
    </tr>
    <tr>
      <td>V6</td>
      <td>1962</td>
      <td>7</td>
    </tr>
  </tbody>
</table>

_Note: [version 6](https://github.com/temni/gas-consumptions-showcase/blob/main/contracts/counter_contract_v6.tolk) is a tolk version of the contract.
Appearantly, we have 32 bits more just for using another compiler_

Yeah, we saved **0.0003435 TON** annually. Not much. But if a contract's code is used to deploy new contracts - size's reduction matters.

## Conclusion
Optimizing gas consumption in TON FunC contracts is essential—especially in high‑transaction environments. 
My experiments demonstrate that even seemingly small changes, such as reordering condition checks or 
grouping frequently accessed storage fields, can yield measurable savings. 
Furthermore, using composite counters updated with pure arithmetic and minimizing 
callback message sizes—while marking storage helper functions as inline to avoid extra 
`CALLREF` costs—can result in significant overall savings (nearly 19% reduction compared to a baseline design).

Finally, a comparison with the Tolk conversion tool revealed that the Tolk version 
(with compute-asm-ltr enabled by default) incurs an additional cost of about 31.536 TON/Year. 
You may save a coin even using a right compiler.

This article does not claim to be exhaustive - there are lots of other ways to reduce your gas consumption.
And we are talking about relatively low impact from a single transaction point of view - sometimes it's a matter of nanograms.
And the vast majority of TON smart contracts are designed to have a stable balance - usually the one who pays off is a client.
But in a scale of thousands of transactions we may do a good favor to our clients.

I hope these insights inspire you to explore gas micromanagement in your own FunC contracts on TON.

--- 

## Links

1. [Basic transaction fees information](https://docs.ton.org/v3/documentation/smart-contracts/transaction-fees/fees)
2. [Showcase repository](https://github.com/temni/gas-consumptions-showcase)
3. [Openlib](https://github.com/continuation-team/openlib.func)
4. [Gas prices](https://docs.ton.org/v3/documentation/tvm/instructions#gas-prices)
5. [TVM doc](https://ton.org/tvm.pdf)
6. [Low level fee doc](https://docs.ton.org/v3/documentation/smart-contracts/transaction-fees/fees-low-level)

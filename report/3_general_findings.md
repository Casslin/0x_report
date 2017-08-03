# 3 - General Findings

<!-- This section discusses general issues that apply to the contract (and test) code base. These issues are primarily related to architecture, security assumptions and other high level design decisions, and are essential to a thorough security review. Realizing that these decisions are made as part of a set of trade offs, the 0xProject team may decide not to take action on all of our findings. They should however clarify the rationale for such decisions. -->

## 3.1 Critical

No general issues of critical severity were found.

## 3.2 Major

### Reentrancy risk from malicious tokens

Any call to an external/untrusted contract has risks which can be difficult to quantify given the dynamic and evolving nature of smart contracts. Although we have not found evidence of an exploit, one which we'd like to discuss is the `Proxy` contract calling the `transferFrom()` function on an untrusted contract, intended to be a token. ([/blob/888d5a/contracts/Proxy.sol#L101](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Proxy.sol#L101)).  A malicious token contract could implement its `transferFrom()` to reenter the `Exchange` contract, for example to `fillOrder()` or `cancelOrder()`.

Reference: [Best Practices; Reentrancy](https://github.com/ConsenSys/smart-contract-best-practices#reentrancy)

**Recommendation**

A possible approach would be `require()` that tokens are registered in the `TokenRegistry`.

  In TokenRegistry.sol, provide a lookup by address function (possibly by adding a `mapping(address=>bool)`): the function could be called `exists` and the idea is that `Exchange.sol` would then have `require(registry.exists(makerToken))`. Implement this recommendation in small steps and commits where each commit includes thorough tests. It is likely that each commit will have more test code: for example when adding an `exists` function to the registry, there would probably be tests for: 1) non-existence, 2) existence, 3) non-existence where a prior existing token was removed 4) updating an existing token...

**Resolution**

**JM TODO: I have followed up with 0x about how they would like to address this.**
<br/><br/><br/>

### 3.2.2 Front Running

Miners have ultimate control over transaction ordering and inclusion of transactions on the Ethereum Blockchain. This means that miners are ultimately able to decide which transactions are filled or canceled. Given this inherent power of miners it opens up a possible form of market manipulation: Front running. Front running is the practice of entering into a trade with knowledge of a transaction that will influence the price of the underlying asset or security.

Although 0x uses an off-chain orderbook it is still susceptible to front running as orders are cleared on the blockchain. For example, in Exchange.sol the function fill is susceptible to front running as a miner can always include their transaction first which results in the taker's fill being rejected or only partially filled. Similarly, when a miner sees a maker call cancel they can just fill the order first. Hence, if a maker accidentally places an unfavorable trade they may not be able to cancel it. The functions fillOrdersUpTo and batchFillOrders help mitigate front running by providing backup orders to fill so the taker is not just left with an error log, but nothing prevents a miner from iterating through the orders and filling them all or profiting from slippage.  For instance, a miner might see a taker call fillOrdersUpTo with a large order and then call fill on the lowest priced order and then profit on the additional slippage from the taker's order.

Additionally, given the nature of blockchains, miners are not the only ones able to front run. As an example, a taker could see another taker broadcast a fill order. The malicious taker could instantly broadcast a competing fill order with a higher gas price to increase the probability of their order being filled first.

**Resolution**

Although it is difficult to solve front running problems on a blockchain, 0x has spent a lot of time pondering this problem, and have come up with some impressive solutions to combat front running. Some methods they came up with include a matching model where a relayer only accepts orders that specify the relayer as the taker (this prevents front running as only the relayer can fill the order), a deposit contract model where relayers can create a custom deposit contract that only accepts orders that specify the deposit contract as a taker and then this contract can implement functionality on top of `fillOrder`, and an orderTypes parameter to the message format that will allow for orders with varying functionality that can potentially prevent or disincentivize front running.
<br/><br/><br/>

### 3.2.3 Lack of specifications and documentation

Ref: [Best Practices: specifications and documentation](https://github.com/ConsenSys/smart-contract-best-practices#security-related-documentation-and-procedures)

There is a lack of documentation, with many interactions and components of the system not covered at all in the white paper.  The [critical issue of rounding](../4_specific_findings.md#41-critical) lacks a specification and originally had [no tests](https://github.com/0xProject/contracts/issues/92). Another example is the [Token Distribution contract](#description-token-distribution). Furthermore, it may be preferable for the system to be more codified and deterministic than being dependent on centralized actions such as where the timing is essentially arbitrary.
<br/><br/><br/>

### GL/JC 3.2.4 Rounding of numbers

Given the 0x protocol allows for partial fills, rounding errors are extremely pertinent to the protocol as they can act as a large hidden cost to takers and ultimately result in the loss of tokens. Rounding errors affect partial fills where R = (fillTakerTokenAmount*makerTokenAmount)%takerTokenAmount does not equal 0, and increases linearly as the remainder increases. Furthermore, since the EVM does not support floating point numbers rounding errors need to be approximated. However, the precision of this approximation can be increased by multiplying the remainder (i.e - multiplying the remainder by 10 increases the precision by 1 decimal point). 0x’s original implementation of the `isRoundingError` function: `return (target < 10**3 && mulmod(target, numerator, denominator) != 0);` incorrectly assumed that if the order’s makerTokenAmount is greater than 1000 there will never be a rounding error greater than .1%. An obvious counterexample is a trade where order.makerTokenAmount is 1001, order.takerTokenAmount is 17, and fillTakerTokenAmount is 1. The rounding error here is (58.8823529 - 58)/58 = 1.5%, yet `isRoundingError` returns false.

**Reccomendation**

We recommend re-implementing `isRoundingError`, using the definition of [approximation error](https://en.wikipedia.org/wiki/Approximation_error) and creating thorough unit tests that include boundary conditions such as a .09% error, a .1% error, and a .11% error. For instance, orders where a taker purchases 10000 token A, but only receives 9989 token A, 9990 token A, or 9991 token A. We also believe it would be valuable to garner feedback from the community and future users of the protocol on what an appropriate error threshold is.

**Resolution**

In 0x’s [reimplementation](https://github.com/0xProject/contracts/pull/132/commits/c1c4befeac352eaa144cc9c2185e618bea505c82) of `isRoundingError`, they were able to simplify the rounding error calculation, [using some mathematical magic](https://www.wolframalpha.com/input/?i=((a*b%2Fc)+-+floor(a*b%2Fc))+%2F+(a*b%2Fc)+%3D+((a*b)%25c)%2F(a*b)) this leads to a much more elegant and straightforward implementation, while also reducing gas costs for the takers. Additionally, the 0x team added [unit tests](https://github.com/0xProject/contracts/pull/129/files#diff-e1e51de7476e0cab7ae50ddcbd4c0ff1R105) to the `isRoundingError` function and have included boundary tests.
<br/><br/><br/>



### Unfillable Orders

Another point worth mentioning, due to rounding errors, are unfillable orders. Unfillable orders arise when all potential fills result in too high of a rounding error, so the order is essentially bricked. An example of such an order is outlined below.

Alice creates an order of 1001 token A for 3 token B. Bob then fills this order with fillTakerTokenAmount = 2. This order only has a .05% error, so the order goes through without any problems. However, now if any other taker tries to fill the remaining 1 token B `isRoundingError` will always return true as it has a .19% error. Now, this order is in a perpetual limbo and will waste potential takers' gas until Alice cancels the order. 



## 3.3 Medium

### Makers "griefing attack" on takers

"Griefing" attack of creating many orders is possible, allowing a maker to burn people's gas.  For example: a taker is "griefed" by a maker who's order is no longer valid since the maker tokens no longer exist. This is hard to defend against, as it requires constantly monitoring the maker's token allowance given to the Exchange, (and possibly also balance).

**Recommendation**

Provide tools, scripts, libraries to make it easy for people to identify orders which are no longer valid.
<br/><br/><br/>

### Pragma statements should set a precise version [[issues/102]](https://github.com/0xProject/contracts/issues/102)

Ref: [Best Practices: Locking the pragmas to a specific compiler version](https://github.com/ConsenSys/smart-contract-best-practices#lock-pragmas-to-specific-compiler-version).

Contracts should be deployed using a specific compiler version and flags that they have been tested with. Locking the pragmas to a specific compiler version helps ensure this and prevents contracts from accidentally being deployed with a newer compiler version which may have a higher risk of undiscovered bugs.

**Recommendation**

Lock the pragmas to a specific version in all contracts that will be deployed.

**Resolution** [pull/117](https://github.com/0xProject/contracts/pull/117)

0.4.11 compiler version will be used.
<br/><br/><br/>



### TODO Keep test contracts separate

TODO

**Recommendation**

Move test contracts to its separate directory such as `contracts/test` or `test/contracts`.

**Resolution**

Test contracts were moved to `contracts/test`.
TODO
https://github.com/0xProject/contracts/pull/133/files#diff-879bc9679d7af5c32e07a561199bf7c0


## 3.4 Minor

### Negative maker fees are not possible

The `makerFee` value is a uint, making it impossible to give negative fees, which is sometimes useful for incentivizing liquidity.

**Recommendation**

None. This is a design decision with no impact on security.
<br/><br/><br/>

### Use `enum` for error codes [[issues/105]](https://github.com/0xProject/contracts/issues/105)

On https://github.com/0xProject/contracts/blob/master/contracts/Exchange.sol#L29 and onwards you declare error Codes as 8-bit constants to address those codes by their textual definitions throughout the contract's code, however, it is advisable you do that with an `enum` type as per Solidity specs.

**Recommendation**

Substituting the declaration of error codes as storage variables for an `enum` type.

The purpose of this latter type is exactly to address an enumerable feature through a textual definition throughout the code (with the purpose to not lose readibility) which is exactly the case here.

**Resolution** [[pull/111]](https://github.com/0xProject/contracts/pull/111)

The 8-bit constants were indeed substituted for an `enum` type refering all the textual error definitions, as suggested in the original issue's corpus.
<br/><br/><br/>

### Usage of capitalized names for variables

We applaud the attempts to highlight and distinguish the use of certain state variables such as
https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L35-L36

In our opinion, accessing state variables, especially writing to them, should be "highlighted" in code to prevent confusion against local variables.  We strongly encourage the community to improve clarity around state variables. In this instance, we would just note that the Solidity style guide recommends that capitals be used for constants (akin to other languages).

**Recommendation**

None. At this stage close to release, we do not think the code churn from renaming these variables is worthwhile.
<br/><br/><br/>

### JM Note about ecrecover issue in `solidity <0.4.14`

Truffle is planning to release with support for 0.4.14 on Monday

### Spelling, names, grammar...

Although above we have reported the Major issue of a lack of specifications and documentation, and have reviewed them and code comments, we note that are some spelling mistakes and grammar improvements, and some variable names in code that could be improved, but we do not attempt to record or rectify these.

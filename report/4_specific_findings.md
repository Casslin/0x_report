# 4 - Specific Findings

<!-- This includes most items that can be discussed in one or more specific locations in the code base. When there is a single file or location, it should be called out, when there are multiple similar issues, they can be grouped and listed together. -->

<!-- TODO: move items to proper severity level -->

## 4.1 Critical

###  `Exchange::isRoundingError()` does not return true for errors > 0.1% [[issues/98]](https://github.com/0xProject/contracts/issues/98)

The sole documentation on `isRoundingError()` is a comment in the code stating "[Checks if rounding error > 0.1%](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L468)".


The [implementation](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L473-L478) is broken:

`return (target < 10**3 && mulmod(target, numerator, denominator) != 0);`

A trivial case is `target` values >= 1000 will never indicate a rounding error: the function would always return `false`.

**Recommendation**

1. As there's no other documentation about the system that describes factors such as desired behaviors of the system and the interaction of components and functions like `isRoundingError`, this issue has been classified `Critical` as users of a financial system should have defined and precise behavior. We recommend more documentation on `isRoundingError` including descriptions as they apply to presumably providing users with guarantees and protections, and the limits to those protections and when they will not hold. For example, if certain protections are only provided to users if they trade 1000+ tokens, then those should be clearly documented.

2. We recommend [unit testing `isRoundingError`](https://github.com/0xProject/contracts/issues/92) and reimplementing the function using the standard definition of [approximation error](https://en.wikipedia.org/wiki/Approximation_error) 

**Resolution** 

0x's first reimplementation of isRoundingError is below:
```
function isRoundingError(uint numerator, uint denominator, uint target)
        public
        constant
        returns (bool)
    {
        if (mulmod(target, numerator, denominator) == 0) return false; // No rounding error.
        // (numerator * target / denumerator) * 1000000
        uint partialAmountWithErr = safeMul(getPartialAmount(numerator, denominator, target), 1000000);
        // (numerator * target * 1000000 / denumerator)
        uint partialAmountWithoutErr = safeDiv(
            safeMul(
                safeMul(numerator, target),
                1000000
            ),
            denominator
        );
        uint errPercentageTimes1000 = safeDiv(
            safeSub(partialAmountWithoutErr, partialAmountWithErr), // Absolute rounding error, times 1,000,000
            safeDiv(partialAmountWithoutErr, 1000)                  // Amount being filled, times 1,000
        );
        return errPercentageTimes1000 > 1;
    }: 
```
In 0x’s reimplementation `isRoundingError` first checks if there is a rounding error by using the mulmod opcode. It then calculates the partial amount the taker will receive with the error included (we need this to actually calculate the percent error). This value is then multiplied by 1,000,000 since we’re trying to solve:

  (x - floor(x))/x <= .001 
  
  where x = fillTakerTokenAmount * (makerTokenAmount/takerTokenAmount) = the amount of makerToken the taker will receive. 

In Solidity, we cannot represent decimals so we multiply this entire expression by 1000 which gives us: 

(1000x - 1000floor(x))/x <= 1. 

However, the x in the denominator could still have a rounding error, thus we multiply the numerator and denominator by 1000 again which gives us 

(1,000,000x - 1,000,000floor(x))/1000x <= 1. 

This elucidates why we multiply by 1,000,000. Now, we estimate the amount the taker should receive without any error. We calculate this by performing this calculation: 

filledTakerTokenAmount * makerTokenAmount * 1,000,000/takerTokenAmount. 

(It's worth noting that we multiply by 1,000,000 before dividing to increase the precision of the calculation) 

Now, that we have x and floor(x) the code calculates the error percentage (times 1000) by subtracting the partialAmountWithError from the partialAmountWIthoutError and dividing that by the partialAmountWithoutError*1000. It then checks if this value is > 1 (.1% * 1000). If so, there’s a rounding error and it returns true. 

Shortly after this implementation 0x was able to simplify the logic significantly:
```
function isRoundingError(uint numerator, uint denominator, uint target)
        public
        constant
        returns (bool)
    {
        uint remainder = mulmod(target, numerator, denominator);
        if (remainder == 0) return false; // No rounding error.

        uint errPercentageTimes1000 = safeDiv(
            safeMul(remainder, 1000),
            safeMul(numerator, target)
        );
        return errPercentageTimes1000 > 1;
    }
```
In 0x’s final implementation of `isRoundingError`they were able to simplify the percent error formula to 

R/(fillTakerTokenAmount * makerTokenAmount) <= .001 

where R = (fillTakerTokenAmount * makerTokenAmount)%takerTokenAmount = the remainder of the calculation. 

Multiplying each side by 1000 yields

1000 * R/(fillTakerTokenAmount*makerTokenAmount) <= 1.

And if the calculation is greater than 1 there's a rounding error greater than .1% and the function returns true.

Furthermore, 0x thoroughly [tested](https://github.com/0xProject/contracts/blob/ad79fb669bec3ed431ff4745e978c1535ca82eae/test/ts/exchange/helpers.ts) the `isRoundingError` function and included multiple edge cases.
<br/><br/><br/>

## 4.2 Major

### GS `Exchange::isTransferable()` reentrancy risks [[issues/107]](https://github.com/0xProject/contracts/issues/107)

The `isTransferable()` function, referenced in:
https://github.com/0xProject/contracts/blob/master/contracts/Exchange.sol#L151
is materializes a [dangerous design pattern](https://github.com/ConsenSys/smart-contract-best-practices#reentrancy).

It does not only represent a repetition of checks against a benign (or malicious) ERC20 token and hence wasting gas (*claim 1*) but also an unnecessarily added reentrancy point (*claim 2*).

About *claim 1*:
* The function checks assume the contract's state stays intact throughout multiple calls which, by ERC20 specs, should, but in a malicious token shouldn't. (https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L528)
* Referring back to the point above, if the assumption about the token contract being benign is already made why check the balance again in `isTransferable()` when the same check is already made inside the ERC20 token contract? (gas is already wasted for that!)

About *claim 2*:
I couldn't, to the extent of my analyzing and testing skills, find any reentrancy *bug* but if there's any lesson *TheDao* has taught us is that **[accessing/changing state variables *after* an external call](https://github.com/ConsenSys/smart-contract-best-practices#reentrancy)  in a function is not only dangerous but should be avoided at all costs**.

Such is the case on https://github.com/0xProject/contracts/blob/master/contracts/Exchange.sol#L151 as, following the external calls in `isTransferable()`, `filled[order.orderHash]` is accessed in https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L159.

While it doesn't represent a *bug* in itself it is a poor design choice.

**Recommendation**

After some clarification in the filed Github issue comments the second recommendation still stands: one could have the contract call the `isTransferable()` function with a low-level Solidity `call()` function so as to not have subsequent `fillOrder()` calls interrupting a batched number of orders with bubbling up `throw`s.

This solution would, however, entail the addition of an `address` parameter in the inputs of the function in question.

**Resolution**  [[pull/112]](https://github.com/0xProject/contracts/pull/112)

The 0x team decided not to go with the recommended solution and instead resolved the issue by reducing the gas forwarded in both of the affected functions: `token.balanceOf` and `token.allowance` to a constant value of `4999 gas` in order to prevent state changes when calling malicious tokens (for any state change requires, at least, `5000 gas`).
<br/><br/><br/>

### JC Correctness of `isRoundingError()` and `getPartialAmount()` which have zero tests [[issues/92]](https://github.com/0xProject/contracts/issues/92)


TODO `isRoundingError()` and `getPartialAmount()` have zero tests and no specifications.

**Resolution**

0x implemented tests for both [isRoundingError and getPartialAmount](https://github.com/0xProject/contracts/pull/129/commits/e7c291037d5f1f12be6410dc37c5abbf2de8d9a3)
<br/><br/><br/>

### Redesign of the timelock pattern in custom MultiSig contract [[issues/94]](https://github.com/0xProject/contracts/issues/94)


The
`MultiSigWalletWithTimeLockExceptRemoveAuthorizedAddress` contract has inefficient coding patterns in place.

The one anti-pattern in question here is https://github.com/0xProject/contracts/blob/master/contracts/MultiSigWalletWithTimeLockExceptRemoveAuthorizedAddress.sol#L12 where the contract is `assert`ing a custom function which itself `assert`s a byte-by-byte comparison of a tightly packed `bytes` array vs. a `bytes4` type signature.

**Recommendation** [[pull/96]](https://github.com/0xProject/contracts/pull/96)

A custom assembly block (very simple and straightforward) was built to approximate the functionality of a type cast from a tightly packed `bytes` array to a `bytes4` type (its first 4 bytes, if enough are found).

This is not only much more efficient (saves approximately 2/3 of the original function's gas) but also more secure since the only functionality testing needed is for the 4 bytes cast, which is much more easier to spec out. All other checks are relayed back to Solidity's own constructs.

**Resolution**

* https://github.com/0xProject/contracts/pull/114

Verbatim used by @abandeali1 to justify not implementing the recommendation given:

```
After internal discussions, we have decided to go for code clarity over efficiency in this case. isFunctionRemoveAuthorizedAddress will most likely be only called once in the contract's lifetime, so efficiency isn't particularly important.
```
(v. https://github.com/0xProject/contracts/pull/114#issuecomment-318459655)

We feel the need to note that the wording "(...) clarity over efficiency (...)" is, in our view, debatable and not representative of the whole truth as there's also the security topic touched in the recommendation corpus above but respect the 0x's team final decision as we understand that it is more difficult to trust code not made in-house.
<br/><br/><br/>

### Proper usage of [`require` vs `assert`](https://github.com/ConsenSys/smart-contract-best-practices#use-assert-and-require-properly)

Use of the `assert` function should be limited to checking for states and error conditions which should be unreachable if properly designed. The `require` function should be used to validate  inputs, contract state, or return values from calls to external contracts.

Another important difference is that `require` compiles to the `REVERT` opcode (`0xfd`), which will refund unused gas after the [Metropolis Release](https://github.com/ethereum/EIPs/pull/206).

**Recommendation**

Review the use of `require` and `assert` across all contract files, to ensure that these functions are used in accordance with solidity documentation.

**Resolution**

This was addressed in [pull/117](https://github.com/0xProject/contracts/pull/117/commits/df7fe499c543f5d2c83366e907c3a1cc8c4243a8).
<br/><br/><br/>

## 4.3 Medium

### JC Require callers of Exchange to use token amounts > 0

* https://github.com/0xProject/contracts/issues/101

Callers of functions on `Exchange.sol` can fill and cancel orders for 0 tokens. This would lead to spurious [`ERROR_ORDER_FULLY_FILLED_OR_CANCELLED` logs](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L142).

**Recommendation**

Add `require(fillTakerTokenAmount > 0);` to `fillOrder`
https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L126

Do similar for `cancelOrder` https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L217

**Resolution**

Needs work because these commits are an improvement to a different issue https://github.com/0xProject/contracts/commit/19404fe0eba32d907dee0e69e6f36d0137d6aeb0 and https://github.com/0xProject/contracts/commit/97543b7226fa6bffe2e2bdb931328e4f2397cc6c

Fixed `fillOrder`
https://github.com/0xProject/contracts/commit/97543b7226fa6bffe2e2bdb931328e4f2397cc6c

Still need to fix `cancelOrder`.


<br/><br/><br/>

### Incorrectly named constructor in `ZRXToken` contract

* https://github.com/0xProject/contracts/issues/88

The constructor name was not updated during a previous change when the ZRXToken was renamed.

**Recommendation**

Fix the constructor name to `ZRXToken` and add tests for it.

**Resolution**

Fixed in https://github.com/0xProject/contracts/commit/1a61abe5710fad124b56379d6a8584334afbb244 but no test added.
<br/><br/><br/>



### MultisigWallet implementation

**Recommendation**

TODO This recommendation for using the multisigwallet: go to https://etherscan.io/address/0x851b7f3ab81bd8df354f0d7640efcd7288553419#code
and compile it  with the version listed (v0.4.10+commit.f0d539ae)  with optimizer disabled
<br/><br/><br/>



### EtherToken implementation

**Recommendation**

TODO go to
https://etherscan.io/address/0xD76b5c2A23ef78368d8E34288B5b65D616B746aE#code
and compile it the version listed (v0.4.11+commit.68ef5810)  with optimizer enabled.  It would be safer to avoid the optimizer, but this is what was used for the contract that's currently holding 19,000+ ETH.
<br/><br/><br/>




### Token metadata can be silently overwritten in _TokenRegistry_

* https://github.com/0xProject/contracts/issues/115

In the _TokenRegistry_ contract neither the `addToken()`, `setTokenName()` or the `setTokenSymbol()` check for overwrites in both the `tokenByName` or `tokenBySymbol` mappings.

Although the contract in question is supposed to be controlled only by the DAO/MultiSig, as per the specs, this lack of checks could result in some major confusion in the use of the 0x UI.

Not to mention that assuming a flaw or compromised future DAO it could lead to some severa social engineering attacks.

**Recommendation**

Implement a simple `require()` statement to check if the mapping key being added is already set.

**Resolution**

* https://github.com/0xProject/contracts/commit/be67efaf2bdba62ea8a359228bfe906adc4aef86
* https://github.com/0xProject/contracts/pull/118

Modifiers `nameDoesNotExist` and `symbolDoesNotExist` were added. These implement the `require`s suggested above to check for existing values prior to executing the setter.
<br/><br/><br/>

### `setCapPerAddress()` can be changed by the owner at any time [[issues/84]](https://github.com/0xProject/contracts/issues/84)

As there was no specification at the time of our initial review, it was not clear whether this is intended behavior. In past token sales, typically there are restrictions in code on when terms of the sale can be modified: for example, an obvious one would be to not allow the capPerAddress to be changed by anyone once the sale has started.

**Recommendation**

Clarify the intended design.

**Resolution**

The `setCapPerAddress()` function was replaced with `getEthCapPerAddress()` in [pull/108](https://github.com/0xProject/contracts/pull/108/commits/f56e3fdeaf741beadb3a6f655b49e71ba718e1c3#diff-9f72358cf6a5cd5763f962f132d50481R228). The new function increases the cap each day that passes. We have reviewed the new code and found no safety issues.
<br/><br/><br/>

### Enforce invariants with `assert` instead of "safeMath"

"safeMath" libraries have helped developers avoid issues with overflows and underflows.  However, there are cases when enforcing invariants with `assert` is preferable to excessive use of "safeMath": enforcing invariants often highlights that an overflow or underflow condition is impossible, thus no need for "safeMath" at all.

*Example*

https://github.com/0xProject/contracts/blob/e51d4dcb4c8e0d93815e9d2a5c511d60ce017870/contracts/TokenSale.sol#L169-L171

```
        uint remainingEth = safeSub(order.takerTokenAmount, exchange.getUnavailableTakerTokenAmount(order.orderHash));
        uint ethCapPerAddress = getEthCapPerAddress();
        uint allowedEth = safeSub(ethCapPerAddress, contributed[msg.sender]);
```

Compare the above with the following:

```
        assert(order.takerTokenAmount >= exchange.getUnavailableTakerTokenAmount(order.orderHash));
        uint remainingEth = order.takerTokenAmount - exchange.getUnavailableTakerTokenAmount(order.orderHash);

        uint ethCapPerAddress = getEthCapPerAddress();
        assert(ethCapPerAddress >= contributed[msg.sender]);
        uint allowedEth = ethCapPerAddress, contributed[msg.sender];
```

Note how `assert` has obviated the need for "safeMath".

Recall that `assert` should never be triggered because they are invariants, and a violation of an invariant is clearly a logic error.

If the first `assert` is triggered, that indicates some logic error in the implementation of `exchange.getUnavailableTakerTokenAmount` or `order.takerTokenAmount` or `order.orderHash`.

If the second `assert` is triggered, that indicates some logic error in the implementation of `getEthCapPerAddress()` or how `contributed[msg.sender]` is updated.

Explicit invariants will allow contracts to be formally verified in the future.  Additionally, using `assert` saves gas over a library call, and excessive use of "safeMath" can lead to many false positives for formal verification tools.
<br/><br/><br/>


### Remove explicit dependency of TokenDistribution on the Proxy

* https://github.com/0xProject/contracts/issues/100

The `TokenDistribution.sol` contract has its own state variable for `PROXY_CONTRACT`, but it doesn't need the dependency as it could query the Exchange. Removing it would eliminate the possibility of a mismatch against the exchange's proxy and it would also simplify deployment.

**Recommendation**

Remove the state variable [`PROXY_CONTRACT`](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/TokenDistributionWithRegistry.sol#L30) and replace https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/TokenDistributionWithRegistry.sol#L218 with `exchange.PROXY_CONTRACT()`.

**Resolution**

Fixed in https://github.com/0xProject/contracts/commit/27163a6973758f560984300409599e47c9bcb83b
<br/><br/><br/>

### `TokenDistributionWithRegistry::setTokenAllowance()` is unnecessary

* https://github.com/0xProject/contracts/issues/89

The [`TokenDistributionWithRegistry::setTokenAllowance()`](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/TokenDistributionWithRegistry.sol#L218) function unnecessarily increases the "surface area" of the `TokenDistributionWithRegistry` contract, and can easily be incorporated into `TokenDistributionWithRegistry::init()`.

**Recommendation** Remove `setTokenAllowance()`, and move its operations in `init()`.

**Resolution** This has been fixed: https://github.com/0xProject/contracts/pull/99
https://github.com/0xProject/contracts/commit/efdba3dfdf29585ddeca39a452c16e197f2c7f46#diff-9f72358cf6a5cd5763f962f132d50481
<br/><br/><br/>

### Explicitly mark visibilities

* https://github.com/0xProject/contracts/issues/93

[Explicitly marking visibility](https://github.com/ConsenSys/smart-contract-best-practices#explicitly-mark-visibility-in-functions-and-state-variables) in functions and state variables makes it easier to catch and reason about incorrect assumptions about who can call a function or access a variable. The recent parity multisig exploit highlights the necessity of this.

**Recommendation** Add visibility specifiers for all state variables and functions.

**Resolution** Fixed in https://github.com/0xProject/contracts/commit/ed92920502ca637c7ea6ff071ad85e28ae56aa94
<br/><br/><br/>



### Use the `isConfirmed` modifier instead of `confirmationTimeSet`

In both the `MultiSigWalletWithTimeLock` and `MultiSigWalletWithTimeLockExceptRemoveAuthorizedAddress` contracts, a transaction has a confirmation time if and only if it is confirmed by the required number of owners. The `confirmationTimeNotSet` and `confirmationTimeSet` modifiers are named according to their design, rather than their purpose, which is to guard against calls to `confirmTransaction()`, and revokeConfirmation().

**Recommendation**

The `isConfirmed()` function could be reused to create a pair of modifiers more accurately named `checkConfirmed()` and `checkNotConfirmed()`. These names would be more explicit and thus readable for future reviewers.

**Resolution**

Fixed in [pull/123](https://github.com/0xProject/contracts/pull/123/commits/ae11a52980df283518d5686de67f381538b67323)
<br/><br/><br/>



### Two base token contracts

Two versions of

**Recommendation**

**Resolution**

TODO
<br/><br/><br/>



### Avoid "named returns" (in long functions)

["Named returns"](https://github.com/ethereum/solidity/issues/1401) in Solidity allocate a variable and simply setting the variable to a value will become the return value for the function, unless an explicit return is encountered. These behaviors are uncommon in general programming experience and in our opinion do not bring enough advantages to warrant burdening both old and new Solidity developers with.

**Recommendation**

For example, replacing the named return [`returns(uint filledTakerTokenAmount)`](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L109) with `returns (uint)` and explicitly declaring https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L140
with `uint filledTakerTokenAmount` would make the code more clear, explicit, and less prone to bugs getting introduced (for example, if `return` was used instead of `return 0` in that function).

**Resolution**

Fixed in https://github.com/0xProject/contracts/commit/f47ee8afff485c5ca3cacf242c0fbbf5221f3c54

The fix in [`cancelOrder`](https://github.com/0xProject/contracts/commit/f47ee8afff485c5ca3cacf242c0fbbf5221f3c54#diff-a5e6c43f03b96911facdf9d4e9b82b9cL224) illustrates the recommendation's benefits.

It turned out that the actual example in the recommendation could not be performed due to EVM stack limits and it was essential to [add the named return back](https://github.com/0xProject/contracts/commit/5972561e916d9110f2de6d4b7af3ed27f773ab34).
<br/><br/><br/>

### Zero tests for `Exchange::batchFillOrKillOrders()`

* https://github.com/0xProject/contracts/issues/116

https://github.com/0xProject/contracts/blob/frozen/test/ts/exchange/wrapper.ts
has no test/s for `batchFillOrKillOrders()`
https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Exchange.sol#L337

**Recommendation**

Add one or more tests with usage and values similar to how `batchFillOrKillOrders()` would actually be called in practice.

**Resolution**

A test was added https://github.com/0xProject/contracts/pull/133/commits/a0b60198e29b62b2b242e4e744c446239bb8e8d5#diff-9be78ea0202647bb62f62111374c3ecaR190  The correctness and rigor was not evaluated due to time constraints.

<br/><br/><br/>



### Use `keccak256` instead of `sha3`

https://github.com/0xProject/contracts/issues/77

In Solidity, there's no difference between the two, except that `keccak256` is accurate and will not mislead new or casual observers about which hash algorithm is being executed (since the standardized SHA-3 is slightly different from Keccak-256).

**Recommendation**

Use `keccak256` instead of `sha3`.

**Resolution**

Partially fixed in: https://github.com/0xProject/contracts/commit/ed92920502ca637c7ea6ff071ad85e28ae56aa94

We thought one case was missed so we fixed: https://github.com/0xProject/contracts/pull/133/commits/eca7163e59d0de911cee15a9621b52f0efe62248

Unfortunately, there is still an occurrence: https://github.com/0xProject/contracts/blob/e51d4dcb4c8e0d93815e9d2a5c511d60ce017870/contracts/MultiSigWalletWithTimeLockExceptRemoveAuthorizedAddress.sol#L58

<br/><br/><br/>



### Token address is not indexed in TokenRegistry events

https://github.com/0xProject/contracts/issues/135

**Recommendation**

Use `address indexed token` in the TokenRegistry events such as:
https://github.com/0xProject/contracts/blob/71281859f3466b51dcc2c09e740c106bff192564/contracts/TokenRegistry.sol#L27

**Resolution**

None (in fairness to the 0x team, this was reported after they provided a final commit hash).

<br/><br/><br/>


## 4.4 Minor

### IPFS hashes more than 32 bytes are unsupported

* https://github.com/0xProject/contracts/issues/75

The `TokenRegistry.sol` contract uses a `bytes32` value to store IPFS hashes.
https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/TokenRegistry.sol#L94

IPFS uses a [multihash](https://github.com/multiformats/multihash) and the TokenRegistry might also be assuming a convention that the first 2 bytes of the IPFS hash, which are not stored on-chain are 0x1220.

**Recommendations**: Assumptions should be documented. Consider future-proofing by using the `bytes` type rather than `bytes32`.

**Resolution** Fixed by storing IPFS hashes stored as `bytes` here: https://github.com/0xProject/contracts/pull/86
<br/><br/><br/>

### TokenDistributionWithRegistry::init does not ensure zero fees [[issues/144]](https://github.com/0xProject/contracts/issues/144)

`TokenDistributionWithRegistry::init()` has no `require` or `assert` statement to ensure that there are no fees, and no `feeRecipient` specified on the order. This can only be verified by viewing the contract's storage after the order has been created. This is particularly relevant given that the `TokenDistributionWithRegistry` contract uses the Exchange mechanism, but inserts itself as the taker, and then forwards the proceeds to the caller of `fillOrderWithEth()`.

**Recommendation**

Add a `require` statement ensuring no extra fees will be paid on order fills.

**Resolution**

[Pull/108](https://github.com/0xProject/contracts/pull/108#discussion-diff-129530972R145), added

```
require(order.feeRecipient == address(0));
```

Per the logic of the `Exchange` contract, this will also ensure that no fees are paid.
<br/><br/><br/>

### No versioning support in Exchange contract

* https://github.com/0xProject/contracts/issues/113

It's not clear how an upgraded exchange contract will differentiate from this contract.

**Recommendation**

Exchange contract versions could be clarified with a `string public version` value in `Exchange.sol`, or else adding `mapping (address => string) public version` to `Proxy.sol`.

**Resolution**

Fixed in [pull/117](https://github.com/0xProject/contracts/pull/117/commits/edd4a09f6fa55d045cc3d20d8687ec688263fc00).
<br/><br/><br/>

### GL/JM TokenDistributionWithRegistry Order Struct
<!-- George -->
https://github.com/0xProject/contracts/issues/91
<!-- Not sure this one should be included? [TODO cf TokenDistributionWithRegistry::init does not ensure zero fees] -->

### JM/JC Duplication of contract address storage in `TokenDistributionWithRegistry.sol`

TODO https://github.com/0xProject/contracts/issues/138 this could change to
instead of another variable (PROTOCOL_TOKEN_CONTRACT), they should just be able to do `address(protocolToken)`  https://github.com/0xProject/contracts/blob/e51d4dcb4c8e0d93815e9d2a5c511d60ce017870/contracts/TokenSale.sol#L141

* We note that the contract is using [3 storage slots](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/TokenDistributionWithRegistry.sol#L35-L37) which could be avoided.

**Recommendation**

None. This is effectively a one-time cost instead of a recurring cost, so there are no dangers related to increasing costs.
<br/><br/><br/>

### TokenDistributionWithRegistry.sol does not reference Registry.sol

* https://github.com/0xProject/contracts/issues/106

Despite the name, [TokenDistributionWithRegistry](https://github.com/0xProject/contracts/blob/frozen/contracts/TokenDistributionWithRegistry.sol) contract does not import, or make any reference to the Registry contract at all. This is confusing.

**Recommendation**

Use a clearer name, or implement the intended `Registry.sol` functionality.

**Resolution**

0x clarified that the "Registry" being referred to here is the mapping holding a list of `registered` contributors. The contract was renamed to `TokenSale.sol`: https://github.com/0xProject/contracts/pull/120.
<br/><br/><br/>


### `Proxy.sol` contract is not generic as its name implies

* https://github.com/0xProject/contracts/issues/109

The concept of a Proxy contracts is often associated with a general purpose identity contract, as  in [Uport](https://github.com/uport-project/uport-identity/blob/develop/contracts/Proxy.sol) or as proposed in [ERC 121](https://github.com/ethereum/EIPs/issues/121). `TokenProxy.sol` (or something else) may be a better name because the [proxy function here isn't generic but for token transferFrom](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Proxy.sol#L101)

**Recommendations**

Rename [`Proxy.sol`](https://github.com/0xProject/contracts/blob/888d5a02573572240f4c55e03238be603c13c469/contracts/Proxy.sol
) to `TokenProxy.sol`, or another more precise name.

**Resolution**

Fixed: Proxy was renamed to TokenProxy and then TokenTransferProxy in https://github.com/0xProject/contracts/commit/fe28d31e988fe347f290a805db98c8e48773d8d5
<br/><br/><br/>

### Non-meaningful boolean returns in multiple functions

* https://github.com/0xProject/contracts/issues/141

There are several functions that explicitly `return true;` when this information could totally be inferred.

_Exchange.sol_:
https://github.com/0xProject/contracts/blob/frozenUpdated/contracts/Exchange.sol#L302
https://github.com/0xProject/contracts/blob/frozenUpdated/contracts/Exchange.sol#L336
https://github.com/0xProject/contracts/blob/frozenUpdated/contracts/Exchange.sol#L367
https://github.com/0xProject/contracts/blob/frozenUpdated/contracts/Exchange.sol#L426

_TokenTransferProxy.sol_:
https://github.com/0xProject/contracts/blob/frozenUpdated/contracts/TokenTransferProxy.sol#L65
https://github.com/0xProject/contracts/blob/frozenUpdated/contracts/TokenTransferProxy.sol#L86

_ZRXToken.sol_:
https://github.com/0xProject/contracts/blob/frozenUpdated/contracts/tokens/ZRXToken.sol#L26

These return booleans are used in functions made for heavy and repeated use which means they could account for some gas savings for the end-user.

Since these return values cannot be used by web3.js and calling it from other contracts will result in the same outcome *with or without* that statement (since they would `throw` all the same even if not returning anything) our recommendation would be to strip these out of the final codebase.

**Recommendations**

Totally remove the `return` statements and `returns ()` modifiers from the functions

<br/><br/><br/>

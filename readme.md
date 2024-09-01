# CHIP-2024-07-BigInt: High-Precision Arithmetic for Bitcoin Cash

        Title: High-Precision Arithmetic for Bitcoin Cash
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Draft
        Initial Publication Date: 2024-07-24
        Latest Revision Date: 2024-08-28
        Version: 1.1.0

## Summary

This proposal removes the limit on the maximum length of numbers in the Bitcoin Cash Virtual Machine. The number length limit is superseded by the [Arithmetic Operation Cost Limit](https://github.com/bitjson/bch-vm-limits#arithmetic-operation-cost).

## Deployment

Deployment of this specification is proposed for the May 2025 upgrade.

- Activation is proposed for `1731672000` MTP, (`2024-11-15T12:00:00.000Z`) on `chipnet`.
- Activation is proposed for `1747310400` MTP, (`2025-05-15T12:00:00.000Z`) on the BCH network (`mainnet`), `testnet3`, `testnet4`, and `scalenet`.

This proposal requires [`CHIP: Targeted Virtual Machine Limits`](https://github.com/bitjson/bch-vm-limits).

## Motivation

Many financial and cryptographic applications require higher-precision arithmetic than is currently available to Bitcoin Cash contracts.

While Bitcoin Cash's 2022 upgrade [increased the maximum number length from 4 bytes to 8 bytes](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Bigger-Script-Integers.md), the design of the Bitcoin Cash Virtual Machine's anti-Denial-Of-Service limits prevented further increase.

Following [`CHIP: Targeted Virtual Machine Limits`](https://github.com/bitjson/bch-vm-limits), the Virtual Machine (VM) can correctly account for arithmetic operation costs, so higher-precision arithmetic operations can be enabled without increasing the processing or memory requirements of the VM.

## Benefits

By allowing contracts to efficiently operate on larger numbers, this proposal improves contract security and reduces transaction sizes.

### Safer Contracts

This proposal obviates the need for higher-precision math emulation. As a result, existing applications can be simplified, making them easier to develop and review. Additionally, by making arithmetic overflows less common, contracts are less likely to include vulnerabilities or faults which can expose users to attacks and losses.

### Reduced Transaction Sizes

Because this proposal allows existing contracts to remove higher-precision math emulation, transactions employing these contracts are reduced in size. This reduces transaction fees for contract users, and it reduces storage and bandwidth costs for validators.

### Practical Applications

Currently the Bitcoin Cash network has a few decentralized applications that would benefit from higher precision arithmetic limits:

- [AnyHedge](https://gitlab.com/GeneralProtocols/anyhedge), a decentralized hedge solution against arbitrary assets on Bitcoin Cash, currently with [approx. 25,000 BCH in total value locked (TVL)](https://defillama.com/protocol/anyhedge?denomination=BCH).
- [Cauldron](https://gitlab.com/riftenlabs/cauldron-whitepaper), an efficient constant product market maker contract on Bitcoin Cash through micro-pools, currently with [approx. 200 BCH in TVL](https://defillama.com/protocol/cauldron?denomination=BCH).
- [BCH Guru](https://bch.guru/), a unique, on-chain, peer-to-peer crypto price prediction game and NFT collection, built on Bitcoin Cash mainchain, currently with [approx. 0.4 BCH and 326k FURU in TVL](https://app.bch.guru/#/live).
- [Fex Cash](https://github.com/fex-cash/fex/blob/main/whitepaper/fex_whitepaper.md), an advanced automatic market maker (AMM) contract system, currently with [approx. 15 BCH in TVL](https://fex.cash/ranking).

All these applications would benefit from increased precision because they could calculate payouts with full precision while avoiding workarounds.

Future applications that could be created with greater ease:

- High-precision state accumulators for decentralized finance (DeFi) operations, e.g. adding / removing small shares to / from a liquidity pool with accurate calculation of payout / deposit amounts, and splitting and merging such liquidity pools without leakeages due to rounding errors.
These operations are critical building blocks in more advanced decentralized exchange (DEX) systems, over-collateralized stablecoin systems, prediction market (PM) systems, decentralized autonomous organization (DAO) systems, etc.
- Contracts processing block headers in order to verify simple payment verification (SPV) proofs, or in order to feed mining information into contracts in a trustless way.
These applications require performing 256-bit unsigned arithmetics in order to verify accumulated chainwork, so would need at least 513-bit signed integer support.
Applications include hashrate prediction markets, gambling / lottery applications (public entropy accumulation), verifying transactions or state of other blockchains (potential use in cross-chain bridging systems).
- Contracts wanting to implement custom operations on elliptic curve(s), e.g. [key tweaking](https://bitcoin.stackexchange.com/questions/110402/how-to-tweak-a-public-key-for-taproot), amount blinding schemes ([confidential transactions](https://elementsproject.org/features/confidential-transactions)), or signature verification using other ellpitic curves than Bitcoin's secp256k1.
These operations would similarly require 513-bit signed integer support (for 256-bit keys).
- Contracts wanting to verify Rivest-Shamir-Adleman (RSA) signatures, these operations would require 8193-bit signed integer support (for 4096-bit keys).
- Contracts wanting to implement more advanced cryptography, such as implementing zero-knowledge scalable transparent argument of knowledge (ZK-STARK) proof verification.

## Technical Specification

The limit on the maximum length of Bitcoin Cash VM numbers (A.K.A. `nMaxNumSize`) is removed.  
The numbers will still be limited, by the maximum stack item size (A.K.A. `MAX_SCRIPT_ELEMENT_SIZE`).

For reference, here we will specify the whole set of affected operations.

### Script Number Encoding

The numbers are encoded as variable-length byte arrays in little endian byte order (least significant byte first).

Sign is encoded with a sign bit, and sign bit is defined as the highest order bit, e.g. `ff7f` encodes 32767, `ffff` encodes -32767.
This means that some positive values must have `00` appended so to move the sign bit to the next byte, and allow encoding positive values which would need to use the sign bit (`0x80`), e.g. `ff8000` encodes 33023 and `ff8080` encodes -33023.

Encoding of value 0 is special, it is encoded as an empty stack item of 0 length and NOT with 0-byte (`00`) or "negative" 0-byte (`80`).

Numbers must be minimally encoded, e.g. `0200` could be decoded to number 2 but it is not valid and encoding it as `02` is mandatory.

This upgrade affects two non-arithmetic opcodes that are used to convert an arbitrary byte array to a minimally encoded script number and back, we will specify them below as well.

#### OP_NUM2BIN (0x80)

Pop two items from stack.  
The top-most value is read as a desired length of the output's stack item, and the other one as binary value to be converted.  The length must be a minimally encoded script number, otherwise the script fails immediately. The binary value may be any length and does not need to start out as a minimally encoded script number, however.
If the requested length is larger than `MAX_SCRIPT_ELEMENT_SIZE`, fail immediately.  
The value is then transformed to a minimally-encoded script number (meaning trailing zeroes may be popped off). At this point the value's size may shrink.
Then, if the new (possibly reduced) length of the value is larger than the requested length, fail immediately.  
Otherwise, pad the value with 0-bytes until the desired length is reached and then push the result to stack.

Executing the operation on value 0 and length 0 is valid and will return 0 as an empty stack item.  
When positive values are padded the operation will simply add 0-bytes on the higher end, e.g. executing it on `7b` (123) and `05` (5) will return `7b00000000`.  
When negative values are padded the operation will similarly add 0-bytes on the higher end but it must also move the sign bit to highest byte, e.g. executing it on `fb` (-123) and `05` (5) will return `7b00000080`.

#### OP_BIN2NUM (0x81)

Pop one item from stack.
Decode the stack item to a numerical value using script number encoding scheme.
Push the value on stack as a minimally-encoded script number.

For example, byte sequence `0080` is not a valid script number encoding, but the opcode would convert it to value 0 and return an empty stack item which is the only valid encoding for value 0.
Similarly, byte sequence `ff0080` is not a valid script number encoding, but the opcode would convert it to value -255 and return `ff80` which is the only valid encoding for value -255.

This operation may reduce the size of the stack item but it will never increase it.

Before this upgrade, the operation would fail if it would have to return a value outside the int64 range.
After this upgrade, the operation will never fail because it can not increase the size of the stack item so any byte sequence on stack will be decodeable.

### Arithmetic Operations

**General requirements**

If any of the input stack items is not a minimally-encoded script number then the operation must fail, e.g. trying to add `0100` and `01` must fail rather than return `02`.

The operation must fail if any resulting stack item would exceed `MAX_SCRIPT_ELEMENT_SIZE`.

Any result must be returned as a minimally encoded script number, e.g. number 1 is to be returned as `01` rather than `0100`, number -1 is to be returned as `81` rather than `0180`, and number 0 is to be returned as empty stack item rather than `00`.

#### Unary Operations

Any unary operation must fail if executed on empty stack (stack depth of 0).

##### OP_1ADD (0x8b)

Pop one item from stack, add 1 to the value, push the result on stack.
This may increase the size of the stack item, e.g. if executed on `7f` (127) the result must be `ff00` (128).
This may reduce the size of the stack item, e.g. if executed on `8080` (-128) the result must be `ff` (-127).

##### OP_1SUB (0x8c)

Pop one item from stack, subtract 1 from the value, push the result on stack.
This may increase the size of the stack item, e.g. if executed on `ff` (-127) the result must be `8080` (-128).
This may reduce the size of the stack item, e.g. if executed on `ff00` (128) the result must be `7f` (127).

##### OP_NEGATE (0x8f)

Pop one item from stack, if it is nonzero, flip the sign, if it is zero, leave the item unchanged. In either case, push the result on stack.
This will always result in the input operand and the result having the same byte size.

##### OP_ABS (0x90)

Pop one item from stack, if value is negative flip the sign else do nothing, push the result on stack.
This will result in same size of the result.

##### OP_NOT (0x91)

Pop one item from stack. If value is 0 change it to 1, else for all other values,  change it to 0. Push the result on the stack.
This may reduce the size of the stack item to 0, e.g. if executed on `aabbdd` the result will be an empty stack item (0).
Or, it may increase the size of the stack item from 0 to 1, e.g. if executed on  an empty stack item (0), it will yield `01`.

##### OP_0NOTEQUAL (0x92)

Pop one item from stack. If value is not 0 change it to 1. If it is 0, leave it unchanged. Push the result on stack.
This may reduce the size of the stack item to 0, e.g. if executed on `aabbdd` the result will be an empty stack item (0). If executed for an empty stack item (0), it will leave the top stack item unchanged.

#### Binary Operations

Any binary operation must fail if executed on stack depth of 1 or less.

##### OP_ADD (0x93)

Pop two items from stack, add the values together, push the result on stack.

##### OP_SUB (0x94)

Pop two items from stack, subtract the top-most value from the other one, push the result on stack.

##### OP_MUL (0x95)

Pop two items from stack, multiply the values together, push the result on stack.

##### OP_DIV (0x96)

Pop two items from stack, divide the 2nd-to-top value with the top-most value, push the **quotient** on stack, e.g. 3/2 will return 1, -3/2 will return -1, 3/-2 will return -1, and -3/-2 will return 1.

##### OP_MOD (0x97)

Pop two items from stack, divide the 2nd-to-top value with the top-most value, push the **remainder** on stack.
The sign of the result will match the sign of the dividend, e.g. 7/3 will return 1, -7/3 will return -1, 7/-3 will return 1, and -7/3 will return -1.

##### OP_BOOLAND (0x9a)

Pop two items from stack.
If both numbers are non-zero then return 1, else return 0.

##### OP_BOOLOR (0x9b)

Pop two items from stack.
If at least one of the numbers is non-zero then return 1, else return 0.

##### OP_NUMEQUAL (0x9c)

Pop two items from stack.
If the two numbers are equal then return 1, else return 0.

##### OP_NUMEQUALVERIFY (0x9d)

Pop two items from stack.
If the two numbers are equal then continue evaluation with nothing returned to stack, else fail.

##### OP_NUMNOTEQUAL (0x9e)

Pop two items from stack.
If the two numbers are not equal then return 1, else return 0.

##### OP_LESSTHAN (0x9f)

Pop two items from stack.
If the 2nd-to-top value is less than the top-most value then return 1, else return 0.

##### OP_GREATERTHAN (0xa0)

Pop two items from stack.
If the 2nd-to-top value is greater than the top-most value then return 1, else return 0.

##### OP_LESSTHANOREQUAL (0xa1)

Pop two items from stack.
If 2nd-to-top value is less than or equal to the top-most value then return 1, else return 0.

##### OP_GREATERTHANOREQUAL (0xa2)

Pop two items from stack.
If 2nd-to-top value is greater than or equal to the top-most value then return 1, else return 0.

##### OP_MIN (0xa3)

Pop two items from stack.
Return the lesser of the 2 values (if they're equal just return any one value).

##### OP_MAX (0xa4)

Pop two items from stack.
Return the greater of the 2 values (if they're equal just return any one value).

### Ternary Operations

Any ternary operation must fail if executed on stack depth of 2 or less.

##### OP_WITHIN (0xa5)

Pop three items from stack.  
The top-most item defines the "right" and non-inclusive boundary of a range.  
The 2nd-to-top item defines the "left" and includive boundary of a range.  
The 3rd-to-top item is the value to be evaluated.  
If value being evaluated is in the range (greater or equal to the "left" value, and less than the "right" value) then return 1, else return 0.

Examples: evaluating `1 5 8` will return 0, evaluating `5 5 8` will return 1, evaluating `7 5 8` will return 1, and evaluating `8 5 8` will return 0.

Uses where the "left" value is greater or equal to the "right" value are valid and will always return 0, e.g. `X 8 5` will return 0 for any value of X.

## Rationale

### Removal of Number Length Limit

With the activation of the [Arithmetic Operation Cost Limit](https://github.com/bitjson/bch-vm-limits#arithmetic-operation-cost), the additional limitation on VM number length has no impact on worst-case transaction validation performance (see [Tests & Benchmarks](#tests--benchmarks)). This proposal removes the limit, allowing valid VM numbers to extend to the stack element size limit (A.K.A. `MAX_SCRIPT_ELEMENT_SIZE`) of 10,000 bytes. See [`CHIP: Targeted Virtual Machine Limits`](https://github.com/bitjson/bch-vm-limits).

Alternatively, this proposal could raise the limit to a higher constant value like `258`, the constant selected by Satoshi Nakamoto in [`reverted makefile.unix wx-config -- version 0.3.6`](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/commit/757f0769d8360ea043f469f3a35f6ec204740446) (July 29, 2010). However, because the limit is no longer necessary for mitigating worst-case transaction validation cost, selection of any particular constant would be arbitrary.

By fully-removing the limit, overall protocol complexity is reduced, simplifying both future VM implementations and contract development. For VM implementations, eliminating out-of-range cases significantly reduces the combinatorial set of possible inputs and outputs for the numeric operations: `OP_1ADD` (`0x8b`), `OP_1SUB` (`0x8c`), `OP_NEGATE` (`0x8f`), `OP_ABS` (`0x90`), `OP_NOT` (`0x91`), `OP_0NOTEQUAL` (`0x92`), `OP_ADD` (`0x93`), `OP_SUB` (`0x94`), `OP_MUL` (`0x95`), `OP_DIV` (`0x96`), `OP_MOD` (`0x97`), `OP_BOOLAND` (`0x9a`), `OP_BOOLOR` (`0x9b`), `OP_NUMEQUAL` (`0x9c`), `OP_NUMEQUALVERIFY` (`0x9d`), `OP_NUMNOTEQUAL` (`0x9e`), `OP_LESSTHAN` (`0x9f`), `OP_GREATERTHAN` (`0xa0`), `OP_LESSTHANOREQUAL` (`0xa1`), `OP_GREATERTHANOREQUAL` (`0xa2`), `OP_MIN` (`0xa3`), `OP_MAX` (`0xa4`), and `OP_WITHIN` (`0xa5`). For contract authors, eliminating the possibility of out-of-range errors prevents a class of potential vulnerabilities arising from a contract system's failure to validate that intermediate arithmetic results never exceed an (uncommonly-encountered) maximum number length limit.

### Non-Inclusion of Implementation-Specific Technical Details

This proposal specifies only the necessary change to the consensus protocol: removal of the number length limit.

The software changes required to support this consensus change differ significantly from implementation to implementation. VM implementations which already internally use arbitrary-precision arithmetic for VM number manipulation may only need to disable code enforcing the previous limit, while other implementations may be required to integrate an arbitrary-precision arithmetic library or language primitive. In all cases, implementations should verify their functional behavior and performance characteristics against the [Tests & Benchmarks](#tests--benchmarks).

### Non-Inclusion of VM Number Format or Operation Descriptions

Beyond dropping the unnecessary limit on VM number length, this proposal does not modify any other properties of the VM. Notably, the precise format and behavior of VM numbers across all VM operations – especially `OP_1ADD` (`0x8b`), `OP_1SUB` (`0x8c`), `OP_NEGATE` (`0x8f`), `OP_ABS` (`0x90`), `OP_NOT` (`0x91`), `OP_0NOTEQUAL` (`0x92`), `OP_ADD` (`0x93`), `OP_SUB` (`0x94`), `OP_MUL` (`0x95`), `OP_DIV` (`0x96`), `OP_MOD` (`0x97`), `OP_BOOLAND` (`0x9a`), `OP_BOOLOR` (`0x9b`), `OP_NUMEQUAL` (`0x9c`), `OP_NUMEQUALVERIFY` (`0x9d`), `OP_NUMNOTEQUAL` (`0x9e`), `OP_LESSTHAN` (`0x9f`), `OP_GREATERTHAN` (`0xa0`), `OP_LESSTHANOREQUAL` (`0xa1`), `OP_GREATERTHANOREQUAL` (`0xa2`), `OP_MIN` (`0xa3`), `OP_MAX` (`0xa4`), and `OP_WITHIN` (`0xa5`) – are part of network consensus and do not otherwise change as a result of this proposal. For the avoidance of doubt, see the [Tests & Benchmarks](#tests--benchmarks).

## Stakeholder Responses & Statements

[Stakeholder Responses & Statements &rarr;](stakeholders.md)

## Tests & Benchmarks

[`CHIP: Targeted Virtual Machine Limits`](https://github.com/bitjson/bch-vm-limits) includes a suite of functional tests and benchmarks to verify the behavior and performance of all operations within virtual machine implementations, including high-precision arithmetic operations. See [CHIP Limits: Tests & Benchmarks](https://github.com/bitjson/bch-vm-limits/blob/master/tests-and-benchmarks.md) for details.

## Implementations

Please see the following reference implementations for additional examples and test vectors:

### Node Implementations

- C++:
  - [Bitcoin Cash Node (BCHN)](https://bitcoincashnode.org/) – A professional, miner-friendly node that solves practical problems for Bitcoin Cash. [Merge Request !1876](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/merge_requests/1876).

### Other Implementations

- JavaScript/TypeScript
  - [Libauth](https://github.com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash. [Pull Request #139](https://github.com/bitauth/libauth/pull/139).
  - [Bitauth IDE](https://github.com/bitauth/bitauth-ide) – An online IDE for bitcoin (cash) contracts. [Pull Request #101](https://github.com/bitauth/bitauth-ide/pull/101).

## Activation Costs

We define activation cost as any cost that would arise ex-ante, in anticipation of the activation by affected stakeholders, to ensure that their services will continue to function without interruption once the activation block has been mined.
In case of this proposal, activation cost is contained to nodes and libraries that implement the Script VM, and will amount to:

- Some fixed amount of node developer man-hours, in order to release updated node software that will be able to correctly calculate the algorithmic limit.
- Some fixed amount of work by stakeholders running node software, in order to update the software in anticipation of algorithm activation.
- Some fixed amount of effort by others involved in reviewing and testing the new node software version.
- Some fixed amount of effort by library maintainers in order to update their VM implementations to correctly execute arithmetic operations that would be failing pre-upgrade.

This is the same kind of cost that any upgrade to the Script VM system has had or will have.
Nothing is required of stakeholders who are neither running nodes nor involved in node development or smart contract development.
Smart contract developers are the direct beneficiaries of this upgrade, so we expect them to be happy to bear the activation costs.

## Ongoing Costs

We define ongoing cost as any cost that would arise post upgrade.
Thanks to the opcode cost budgeting sytem introduced in [CHIP: Targeted Virtual Machine Limits](https://github.com/bitjson/bch-vm-limits), valiated by benchmarks done, we can be assured that costs of validating transactions WILL NOT be negatively impacted.

## Risk Assessment

If we define risk as [probability times severity](https://en.wikipedia.org/wiki/Risk#Expected_values) then we could make an estimate using the [risk matrix](https://en.wikipedia.org/wiki/Risk_matrix) framework, considering both severity and likelihood in our consideration of risk.

Generic consensus upgrade risks apply.
Probability of bugs and network splits is entirely controlled by node developers and quality of their work.
Generic risks are mitigated by node software development quality control, review, and testing processes, leaving the probability of a critical bug very low, and with that the generic risk is kept low.

Below we will examine some risks specific to this upgrade.

### Big Integer Implementation Risks

As we [mentioned above](#rationale), Satoshi Nakamoto himself removed the BigInt support from Bitcoin's codebase, because of instability in the OpenSSL's big number library, and the risk of different nodes coming to different conclusions about result of some Script operation and it causing a "hard" network split.

Satoshi Nakamoto was a solo developer trying to bootstart a never before seen project, so we can argue that it was a logical choice to prioritize other things, and *temporarily* remove the risk by simply removing big integer support entirely.
We are now in a much better position, where we can give the problem the attention it deserves, and remove such risks while accessing the benefits of having big integers.

It was not the only risky dependency - signature cryptography (with secp256k1 elliptic curve) itself also required an external dependency, and later, when more developers got involved, they implemented [secp256k1](https://github.com/bitcoin/bitcoin/pull/4312) by themselves.

Thankfully, basic arithmetic operations on big integer numbers are a much simpler problem, and a problem where many industries needed a solution, so reliable big number libraries now exist for most programming languages.
Initial implementation for BCHN intends to use [The GNU Multiple Precision Arithmetic Library](https://gmplib.org/) (AKA `gmp`), which is a time-tested and performant library, of which we only need a subset of operations for BCH Script arithmetic opcodes.
The same library was used by libsecp256k1, until 2015 when it was [fully removed](https://github.com/bitcoin/bitcoin/commit/75a880390191aeaf7d2fa326f194349a891db022#diff-242b05bb9c57125ed0fbecbda608e3fa6fd3d7cae41e1cba0dcfd706252a7fc0R413) as a dependency.

There exist big integer libraries for languages other than C++ and used in Bitcoin Cash node implementations:

- Golang's [`math/big`](https://pkg.go.dev/math/big) package, expected to be used with [BCHD](https://github.com/OPReturnCode/bchd), considering it is already internally used for [cryptography operations](https://github.com/OPReturnCode/bchd/blob/67d41ee8838026571863491e88288d6d2d5cb717/bchec/privkey.go#L11).
- Java's [`java.math,BigInteger`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/math/BigInteger.html) library, expected to be used with [Bitcoin Verde](https://github.com/SoftwareVerde/bitcoin-verde), considering it is already internally used for [uint256 operations](https://github.com/SoftwareVerde/bitcoin-verde/blob/e9d140cac8b93a9db572bda906db9decfac1d7ae/src/main/java/com/softwareverde/bitcoin/block/header/difficulty/work/ChainWork.java#L8). It doesn't need to use it for cryptography operatins, because native secp256k1 implementation exists for Java and [is used by Bitcoin Verde](https://github.com/SoftwareVerde/bitcoin-verde/blob/e9d140cac8b93a9db572bda906db9decfac1d7ae/src/main/java/org/bitcoin/NativeSecp256k1.java#L5).

Similarly, there exist big integer libraries for languages in which Bitcoin Cash smart contract libraries are written:

- JavaScript's [`BigInt`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt) library, expected to be used with Bitcoin Cash JavaScript libraries such as [libauth](https://github.com/bitauth/libauth) and [CashScript](https://github.com/CashScript/cashscript).
- Python [natively supports unlimited integer precision](https://docs.python.org/3/library/stdtypes.html#numeric-types-int-float-complex), expected to be used with [python-bitcoincash](https://github.com/dagurval/python-bitcoincash).

All these implementations must produce exact same results for exact same inputs, else node implementations would risk a "hard" network split and smart contract libraries would risk faulty transaction validation breaking user-facing applications.

This risk is mitigated by having a [comprehensive set of test vectors](#test-vectors), which cover all the arithmetic opcodes and around all the edges, and that ensures low implemnatiton risk as any implementation errors would be caught by the test suite, and before deployment.

### Risks to Existing Smart Contracts

This upgrade would make some previously invalid contract bytecode become valid.
If arithmetic opcode's failure mode was intentionally used by some contract, then such contract may start behaving in unintended ways should the opcodes cease to be failing.

This risk existed when Bitcoin Cash upgraded arithmetic opcodes from int32 to int64, and was the reason for [this notice](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Bigger-Script-Integers.md):

> As always, the security of a contract is the responsibility of the entity locking funds in that contract; funds can always be locked in insecure contracts (e.g. `OP_DROP OP_1`). This notice is provided to warn contract authors and explicitly codify a network policy: the possible existence of poorly-designed contracts will not preclude future upgrades from further expanding the range of Script Numbers.
>
> To ensure a contract will always fail when arithmetic results overflow or underflow 8-byte Script Numbers (in the rare case that such a behavior is desirable), that behavior must be either 1) explicitly validated or 2) introduced to the contract prior to the activation of any future upgrade which expands the range of Script Numbers.

What is the likelihood of a contract writer not being aware of the above notice, and then devising such a contract where it would actually result in unintended results rather than not affecting active instances or just expanding the working-as-intended scope of the contract?
The Bitcoin Cash smart contract development ecosystem is still small enough, so most builders know each other, and smart contract applications with any number of users quickly become common knowledge.
As part of the CHIP process they will all be contacted for [final confirmation](#stakeholder-responses--statements).

Even before that, we can already examine how this upgrade would impact some of those:

- If we examine the [Cauldron contract](https://docs.riftenlabs.com/cauldron/whitepaper/#appendix-cauldron-contract) we can see that it would continue to function as intended even with increased precison, rather than fail and require the liquidity pools to be split across smaller ones.
Because of how it was written it automatically benefits from this upgrade.
- If we examine the Fex Cash contract, we can see a different approach: they worked around the precision issue by [explicitly limiting](https://github.com/fex-cash/fex/blob/main/covenants/amm/burn_lp_token.cash#L7) size of input integers.
In this case, the contract would work the same regardless of whether we upgraded integers or not, but would not automatically benefit from increased precision.
- If we examine the AnyHedge contract we can see it does not even use multiplication, and [`nominalUnitsXSatsPerBch`](https://gitlab.com/GeneralProtocols/anyhedge/contracts/-/blob/development/contracts/v0.12/contract.cash#L27) is set at creation of a contract instance and anything higher than current limits would make it unredeemable, while post-upgrade it would continue to work as intended, and it would work with a wider range of contract parameters.
- If we examine the [BCH Guru contract](https://ide.bitauth.com/import-template/eJztWFtv28YS_isGcR7dZO-XvMmyGguRJdeiclwkgbGX2VqtLelYVNAg8H8_Q0qiREm-FWhRFPWDSQ5nZr-Z-WZ2qe_Zf-bhBu5c9i67KYrZ_N3bt-MIb_y4cIvi5k2Y3r0tb2BSjIMrxtPJDwXczW5dAT98JW-Wtm9-nU8n2XEWYR7ux7NSC92hYOLuAO9O2mdH7xf3i6P86khzQSyRSRgBiSVpnfRWeSWASkjAHBGEMhRoaiRN1ARiqPY2AIAQJKLXEksxhnn27vvDcTYPMHH342n5mM2hKG7hDjWui99LwWOQlor4XNy7ydyFpcL3bDyZLQp09el7Nl0Us-l4UnQnEdAXOa4l-cbmzM1v0J0TiWkiePRAaAjJKmnAxxhT8l6qpCAkI7S0VifjSXIQozdJaxJCgOgRyBz-t4BJgP7izsN9td5icjsNv40nv5x8KyBMIwL_lM1vp0X25eF4DyD9WwDEGlQJxzUTlvx6leiHhy_H2Ve4n1d5ZsdZaViMy1qsErtO-1d3u4ChK6bzmzFKKCFkqd1YJtPKWSqUSdES4blhNHoajY6JGSqY9EY5mrykyB_DjXGhLPb0N6jKjFSGX6b339BRtM4zAZFKp41QIfBogRDhnAzeamUUSZRHFyi4SCTqIBGTUIw5zCI4dOvuposJRswQKwZaknK6uA8weHVY6_IehqokdcANF8YpB0FhQZWWPEYgnDJOLTc0mYi9pblFjJpEIoKIEdEGTVNJ_1Qsnc6cH9-Oi9LtZDopOwFb_W5clL2DMoxfIUSyTDP4JBIkTrXlVhOJWdBJSYvd6pRPyVLHIUio0vzwUJLzZfHusCV7-AtrVNHznzklttLqodj04N91aNRT7d8h8bqmweo-1jP_vGHx9HD8E0pUnS7KTFdni61OwqeZm8_LI8innTPHl3p4bOkfbyo2uLi-GA3PTn7OO8NrJY7I78566hKyLzjjUqCGesYDjZZhGpm11AtwwflomARGJMXEGm-4Z1R56rG6kitKwSulhQDpMUAniApJCBM8UBoT9VIrIiwF8NIpZS1ojycvooF-njQwUYWYaEgOO1y5yC3xhFi1ulojy0rv2FRxSM8c80JRFUlSzHEuU9DB4wBBoQaOp7oUpIQQlCQpgAWpELbxghJpPBVeWaCAnEERYzbopPHsB1IQjjPGOBdY8AJZpplVlksqKFYPqQfSqHAoDmlWccAKv15edXg8DpYIM5wqQ2JUkavkTJluPLeKAAGbyAdtOEnCIQLKXUgScRgcfz4pmxK2XMQqci2jMdhXwKQh2lK0ooIrjqNUI0uVjSZiXqOi1hlKrbQEB5XarYcpIRlXoiUMi-wcWaog-Gw9TeebWbB96mrwdL3fbRi6rbpF0YbX9bZcekd3q6bMv81KBzM2v2Ek26b8o1xnEgN5_UDeyQZn6MUyiAQ7A_cipWPUSDRm0FI54QVxnHhrQzlxQgoei6IowTmnQpQ2xBDNrk-OPgmHVC4aIxUyKI_0wmkotFGMeG2w2ZA9RnmIgLTETxf8JMEvGC0U4FQlOvKDXlkkNnqCjU1TEmCE5JbaEIwqwYJgOmqfgJrEApFIdSECtSqFpFLEnOxiLQnKxZLUnyfVS3lUanTbH5asKJ_6o_POT6NWr5J0f_w8OcI_vNUb1Uclovx3Oej1akn7rNP-cNrKW8Pu-4-dy-6PPy9f1QqXg7y8qC1Br1deh2ctJjfiCtNBD_lVt38xytuDUT-vhay-W8dz0Lay7PZPO1e1iLzMssrVYJRfDLr9PL86aw3P6nf0iXePe2xYNTHRl2Eye-k3eyXie5JnS6T23KrDbl9XebnnpEI7vOh1K0r0uxc1p2rh6eWgkp50-wxzUZueji6apXl_2Wnlncv8rNWvX_z5MOhwdPKyWsm9hD25wmGkz5oM_9uqRGzPtNcZDp9PzlprcLmaCU8p73OAskOp3E1aVa9RfjXIBx86_fbg_Lybn3e2mnkT5h9N3z79K-3RSXlpnQy3JtFBi3oo1RbNtq3Rt86bY2h7ucYDLtEf5M2srqft7ogZ5Xu-VyoDLERDcIhzmyJtp6pGtedjr3UOwDvkCcvb0O_0T7dNHout3Jzag9PO04EdjGpV6M1w7Q07fySFdXVOux9fn87nYtnp26cCamjQvwI8fQa8eBn4ZrlLROsObN5sb2mNgh3safrsGG2ebggJwSdqqx49PW3uRb1B-0PePe8s_TS5sz-LPrZ6o86BZcozJBjCV9Pg5UNsb1fZP0qM8uaq-1O7WefnkvMoMQeHNvUXnEx2Aa5G5utQHWBcvfzGw3Kr2FyWdCkptv5Qef6TotJqflMcPLYNHjmy7SZlX3N1TKv5vA48q77_F7PZ9L6AWH5BnbTPrhlh_JpIhFQ-DS867Wz7Ryvy8H_DLmXP), we can see it'd be impacted in similar way as AnyHedge.

These all have active contracts that handle non-trivial amounts of money, and their developers are well aware of the notice, and their contracts wouldn't be broken by this upgrade anyway.
We can expect similar findings with any others, because it would be hard to actually implement a contract that would be broken by this upgrade.

With this in mind we can estimate this risk to be extremely low, because both likelihood and severity of some exotic contract breaking are low.

## Feedback & Reviews

- [BigInt CHIP Issues](https://github.com/bitjson/bch-bigint/issues)
- [`CHIP 2024-07 BigInt` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2024-07-bigint-high-precision-arithmetic-for-bitcoin-cash/1356)

## Changelog

This section summarizes the evolution of this document.

- **v1.1.0 – 2024-08-28**
  - Remove the VM number length limit ([#1](https://github.com/bitjson/bch-bigint/issues/1))
- **v1.0.0 – 2024-07-24** ([`b114c957`](https://github.com/bitjson/bch-bigint/commit/b114c95729e670f4b0780d4fd14590c35d281d77))
  - Initial publication

## Copyright

This document is placed in the public domain.

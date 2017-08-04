# 1. Summary of the Review process and methods

**TODO** title of this section and content below should match up more

## 1.1 Scope

### Source code 

The code base reviewed was the *frozen* branch in the  [0xProject/contracts](https://github.com/0xProject/contracts/tree/888d5a02573572240f4c55e03238be603c13c469) repository.

The state of the source code at the time of the review can be found under the commit hash [`888d5a02573572240f4c55e03238be603c13c469`](https://github.com/0xProject/contracts/tree/888d5a02573572240f4c55e03238be603c13c469).

Revisions made after the review can be found in the *frozenUpdated* branch under **SPECIFIC COMMIT HASH**.

### Documentation

0x provided ConsenSys Diligence with the following documentation:

* The [0x Project Whitepaper](https://0xproject.com/pdfs/0x_white_paper.pdf)
* The [README](https://github.com/0xPro ject/contracts/blob/master/README.md) file for the [0xProject/contracts](https://github.com/0xProject/contracts/tree/frozen) repository.

### Dynamic tests

The pre-existing tests for [0xProject/contracts](https://github.com/0xProject/contracts/tree/frozen) repository were executed using the truffle framework, run against contracts deployed on a local instance of testrpc.

In order for the tests to succeed, the module `ethereumjs-testrpc` had to be uninstalled and reinstalled specifying version `3.0.2`.

```
$ npm uninstall -g ethereumjs-testrpc
$ npm install -g ethereumjs-testrpc@3.0.2
```

The revised code base now includes the script `npm run testrpc`,

### Review Goals

0x contracted ConsenSys Diligence to review the codebase for the following properties:

**Security**: identification of security related issues within each
contract and within the system of contracts.

**Sound Architecture**: evaluation of the architecture of this system through the lens of established smart contract best practices and general software best practices.

**Code Correctness and Quality**:
A full review of the contract source code.  The primary areas of focus include:

* Correctness (does it do what it is supposed to do)
* Readability (how easily it can be read and understood)
* Sections of code with high complexity
* Improving scalability
* Quantity and quality of test coverage

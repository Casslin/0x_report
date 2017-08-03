

## 2.2 Materials Included in Audit
<!-- ^ Needs a better section name -->
<!-- Can we use the word 'audit?' Need to ask Matt Corva. -->

### Source code 

The code audited was from the [0xProject/contracts](https://github.com/0xProject/contracts/tree/888d5a02573572240f4c55e03238be603c13c469) repository (*frozen* branch).

The state of the source code at the time of the audit can be found under the commit hash [`888d5a02573572240f4c55e03238be603c13c469`](https://github.com/0xProject/contracts/tree/888d5a02573572240f4c55e03238be603c13c469).

### Documentation

The following documentation was available to the review team:

* The [0x Project Whitepaper](https://0xproject.com/pdfs/0x_white_paper.pdf)
* The [README](https://github.com/0xPro ject/contracts/blob/master/README.md) file for the [0xProject/contracts](https://github.com/0xProject/contracts/tree/frozen) repository.

### Dynamic tests

The pre-existing tests for [0xProject/contracts](https://github.com/0xProject/contracts/tree/frozen) repository were executed using the truffle framework, run against contracts deployed on a local instance of testrpc.

In order for the tests to succeed, the module `ethereumjs-testrpc` had to be uninstalled and reinstalled specifying version `3.0.2`. [TODO cf the minor item about tests]

```
$ npm uninstall -g ethereumjs-testrpc
$ npm install -g ethereumjs-testrpc@3.0.2
```

The revised code base now includes the script `npm run testrpc`,
<br/><br/><br/>


## 2.3 Audit Goals

The focus of our audit was to ensure the following properties:

**Security**:
identifying security related issues within each
contract and within the system of contracts.

**Sound Architecture**:
evaluation of the architecture of this system through the lens of established smart contract best practices and general software best practices.

**Code Correctness and Quality**:
a full review of the contract source code.  The primary areas of focus include:

* Correctness (does it do was it is supposed to do)
* Readability (How easily it can be read and understood)
* Sections of code with high complexity
* Improving scalability
* Quantity and quality of test coverage

![](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/images/1.jpg)
<div align="right">
  <i>Generated with midjourney 6</i>
</div>

## Table of Contents

1. [**Introduction**](#introduction)
2. [**Configuration**](#configuration)
   - [Harness](#harness)
   - [Specs](#specs)
   - [Confs](#confs)
   - [Mutations](#mutations)
3. [**Execution**](#execution)
4. [**Preparation**](#preparation)
   - [Separating Functionality](#separating-functionality)
   - [Methods Declaration](#methods-declaration)
   - [Unresolved Calls](#unresolved-calls)
   - [Shadowing Storage](#shadowing-storage)
5. [**Thinking about Properties**](#thinking-about-properties)
6. [**Developing Properties**](#developing-properties)
7. [**Quality Assurance**](#quality-assurance)
   - [Manual Mutations](#manual-mutations)
   - [Coverage Information](#coverage-information)
   - [Identifying Problems](#identifying-problems)
8. [**Automation**](#automation)
9. [**Conclusion**](#conclusion)

<a name="introduction"></a>
## Introduction

In 2023, I got involved in formal verification by joining five Certora [community contests](https://www.certora.com/contests), moving myself to a top position on the [leaderboard](https://www.certora.com/leaderboard). I learned a lot and developed a unique workflow and mindset, which I want to share in this article. I'll use the [Badger eBTC Competition](https://code4rena.com/audits/2023-10-badger-ebtc-audit-certora-formal-verification-competition) as an example in this article. This competition was an exciting challenge where participants were motivated to develop and validate comprehensive properties of 4 smart contracts: [EBTCToken](https://github.com/alexzoid-eth/2023-10-badger/blob/main/packages/contracts/contracts/EBTCToken.sol), [ActivePool](https://github.com/alexzoid-eth/2023-10-badger/blob/main/packages/contracts/contracts/ActivePool.sol), [CollSurplusPool](https://github.com/alexzoid-eth/2023-10-badger/blob/main/packages/contracts/contracts/CollSurplusPool.sol), and [SortedCdps](https://github.com/alexzoid-eth/2023-10-badger/blob/main/packages/contracts/contracts/SortedCdps.sol). 

In this article, rather than reiterating technical details about formal verification, predicate logic, and the Certora Prover – all of which are thoroughly explained in the latest [Certora tutorials](https://docs.certora.com/projects/tutorials/en/latest/lesson1_prerequisites/index.html) – I will focus on sharing a practical workflow and my personal insights. Additionally, I'll include a collection of helpful resources and links to guide you through the process. 

> "Formal verification might sound hard, but you don't need to be a math expert to use it." [(c)](https://twitter.com/CertoraInc/status/1731719978771227071)

Simply put, the formal verification process involves crafting properties (akin to writing tests) and submitting them alongside compiled Solidity smart contracts to a remote prover. This prover essentially transforms the contract bytecode and your rules into a mathematical model, determining the validity of your rules.  

For those new to Certora Prover, it's crucial to understand that:
1. The prover operates at the bytecode level. It even [works](https://www.certora.com/blog/vyper-announcement) with Vyper language as well.
2. Unlike fuzz testing, where functions are repeatedly executed with varying parameters, the prover efficiently translates the contract's bytecode and rules into a mathematical model that proves every possible code execution .
4. Variables, including the contract state, blockchain environment, and return values of unresolved external calls, are assigned a range of all possible values. It's your responsibility to define these bounds.

<a name="configuration"></a>
## Configuration

To begin, it's essential to prepare all configuration files. The recommended file structure for configs, historically used in projects like [this one](https://github.com/alexzoid-eth/2023-10-badger-fv/tree/main/certora), is organized as follows:
```
certora\
    harness\
    specs\
    confs\
    mutations\
```

Each folder has a specific role, which I will discuss in detail in the following sections.

<a name="harness"></a>
### Harness

> Prover interacts with `external` (`public`) functions and variables. Access to `internal` can be provided via harness contracts.

For each contract you intend to test, create a corresponding harness contract. Additionally, if necessary, add mock contracts to simulate external calls. These harness contracts are the ones the Prover will interact with. Essentially, a harness is a wrapper inherited from the contract under test. Its primary function is to provide useful features, like access to `internal` variables and functions, and to overcome certain limitations of CVL language.

Here is an example of the file structure for harness contracts and mocks:
```
certora\harness\
├── ActivePoolHarness.sol         // Harness for `ActivePool.sol`
├── CollateralTokenTester.sol     // Mock for `CollateralToken`
├── CollSurplusPoolHarness.sol    // Harness for `CollSurplusPool.sol`
├── DummyERC20A.sol               // Mock for an `ERC20` token
├── DummyERC20B.sol               // Mock for another `ERC20` token
├── DummyERC20Impl.sol            // Basic `ERC20` implementation
├── EBTCTokenHarness.sol          // Harness for `EBTCToken.sol`
└── SortedCdpsHarness.sol         // Harness for `SortedCdps.sol`
```

A simple harness file is just a derivative of the tested contract. It doesn't add new functionality at the moment but inherits the original contract's features. An example can be found [here](https://github.com/alexzoid-eth/2023-10-badger/blob/main/certora/harness/ActivePoolHarness.sol#L1-L18), showing the `ActivePoolHarness.sol`:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.17;

import "../../packages/contracts/contracts/ActivePool.sol";

contract ActivePoolHarness is ActivePool { 
    
    constructor(
        address _borrowerOperationsAddress,
        address _cdpManagerAddress,
        address _collTokenAddress,
        address _collSurplusAddress,
        address _feeRecipientAddress
    ) ActivePool(
        _borrowerOperationsAddress, _cdpManagerAddress, _collTokenAddress, _collSurplusAddress, _feeRecipientAddress
    ) { }
}
```

<a name="specs"></a>
### Specs

Create a dedicated specification file for each contract you're testing. These files should be placed in the `spec` directory as follows:

```
certora\specs\
├── ActivePool.spec
├── CollSurplusPool.spec
├── EBTCToken.spec
└── SortedCdps.spec
```

I recommend structuring each specification file into distinct sections, separated by comment lines. Each section serves a specific purpose:

```spec
/////////////////// METHODS ///////////////////////

methods {
}

///////////////// DEFINITIONS /////////////////////

////////////////// FUNCTIONS //////////////////////

///////////////// GHOSTS & HOOKS //////////////////

///////////////// PROPERTIES //////////////////////
```

- **METHODS** ([link](https://docs.certora.com/en/latest/docs/cvl/methods.html)): This section includes additional information about contract methods. While not always necessary, it's good practice to declare all external methods of the tested contract and any linked contracts here.

- **DEFINITIONS** ([link](https://docs.certora.com/en/latest/docs/confluence/anatomy/definitions.html)): Similar to macros in other languages, you can define constants or simple expressions in this section. 

- **FUNCTIONS** ([link](https://docs.certora.com/en/latest/docs/confluence/anatomy/functions.html)): In CVL, a function can either be invoked within a rule or serve as a stub for a contract's external function. 

- **GHOSTS & HOOKS** ([Ghosts](https://docs.certora.com/en/latest/docs/cvl/ghosts.html) & [Hooks](https://docs.certora.com/en/latest/docs/cvl/hooks.html)): Mostly contains shadow copies of storage variables. Its importance lies in providing access to `private` variables and facilitating the implementation of effective invariants.

- **PROPERTIES**: This section is where the [rules](https://docs.certora.com/en/latest/docs/cvl/rules.html) and [invariants](https://docs.certora.com/en/latest/docs/cvl/invariants.html) are defined. It's the core of your specification.

I will delve into each of these blocks in more detail further in the article. For now, just incorporate this structured comment block into each specification file.

<a name="confs"></a>
### Confs

> Remember, the prover verifies specification against one main contract, with others being linked to it.

Initially, it's necessary to create separate configuration files for each contract:

```
certora\confs\
├── ActivePool_verified.conf
├── CollSurplusPool_verified.conf
├── EBTCToken_verified.conf
└── SortedCdps_verified.conf
```

The suffix `_verified` in each file name is a convention from Certora community contests. It signifies that the configuration is intended to prove the declared functionality. Another common suffix is `_violated`, indicating that the configuration is designed to demonstrate an actual bug (where a rule is violated, revealing a flaw).

Let's examine `ActivePool_verified.conf` for a clearer understanding (see the [example](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/confs/ActivePool_verified.conf)):

```json
{
    "files": [
        "certora/harness/ActivePoolHarness.sol",
        "certora/harness/CollateralTokenTester.sol",
        "certora/harness/CollSurplusPoolHarness.sol",
        "certora/harness/DummyERC20A.sol",
        "certora/harness/DummyERC20B.sol",
    ],
    "link": [

        "ActivePoolHarness:collateral=CollateralTokenTester",
        "ActivePoolHarness:collSurplusPoolAddress=CollSurplusPoolHarness",

        "CollSurplusPoolHarness:activePoolAddress=ActivePoolHarness",
        "CollSurplusPoolHarness:collateral=CollateralTokenTester",
    ],
    "verify": "ActivePoolHarness:certora/specs/ActivePool.spec",
    "loop_iter": "3",
    "optimistic_loop": true,
    "rule_sanity": "basic",
    "msg": "ActivePoolHarness",
    "parametric_contracts": [ "ActivePoolHarness" ]
}
```

The configuration contains several key blocks:

1. **`files` Block**: Lists all contracts to be compiled, all of which can be utilized in our specifications.

2. **`link` Block**: Defines static links between contracts listed in `files`. This is crucial because the constructor logic is not 'executed' before proving rules. Therefore, static links in configuration files globally set immutable variables and dependencies.

3. **`verify`**: Specifies the main contract to be verified against the specification.

4. **Additional Options**: For a complete list of flags and options, refer to the official [documentation](https://docs.certora.com/en/latest/docs/prover/cli/options.html).

An important consideration in configuration is the linking of variables. While it's not mandatory, I strongly recommend linking `immutable` variables, which are set in the `constructor`. This is crucial because the prover does not account for `constructor` logic.  For external calls to dependent contracts and mocks, linking is not mandatory, but I advise doing so to avoid unexpected results.

If you're dealing with variables that can change (non-immutable variables), it's better to link them dynamically in your rules. This lets you test how they change more effectively.

While it's not mandatory to create a configuration file (since all options can be passed via command line parameters to `certoruRun`), employing a separate [configuration file](https://docs.certora.com/en/latest/docs/prover/cli/conf-file-api.html) is considered best practice.

<a name="mutations"></a>
### Mutations

The final directory in our setup is for mutations, which are altered versions of the contract files, designed to test specific rules. There are two main types of mutations:

1. **Manual Mutations**: These are typically modified contract files adapted to prove a specific rule. My advice is to create and test these mutations immediately after implementing each rule. This proactive approach helps ensure that each rule is not only implemented correctly but is also effectively doing its job.

2. **Gambit Mutations**: To gather coverage information, a special engine called `Gambit` is used (see [documentation](https://docs.certora.com/en/latest/docs/gambit/index.html)). It generates numerous mutated files, each of which is then tested against your specifications. The idea here is to see if rules are violated in these mutated scenarios.

Here’s how your mutations directory structure might initially look:

```
certora\mutations\
├── ActivePool
├── CollSurplusPool
├── EBTCToken
└── SortedCdps
```

<a name="execution"></a>
## Execution

For newcomers:
- install prover with `pip3 install certora-cli`, also I recommend check updates from time to time with `pip3 install certora-cli --upgrade`
- request a Certora licence key via [website](https://www.certora.com/signup?plan=prover) or [discord](https://discord.com/channels/795999272293236746/1080511450075893800) and set it as `CERTORAKEY` environment variable

Setup appropriate solidity compiler version, for `Badger eBTC` it is `0.8.17`, with command `solc-select install 0.8.17 && solc-select use 0.8.17`

Let's add the first rule and check that everything is done as it should. Add a rule `sanity` to the `certora\specs\ActivePool.spec`. 

```spec
///////////////// PROPERTIES //////////////////////

rule sanity(method f, env e, calldataarg args) {
    f(e, args);
    satisfy(true);
}
```

This basic rule ensures that all external functions can execute without causing a revert. Typically, the prover overlooks any execution s leading to a revert. However, using the `satisfy` system function, we can confirm if a specific condition holds `true` in at least one scenario. In other words, it checks that each `external` function can successfully execute without reverting in at least one possible case.

Now we can execute a prover with `certoraRun certora/confs/ActivePool_verified.conf` from the root directory. 

Output will be something like this:
```bash
// ... Compilation messages ...

Connecting to server...

Job submitted to server

Follow your job at https://prover.certora.com
Once the job is completed, the results will be available at https://prover.certora.com/output/52567/3828709713ec4504b61f3e8a2f824703?anonymousKey=1a9b6143a36bed6d9ab0c1afd325747842949db9
```

You got a shareable link to result of your rule execution. It's comfortable to use, but keep in mind that all associated files like your specification, configs and contracts are available as well. If you want to securely share your result with Certora command in debug purpose, you can simply remove `anonymousKey` from the url. 

![](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/images/2.png)

<a name="preparation"></a>
## Preparation

Before diving into property formulation, it's beneficial to make some preparations.

<a name="separating-functionality"></a>
### Separating Functionality

I recommend viewing your contracts from two perspectives: firstly as the main contract under test, and secondly as a potential externally linked contract. This approach leads to a logical division of functionality. In the externally linked part, include all contract setup operations, and in the tested contract, import these operations.

For instance, in our project scope, we have four contracts: `EBTCToken`, `ActivePool`, `CollSurplusPool`, and `SortedCdps`. Each of these can be both the focus of testing and a contract linked externally. Let's illustrate this separation:

```
certora\specs\
├── base
│   ├── activePool.spec
│   ├── collSurplusPool.spec
│   ├── eBTCToken.spec
│   └── sortedCdps.spec
├── ActivePool.spec
├── CollSurplusPool.spec
├── EBTCToken.spec
└── SortedCdps.spec
```

In this structure, we use the root `spec` directory for testing the contracts, and the `base` directory for specifications meant for importing. Begin by adding the `base` directory and the relevant files. For example, insert `import "./base/activePool.spec";` at the beginning of `ActivePool.spec` and do the same for the other files.

<a name="methods-declaration"></a>
### Methods Declaration

The next step involves declaring all external methods in the `METHODS` block. This is not just a best practice but also a way to achieve specific behaviors in testing. For instance, an external method not declared in this block is assumed to interact with the current blockchain environment (`block`, `msg`). By declaring a method as `envfree` in the methods block, you explicitly indicate that it does not rely on the blockchain environment, leading to clearer specifications.

The methods block is a complex topic and goes beyond the scope of this summary, so for a more in-depth understanding, refer to the [Methods Block](https://docs.certora.com/en/latest/docs/cvl/methods.html) and [Working with Multiple Contracts](https://docs.certora.com/en/latest/docs/user-guide/multicontract/index.html) documentation.

<a name="unresolved-calls"></a>
### Unresolved Calls

When using the Prover, it's crucial to understand how it deals with calls to unresolved functions. By default, the Prover adopts a strategy known as "havocing," where it assumes that these unresolved calls could result in almost any state change. This broad assumption can introduce unexpected behavior into your specification. Therefore, I strongly advise against leaving any calls unresolved.

To identify these unresolved calls, refer to the `Contracts Call Resolutions` [section](https://prover.certora.com/output/52567/3828709713ec4504b61f3e8a2f824703?anonymousKey=1a9b6143a36bed6d9ab0c1afd325747842949db9) of the Prover's output.

![](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/images/3.png)

To effectively handle these unresolved calls, follow the guidance provided in the [Handling Unresolved Method Calls](https://docs.certora.com/en/latest/docs/user-guide/multicontract/index.html#handling-unresolved-method-calls) section of the Certora documentation. This resource provides detailed steps and best practices for managing such scenarios to ensure your specification behaves as intended.

<a name="shadowing-storage"></a>
### Shadowing Storage

Creating a shadow copy of all storage variables (in `GHOSTS & HOOKS` block) is a strategic step before formulating properties. This approach streamlines the process, allowing you to focus on developing invariants. 

There are two main reasons to adopt this strategy:
1. **Access to Private Variables**: Without altering the original contract code, creating a shadow copy is the only way to access `private` variables.
2. **Quantifiers Limitations**: By default (without `--allow_solidity_calls_in_quantifiers` flag), direct calls to contract functions are not supported in quantifiers.

A typical shadow copy involves three components:
1. **The Ghost Variable**: Represents the shadow copy of the actual storage variable.
2. **Read Access Hook**: Utilizes the `Sload` hook for read operations.
3. **Write Access Hook**: Uses the `Sstore` hook for write operations.

As an example, consider the shadow copy for `address public feeRecipientAddress;` from [ActivePool.sol](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/packages/contracts/contracts/ActivePool.sol#L29). It includes two ghost variables (`ghostFeeRecipientAddress` and `ghostFeeRecipientAddressPrev`) along with `Sload` and `Sstore` hooks. Keeping track of the variable's previous value is useful for analyzing state transitions.

Here is how it looks in [practice](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/specs/base/activePool.spec#L45-L64):

```spec
//
// Ghost copy of `feeRecipientAddress`
//

ghost address ghostFeeRecipientAddress {
    init_state axiom ghostFeeRecipientAddress == 0;
}

ghost address ghostFeeRecipientAddressPrev {
    init_state axiom ghostFeeRecipientAddressPrev == 0;
}

hook Sload address val _ActivePool.feeRecipientAddress STORAGE {
    require(ghostFeeRecipientAddress == val);
}

hook Sstore _ActivePool.feeRecipientAddress address val STORAGE {
    ghostFeeRecipientAddressPrev = ghostFeeRecipientAddress;
    ghostFeeRecipientAddress = val;
}
```

For detailed information, please refer to the [Ghosts](https://docs.certora.com/en/latest/docs/cvl/ghosts.html) and [Load and Store Hooks](https://docs.certora.com/en/latest/docs/cvl/hooks.html#load-and-store-hooks) sections in the documentation. 

Though setting up hooks can be labor-intensive, this crucial preparation step significantly enhances the efficiency of your verification process in the long run.

<a name="thinking-about-properties"></a>
## Thinking about Properties

As we reach this phase, it's time to start formulating our properties. However, before diving into code, it's crucial to take a moment to systematically conceptualize your properties.

Begin by framing your properties in simple English ([PROPERTIES.md](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/packages/contracts/specs/PROPERTIES.md)). This approach helps in clearly defining what you aim to achieve before any coding begins. For guidance on this process, consider these insightful posts: [Post #1](https://twitter.com/agfviggiano/status/1687854392202997760) and [Post #2](https://twitter.com/agfviggiano/status/1735235127171551320), which offer detailed explanations.

Certora's team has identified five primary [categories of properties](https://github.com/Certora/Tutorials/blob/master/06.Lesson_ThinkingProperties/Categorizing_Properties.pdf).
![](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/images/5.png)

I suggest starting with the `Valid States` category. These properties are crucial when linking your specification to another contract. They encompass initial setups like constructor configurations, initial storage variable values, correctness of linked lists, and summaries of user balances relative to the total balance. From an external contract's viewpoint, these invariants are essential for proper setup and utilization.

Place these properties in the `base` directory. For instance, many [properties](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/specs/base/sortedCdps.spec#L248-L417) in `SortedCdps` are designed to ensure a correctly structured list, fitting into the `Valid States` category. Beginning with this category is strategic, as many other property types often rely on having a valid state as a foundation.

After that, progress from `High-level` properties to `Unit Tests`, transitioning from those with broader project impact to more specific ones.

<a name="developing-properties"></a>
## Developing Properties

When constructing properties in formal verification, we mainly deal with two types: [Invariants](https://docs.certora.com/en/latest/docs/cvl/invariants.html) and [Rules](https://docs.certora.com/en/latest/docs/cvl/rules.html). 

In simple terms, an `invariant` functions as follows: it establishes an initial condition for the contract's environment, then an external function of the contract is executed. After execution, the prover checks whether the contract state still meets the invariant's criteria. 

Consider the basic invariant [sortedCdpsMaxSizeGtZero](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/specs/base/sortedCdps.spec#L260-L261):

```spec
invariant sortedCdpsMaxSizeGtZero() _SortedCdps.maxSize() != 0
    filtered { f -> !HARNESS_REPLACED_OR_VIEW_FUNCTIONS(f) }
```

Here, the `filtered` block narrows down the set of external functions to be tested. The prover assumes `_SortedCdps.maxSize() != 0` before and checks this condition after the function execution.

On the other hand, a `rule` is structured differently and comprises three segments: setting up the environment, executing the function, and then verifying the post-execution environment with `assert` or `satisfy`. The rule is meaningful only when it includes an `assert` or `satisfy` statement.

For instance, look at the simple rule [reInsertNotAffectSize](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/specs/SortedCdps.spec#L126-L136):

```spec
// reInsert() should not change list size
rule reInsertNotAffectSize(env e, bytes32 _id, uint256 _newNICR, bytes32 _prevId, bytes32 _nextId) {

    mathint sizeBefore = ghostSize;

    reInsert(e, _id, _newNICR, _prevId, _nextId);

    mathint sizeAfter = ghostSize;

    assert(sizeBefore == sizeAfter);
}
```

It begins by saving the initial state of `ghostSize`, executes the `reInsert` function, and concludes with an `assert` to confirm that the list size remains unchanged.

When writing your properties, adopt a methodical approach by breaking down complex rules into simpler ones. This strategy of decomposition, coupled with constraining the range of possible values, can significantly speed up the rule execution process.

To test a specific property, use the `--rule` flag. This tests only the selected rule, saving time: `certoraRun certora/confs/ActivePool_verified.conf --rule sanity`.

For additional information on CVL (Certora Verification Language), I recommend paying attention to official [Documentation](https://docs.certora.com/en/latest/docs/cvl/index.html), video of [Practical Introduction](https://www.youtube.com/watch?v=DtVj788m3Qo), and exploring a range of [examples](https://github.com/Certora/Examples) and [real projects](https://github.com/Kirkeelee/Certora-examples).

If you encounter any questions or need further clarification, the [Certora Discord help-desk](https://discord.com/channels/795999272293236746/1104825071450718338) is an excellent resource for information and support.

<a name="quality-assurance"></a>
## Quality Assurance

Ensuring the quality of your properties is as crucial as testing in traditional programming. To gain a deeper understanding of this process, I recommend two insightful videos: [Checking Specifications - What's the Quality of My Rules?](https://www.youtube.com/watch?v=PjUua2Hi1GA&t=433s) and [Webinar: How to Prevent Prover Timeouts](https://www.youtube.com/watch?v=mntP0_EN-ZQ).

The QA process for formal verification can be categorized into three main stages:

<a name="manual-mutations"></a>
### Manual Mutations

A manual mutation involves altering the contract code, typically represented as the original contract file with specific modifications. To validate the effectiveness of your property, it should pass with the unaltered contract code and fail (or be violated) when tested with the manual mutation. This approach confirms that your rule is not superficial and functions as intended.

One practical method for testing with manual mutations is leveraging `git` commands. For instance, after introducing a mutation in `ActivePool.sol`, run `certoraRun` as usual. Then, use `git restore packages/contracts/contracts/ActivePool.sol` to revert to the original contract file. 

In my specifications, you'll notice notations like `[%number%]` in the comment blocks of properties. These are my internal markers indicating the filename of the manual mutation used for testing. For example:

```spec
// [2] In initialized state authority address should not be zero
use invariant authNoOwnerInitializedAndAddressSetInConstructor;

// [g4] authority() should be set in constructor from CdpManager
use invariant authNoOwnerSetAuthorityFromCdpManagerInConstructor;

// [3] feeBps should not be greater than MAX_FEE_BPS 
use invariant erc3156FlashLenderFeeBpsNotGtMaxfeeBps;
```

Here, `[2]` references the property tested with the mutation file at [certora/mutations/ActivePool/2.sol](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/mutations/ActivePool/2.sol#L58-L78), and `[g4]` indicates a `Gambit` autogenerated mutation in [certora/mutations/ActivePool_gambit174/4.sol](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/mutations/ActivePool_gambit174/4.sol#L62-L63).

It’s advisable to perform this test for each property immediately after its development.

<a name="coverage-information"></a>
### Coverage Information

Historically, mutation testing in Solidity relied on [Gambit](https://github.com/Certora/gambit), an open-source mutation generator ([documentation](https://docs.certora.com/en/latest/docs/gambit/index.html)). Nowadays, `Gambit` has been integrated into the more comprehensive `certoraMutate` engine ([documentation](https://docs.certora.com/en/latest/docs/gambit/mutation-verifier.html)), which you should focus on. This tool combines the functionalities of `Gambit`, `certoraRun`, a server infrastructure designed for extensive testing, and a user-friendly [dashboard](https://mutation-testing.certora.com/).

Configuring `certoraMutate` is similar to setting up the prover using `conf` files. For example, the configuration for `ActivePool.sol` is available at [certora/confs/gambit/ActivePool.mconf](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/confs/gambit/ActivePool.mconf):

```json
{ 
   "gambit": {                                                                   
      "filename" : "../../../packages/contracts/contracts/ActivePool.sol",
      "num_mutants": 0
   },                                                                           
   "manual_mutants": {                                                          
      "../../../packages/contracts/contracts/ActivePool.sol" : "../../mutations/ActivePool"
   }                                                                            
}  
```

Execute with `certoraMutate --prover_conf certora/confs/ActivePool_verified.conf --mutation_conf certora/confs/gambit/ActivePool.mconf`.

Setting `num_mutants` to zero means only manual mutations will be executed. The process involves replacing the original `ActivePool.sol` with each file in the [../../mutations/ActivePool](https://github.com/alexzoid-eth/2023-10-badger-fv/tree/main/certora/mutations/ActivePool) directory and running `certoraRun`.

Alternatively, for more in-depth coverage analysis, you can add the `--coverage_info [none|basic|advanced]` flag to `certoraRun`. The `advanced` option provides more detailed insights but is slower. An example of this can be seen [here](https://prover.certora.com/output/52567/79c0d8b34f934d4bac6142136a68ee3f?anonymousKey=d4c7e909bc09c9acaee84c109ed83c3aab93a2d0), where `certoraRun certora/confs/ActivePool_verified.conf --rule sanity --coverage_info advanced` was executed. To view this, first click `Job Info` on the left panel, then `Unsat Core page` on the right side of the window.

![](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/images/4.png)

<a name="identifying-problems"></a>
### Identifying Problems

To guarantee the quality of your property specifications, consider using the `--rule_sanity` option, which performs automatic checks to identify common errors in specifications. Examples include unreachable `assert` statements due to reverts, `asserts` that are always `true`, invariants that invariably pass, or superfluous `require` and `assert` statements.

For detailed insights into these checks, refer to the [sanity checks documentation](https://docs.certora.com/en/latest/docs/prover/checking/sanity.html).

<a name="automation"></a>
## Automation

Minimizing the time spent on proving properties and testing them with manual mutations is crucial. To facilitate this, I developed several scripts for efficient workflow management.

1. **Generating Manual Mutations**:
   - The script [certora/mutations/addMutation.sh](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/mutations/addMutation.sh) assists in creating manual mutations. After manually editing the contract file, this script can generate a mutation. It requires two input parameters: the name of the configuration and the relative path to the contract file.
   - Example usage: `./certora/mutations/addMutation.sh ActivePool ./packages/contracts/contracts/ActivePool.sol`
   - The script copies the mutated contract file into the `certora/mutations/ActivePool/` directory and adds a mutation comment, similar to the output of the `git diff` command.

   See [certora/mutations/ActivePool/2.sol](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/mutations/ActivePool/2.sol#L46-L83) for an example mutation:
   ```solidity
       constructor(
           address _borrowerOperationsAddress,
           address _cdpManagerAddress,
           address _collTokenAddress,
           address _collSurplusAddress,
           address _feeRecipientAddress
       ) {
           borrowerOperationsAddress = _borrowerOperationsAddress;
           cdpManagerAddress = _cdpManagerAddress;
           collateral = ICollateralToken(_collTokenAddress);
           collSurplusPoolAddress = _collSurplusAddress;
   
   /**************************** Diff Block Start ****************************
   diff --git a/packages/contracts/contracts/ActivePool.sol b/packages/contracts/contracts/ActivePool.sol
   index 40b6a1f..1859c0b 100644
   --- a/packages/contracts/contracts/ActivePool.sol
   +++ b/packages/contracts/contracts/ActivePool.sol
   @@ -58,7 +58,7 @@ contract ActivePool is IActivePool, ERC3156FlashLender, ReentrancyGuard, BaseMat
    
            // TEMP: read authority to avoid signature change
            address _authorityAddress = address(AuthNoOwner(cdpManagerAddress).authority());
   -        if (_authorityAddress != address(0)) {
   +        if (true) {
                _initializeAuthority(_authorityAddress);
            }
    
   **************************** Diff Block End *****************************/
   
           feeRecipientAddress = _feeRecipientAddress;
   
           // TEMP: read authority to avoid signature change
           address _authorityAddress = address(AuthNoOwner(cdpManagerAddress).authority());
           if (true) {
               _initializeAuthority(_authorityAddress);
           }
   
           emit FeeRecipientAddressChanged(_feeRecipientAddress);
       }
   ```

2. **Executing the Prover Against Your Rule**:
   - The [certora/mutations/checkMutation.sh](https://github.com/alexzoid-eth/2023-10-badger-fv/blob/main/certora/mutations/checkMutation.sh) script is designed to run the prover against your rule. This can be done in two ways:
     - To test the rule against the original contract: `./certora/mutations/checkMutation.sh ActivePool ./packages/contracts/contracts/ActivePool.sol`. Optional parameters like `--rule sanity` are supported.
     - To test against a mutated contract: `./certora/mutations/checkMutation.sh ActivePool ./packages/contracts/contracts/ActivePool.sol 2`. Here, '2' indicates the mutation file name to be used (`certora/mutations/ActivePool/2.sol`).

3. **Workflow Steps**:
   - **Step 1**: Implement a rule, such as `sanity`.
   - **Step 2**: Verify that the rule is not violated using the original contract file using `checkMutation.sh`.
   - **Step 3**: Introduce a manual mutation in the `ActivePool.sol` contract.
   - **Step 4**: Save the mutated file into `certora/mutations/ActivePool/` and restore the original using `addMutation.sh`.
   - **Step 5**: Validate that the rule is violated with the mutated contract file using `checkMutation.sh`.

By integrating these scripts into your development process, you can significantly accelerate the implementation and testing of rules.

<a name="conclusion"></a>
## Conclusion

Formal verification fulfills a crucial dual role. On one hand, it validates code functionality through mathematical analysis, akin to traditional testing. On the other, it acts as an advanced tool for auditing, ensuring not just functionality but also the security of the code. This comprehensive approach is positioning formal verification as a standard practice in the realm of DeFi.

# Gecko Fuzz 🦎

# 🦎 Introduction
Gecko is a DAO leveraging crowd-sourced computation power to achieve **fast**, **accurate** and **cheap** auomated auditing on the Solana network, through a decentralised fuzzing infrastructure. 

The importance of auditing has grown significantly in recent years as organizations strive to ensure the integrity and security of their systems. However, despite the importance of auditing, it remains challenging, with many auditing companies struggling to provide comprehensive and accurate reports.

The use of human auditors by auditing firms presents several challenges, including the high costs of recruiting and training qualified personnel and the potential for human error. With the increasing complexity of software systems and the growing volume of data to be analyzed, manual audits can become increasingly time-consuming and error-prone. On the other hand, automated auditing solutions also present their own set of challenges. These solutions typically require high computational power and incur high running time overhead. Thus, many traditional automated auditing tools sacrifice completeness and soundness of the analysis for faster response time, resulting in both false negative and positive results.

In contrast, Gecko aims to parallelize novel automated program analysis techniques to gain accurate results in a reasonable amount of time. To achieve high parallelism with low costs, the Gecko Fuzz platform allows the public to contribute computation power to accomplish the automated auditing in return for token rewards. In the meantime, all the program analysis intermediate statistics and waypoints are verified and stored on Solana, which can finally be leveraged to mint the auditing reports.

Unlike traditional collaborative manual auditing platforms Gecko uses sound automated program analysis (e.g., fuzzing and symbolic execution) techniques to provide accurate auditing reports. Since the program analysis results and intermediate waypoints can be easily verified through a fully automated oracle, the manual confirmation process is no longer needed. While it is impossible to quantify the performance of human auditors, Gecko can quantify the auditing progress and completeness of auditing reports based on metrics backed with on-chain data.

The Gecko platform can offer two key benefits to the ecosystem. Firstly, it allows Solana developers to to access low-cost, highly accurate auditing reports for their projects with on-chain guarantees. Secondly ...


# Technical Details
### Partitioning Plan Synthesis
By converting a program into LLVM bytecode, we can create a weighted control flow graph (CFG) of it with the weight of each edge as relative difficulty of exploring such an edge. Graph partitioning algorithms can then partition the CFG into sub-trees, with the starting node of the CFG as the root of each tree. The partition plan can be concisely represented in O(log n) bytes, where `n` is the size of the CFG, making it possible to be fit into an on-chain variable.

To determine the difficulty of exploring each edge in the CFG, we utilize static analysis tools. We pinpoint the comparison instruction that leads to the edge and determine the domain size of both the LHS and RHS. The domain size represents the likelihood of program execution failing into either side if the input is randomly selected. Currently, we use heuristics to determine the domain size. As future work, we can use abstract interpretation algorithms with a constraint solver to calculate it. The exploration difficulty is then estimated by dividing the domain size of the LHS and RHS.

For instance, consider following simple program:

``` rust
// input: Vec<u8>
if (input[0] > 20) { // Line 1
    bug(); // Line 2
} // Line 3
```

The CFG would be:

```
              ┌──────────────┐
       ┌──────┤    Line 1    │
       │      └───────┬──────┘
       │ E2           │ E1
       │      ┌───────▼──────┐
       │   ┌──┤    Line 2    |
       │   │  └──────────────┘          
       │   │ E3              
┌──────▼───▼───┐ 
│    Line 3    |
└──────────────┘
```

Given `u8` domain is 256, weight (exploration difficulty) of E1 is `(256 - 20) / (256 + 20)` and `E2` is `(256 + 20) / (256 - 20)`. By intuition, E2 is indeed more likely to be explored than E1. As there is no comparison instruction in during transition of E3, the exploration difficulty is 0, meaning as long as we can reach Line 2, we can reach Line 3.

### Dynamic Program Anannslusi (DPA)
Gecko compiles the Solana contract into LLVM bytecode and leverages fuzz testing techniques, which involve sending random input to the program. This method, also known as heuristic search, aims to achieve 100% code coverage and uncover all vulnerabilities. While infinite time would guarantee zero false negatives, we use formal methods such as symbolic and concolic execution for guiding the fuzz testing search to reduce the time needed. Additionally, by partitioning the program into smaller, more manageable subprograms for each node, we can reduce the time required linearly as the number of nodes increases.

Fuzz testing employs partitioning through the use of an instrumented target. If an input causes execution of code outside the partition plan, the target will terminate. Early termination reduces the time spent exploring code not within the partition, saving significant time. Similarly, symbolic and concolic execution can also conduct early-termination to avoid exploring code outside the partition.

### Reaching Consensus
Verifying partition plans and interesting test cases can be costly or even impossible on the chain. Thus, validator nodes use off-chain oracles. Gecko uses rollup techniques to move the oracle results onto the chain and reach consensus. Specifically, an optimistic rollup pallet is implemented to achieve consensus on partition plans and interesting test cases. Once a validator node mints a partition plan or an auditor node mints a test case NFT, other validator nodes can submit fraud proofs to challenge it within 50 blocks, or it will be committed. Unlike human auditors or judges, validator nodes can find evidence to challenge false claims in microseconds, as the verification process is automated and inexpensive, making optimistic rollups effective.

**Interactively Partition Plan Verification:** Claimer can create a partition plan by submitting the weighted CFG and list of nodes in the CFG that needs to be divided. A challenger can either challenge the weighted CFG or the partition plan. To challenge the weighted CFG, the challenger submits a fraud-proof consisting of the root node of the minimum differing subtree in the CFG. The chain partially re-generates from that root node to the first child node by looking at branch, jump, and call instructions. That node must equal either party's differing node if at least one party is honest. Although generating full CFG is a costly operation as multiple complex graph analysis algorithm is needed, generating the next node with a known subgraph and context is cheap. To challenge the partition plan, the challenger must submit a better plan. The chain can compare the balance of each subgraph's total weights and determine which is the best partition plan. Comparison is very cheap since the chain only needs to sum up the weight of each subgraph and divide them.

**Interactively Testcase Verification:** Claimer can confirm a test case by submitting the execution trace (a trace of basic blocks hit during execution) of the test case to the chain. The initial fraud-proof consists of the first differing program counter (PC) in execution trace and the state (i.e., dirty page of the memory and stack) before the differing PC. The challenged claimer can dispute the state and find the first differing state interactively with the challenger. When either the differing PC or state is found, the chain will re-execute partially from the state and PC with consensus (i.e., state and PC before the differing ones) using LLVM bytecode virtual machine. Since the execution would lead to a concrete result that is directly equal to that of either challenger or claimer, the chain can decide which party is gaming. Partial re-execution is not costly since the chain only needs to execute the basic block with dispute, which is usually a few simple instructions. A potential future work would be replacing this process with zero-knowledge proof.


# 🔥 Future
- Bring in ZK-SNARKs for testcase sharing.


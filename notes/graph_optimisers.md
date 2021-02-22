# Transferable Graph Optimisers for ML Compilers

## What is the problem

Current ML compilers rely on heuristic based algorithms to solve the optimisation problem however the issue with that is that:

- hard to manage
- sub-optimal solutions for newer model architectures
- sample inefficient
- tackle a single optimisation problem
- and do not generalise well to unseen graphs.

All of these issues make it difficult for them to be deployed in practice.

## Why do we need to address this

The reason this is an issue is because there are high computation requirements for training large models, and we need an efficient use ML accelerators(GPUs and TPUs).

**These ML accelerators map high level computational graph to operations that are executable on the device.** We are mapping a computational graph to the machine code. Some of these device specific optimisations include Tensorflow XLA, Glow, MLIR, and AutoTVM, **map the high level computational graph to operations.**

To do this, it needs to solve a variety of optimisations problems:

- graph rewriting
- assignment of operations on devices
- operator fusion
- layout and tiling of tensors
- operator scheduling

The usual solution is apply heuristics to solve this issue but **they are suboptimal for unseen problems and misses out on opportunities for joint optimisations across tasks.**

It has been shown on previous RL-based approaches that they outperform both domain experts and heuristics but:

- They have substantial computational resources
- Do not generalise well to new graphs.
- solve basically only a single optimisation problem without any knowledge sharing across tasks.
- this is an issue because optimisation problems in stack are usually coupled.
- can result in a near sequential because of bad scheduling

## How is this solved?

The paper suggests an end to end deep RL method in GO for Ml compiler graph optimisations where the learned policy is generalisable to new graphs and transferable across tasks. Consists of:

- **inductive graph-embedding network that encodes operation features**
- dependencies in a trainable graph representation.
- the policy networks transforms the graph into **an optimisation decision with soft attention (which is a deterministic approach)**
- Can be jointly trained in with a supervise reward, **it is also trained over a set of computation graphs instead of one at a time and then fine tuned on new graphs.**
- By moving the learned graph embeddings and optimisations polices it converges fast using less resources. It also uses super positioning, **to optimise a batch containing graphs with very different sizes.**

### Graph Optimisation Problems

ML Computations are represented as computation graphs $G(V, E)$ where the nodes $V$ represent the computations and the edges represent the data flows.

- **Device Placement**
  
  - Given a computational graph, the **goal of device placement is to learn a policy **$\pi : G \rightarrow D$ that assigns a device $D \in D$ for all $G \in G$ **to maximise a reward** $r_{G, D}$ based on the runtime.Given a computational graph, the goal of device placement is to learn a policy  that assigns a device  for all  to maximise a reward  based on the runtime.

- **Operation scheduling**
  
  - The operation in the dataflow graph **is ready to run when the incoming tensors are present in device memory.**
  - A frequently used scheduling strategy that is very inefficient is to use a **FIFO order which maintains a ready queue of operations for each device.** These schedules **exhibit underutilised devices since operations for these are blocked on producer operations.**
  - Instead the paper uses a priority based scheduler and maintains a priority queue of deivce ready operations.
  - This can be set using a learning policy $\pi: G \rightarrow P$ that assigns a scheduling priority $P \in P$ for all operations in the graph to maximise a reward $r_{G, P}$ 

- **Operator Fusion**
  
  - It is the process of merging multiple operations into a single operation.
  - Say we have two operations, K1 and the output is consumed by K2, if these are fused then the intermediate data is **immediately used** for K2 when K1 finishes on the device, **without the need to perform read and write transactions with global memory.**
  - Strategies such as considering operations in topological order can make inefficient fusion decisions and lead to optimal performance. 
  - If the operator fusion is reformatted a priority based ordering problem, then learning a policy becomes $\pi : G \rightarrow F$ that assigns a fusion priority $F \in F$ for all ops in the graph to maximise a reward $r_{G. F}$ defined based on run time, each node is associated with a fusion priority score $F \in F$ where the **size of the fusion priority is a hyperparameter.**
  - An example of where a priority would be useful is in the below example where we have an element wise multiplication, reduction and sigmoid, and if we choose to use operator fusion on the multiply and reduce then the memory would stay in the scratchpad memory for the reduce operation.

- Generalisation
  
  - Old approaches only focus on a single graph
  - The goal is to simultaneously reduce the expected run time of the optimisation over multiple dataflow graphs, $J(\theta) = E_{G \sim G, T \sim \pi_{\theta}(G)} [r_{G, T}]$
  - T in this case is ${D, P, F}$ from the above.

### Approach

The proposed policy network $\pi_\theta$ consists of **a graph embedding network that learns the graphical representation** $h_{G}$ of any computational graph **and a scalable decision network that learns an optimisation strategy, ** $p(y | G)$ over the given graph representation.

$p(y | G) = \prod_{i = 1..N} p(y_{i}| h_{G}, y_{i - 1}, y_{i - 2}, ...)$

**However this is difficult to do since the number of nodes N can be as large as 10K.** And there is no way to parrellise this such that it is consistent.

Instead this is done using as iterative but non auto-regressive approach as an approximation:

$p(y^{(t)} | G) = \prod_{i = 1..N}p(y_{i}^{(t)} | h_{G}, y^{(t - 1)})$

The architecture is said to be invariant over the underlying graph topology and can be applied over multiple input graphs, GO optimises the objective described in section 3 using the **Proximal Policy Optimisation (PPO)**. 

---
type: Paper Note
title: Benchmarking Agentic Workflow Generation
description: A study of how well LLM agents decompose tasks into executable node chains and dependency-aware workflow graphs.
resource: https://arxiv.org/abs/2410.07869
pdf: https://arxiv.org/pdf/2410.07869v3
tags:
  - llm-agents
  - workflow-generation
  - planning
  - evaluation
  - benchmarks
timestamp: 2026-07-12T13:12:27+09:00
status: reviewed
---

# Overview

## One-sentence takeaway

LLMs can list plausible subtasks more reliably than they can identify the dependencies and parallelism between those subtasks; explicit workflows nevertheless improve downstream agent accuracy and reduce inference time.

## Research question

Complex agent tasks require more than producing the right final answer. An agent must decompose a task into executable subtasks, order dependent work correctly, and recognize work that can run in parallel. Existing evaluations tend to measure end-to-end success, cover only a narrow task family, represent plans as linear sequences, or rely on subjective LLM judging.

The paper asks how to evaluate workflow-generation ability directly across varied agent scenarios and how much harder graph planning is than linear planning.

## Proposed approach

### WorfBench

WorfBench takes a task description and a candidate action list as input. The expected output is constructed in two stages:

1. A **Node Chain** lists the minimum executable subtasks in one valid order.
2. A **Workflow Graph** adds directed dependency edges between those nodes, producing a directed acyclic graph (DAG) with `START` and `END` nodes.

The benchmark covers four scenario types:

- **Function Call:** multi-step API and tool use from ToolBench and ToolAlpaca, with Seal-Tools held out.
- **Embodied:** interaction trajectories from ALFWorld, WebShop, and operating-system tasks, with InterCodeSQL held out.
- **Problem-Solving:** mathematical, commonsense, and multimodal reasoning tasks from LUMOS.
- **Open-Grounded:** general procedural tasks derived from WikiHow.

The final dataset contains 18,679 training instances and 2,146 test instances. Held-out tasks make up 33.69% of the test set, and the average workflow contains 4.17 nodes.

For Function Call tasks, the source datasets provide gold function calls. The authors reverse-engineer a Thought for each call with GPT-4, execute the call to obtain an Observation, and use the resulting Thought-Action-Observation trajectory to synthesize one natural-language node per step. After a Node Chain is available, GPT-4 predicts dependency edges between numbered nodes.

Quality control filters Node Chains by checking whether each node retrieves the corresponding gold function. It filters Workflow Graphs by deterministically topologically sorting them and requiring the result to match the Node Chain. The authors then remove trivial workflows and manually verify the test set.

### WorfEval

WorfEval separates semantic node alignment from sequence and graph evaluation:

1. Sentence-BERT embeds gold and predicted node descriptions.
2. Cosine similarities below a threshold of 0.6 are discarded.
3. Maximum-weight bipartite matching creates a one-to-one mapping between semantically corresponding nodes.
4. Longest Increasing Subsequence (LIS) measures how much of the predicted Node Chain follows a valid topological order of the gold graph.
5. Maximum Common Induced Subgraph (MCIS) measures how many matched nodes also preserve the gold dependency structure.
6. Precision, recall, and F1 normalize the matched sequence or subgraph by both the predicted and gold node counts.

## Main findings

### Graph planning is consistently harder than linear planning

Every evaluated model scores substantially lower on Workflow Graph generation than on Node Chain generation.

| Model | Node Chain F1 | Workflow Graph F1 |
|---|---:|---:|
| GPT-4 | 67.32 | 52.47 |
| Claude 3.5 | 66.70 | 52.53 |
| o1-preview | 66.70 | 51.63 |
| Qwen-2-72B | 67.24 | 50.46 |
| Llama-3.1-70B | 64.60 | 49.59 |

Across models, the graph score trails the chain score by roughly 15 to 20 points. Open-Grounded tasks are particularly difficult: the best reported result, from Claude 3.5, is 61.33 Node Chain F1 and 42.88 Workflow Graph F1.

### More complex workflows reduce performance

GPT-4's chain and graph scores generally decline as the number of nodes and edges increases. The models therefore remain inadequate for long, dependency-rich workflows even when they perform reasonably on short plans.

### Gold nodes do not solve dependency prediction

When GPT-4 receives the gold Node Chain and predicts only edges, its average graph F1 rises from 52.47 to 74.63. This shows that node decomposition accounts for substantial error, but dependency prediction remains imperfect even after the correct subtasks are supplied.

### Fine-tuning helps in-distribution more than out-of-distribution

Fine-tuned Qwen-2-7B and InternLM-2.5-7B models outperform GPT-4 strongly on held-in tasks. Their advantage is much smaller on held-out Seal-Tools tasks, and they fall behind stronger models on the more complex InterCodeSQL tasks. The authors conclude that fitting workflow examples alone does not produce robust, general workflow-planning ability.

### Workflows improve downstream planning

Using generated workflows as structured prior knowledge improves GPT-4's ALFWorld success rate from 27.14 to 40.71 on seen tasks and from 28.36 to 47.01 on unseen tasks. Workflows generated by a 7B model can even improve a 72B model, suggesting a **weak-guide-strong** pattern in which a smaller specialized model guides a larger general model.

Graph execution also reduces average tool-use time by approximately one-fifth to one-third by running independent nodes in parallel. Workflow guidance reduces average planning steps by limiting blind exploration.

## Contributions

- Evaluates task decomposition as an intermediate agent capability rather than only measuring final task success.
- Introduces a multi-scenario benchmark with DAG workflows that represent both sequential dependencies and parallelism.
- Replaces purely subjective judging with semantic alignment followed by sequence- and graph-matching algorithms.
- Demonstrates that workflows can improve both downstream accuracy and inference efficiency.

## Limitations and critique

The authors note that source queries may contain quality problems, the benchmark omits domains requiring heterogeneous action types, and its natural-language DAG representation excludes code-oriented plans, choices, loops, and iterative replanning based on environmental feedback.

Additional concerns follow from the benchmark design:

- GPT-4 synthesizes both nodes and edges, so the gold workflows may reflect its preferred decomposition granularity and dependency assumptions.
- A task may admit several functionally equivalent workflows, but structural comparison to a limited gold graph can penalize valid alternatives.
- MCIS deliberately penalizes extra and missing edges, but this strictness can score execution-equivalent graphs differently.
- The evaluation samples up to 20 topological orders rather than exhaustively enumerating them for highly parallel graphs, trading exactness for tractability.
- The paper does not ablate the Thought, Action, and Observation components independently, so their individual contribution to Function Call node quality remains unclear.

# Key Concepts

## Node Chain

A Node Chain is a linear sequence of natural-language subtasks at minimum executable granularity. It is not necessarily the only correct order: it represents one topological ordering of the underlying workflow graph.

## Workflow Graph

A Workflow Graph is a DAG whose nodes are subtasks and whose directed edges express execution dependencies. If there is an edge from node A to node B, A must be executed before B. Nodes without a dependency path between them may be executed in parallel.

## Semantic node matching

Gold and predicted nodes are free-form natural-language descriptions, so exact string matching would reject paraphrases. WorfEval embeds each node with Sentence-BERT (`all-mpnet-base-v2`) and calculates a rectangular similarity matrix of size:

$$
|V^g| \times |V^p|
$$

where $V^g$ and $V^p$ are the gold and predicted node sets. Similarities below $\beta=0.6$ become zero. Maximum-weight bipartite matching then selects the best one-to-one alignment, preventing one vague predicted node from receiving credit for several gold nodes.

## Longest Increasing Subsequence

After semantic matching, predicted nodes can be represented by the indexes of their corresponding gold nodes. LIS finds the largest subsequence whose relative order agrees with a valid topological ordering of the gold graph. It evaluates ordering, not edges directly.

Because parallel nodes can produce many valid orders, WorfEval compares against sampled topological orders of the gold graph and keeps the largest match. Appendix A.9 states that up to 20 orders are sampled for computational efficiency.

## Maximum Common Induced Subgraph

MCIS finds the largest set of matched nodes whose complete edge structure is identical in the predicted and gold graphs. “Induced” means that all edges among the selected nodes matter: a missing edge, an extra edge, or an edge in the wrong direction can prevent the whole node set from matching.

## Precision, recall, and F1

For Node Chains, if $l$ is the longest valid subsequence:

$$
P_{chain}=\frac{l}{|V^p|},\qquad
R_{chain}=\frac{l}{|V^g|}
$$

For Workflow Graphs, if $k$ is the number of nodes in the MCIS:

$$
P_{graph}=\frac{k}{|V^p|},\qquad
R_{graph}=\frac{k}{|V^g|}
$$

Both use the harmonic mean:

$$
F1=\frac{2PR}{P+R}
$$

Precision penalizes unnecessary predicted nodes, while recall penalizes omitted required nodes. F1 requires both to be high.

# Reading Questions

## Why generate Observations for Function Call tasks, and what data does the benchmark ultimately need?

### Answer

The desired benchmark output is not the Thought-Action-Observation trajectory itself. Each instance ultimately needs:

- **Input:** a task description and candidate action list.
- **Output:** a Node Chain and a Workflow Graph represented by its dependency edges.

ToolBench and ToolAlpaca provide gold function calls but not natural-language workflow nodes. The authors reconstruct each call as a ReAct-style step: GPT-4 reverse-engineers the Thought, the function is executed to obtain an Observation, and GPT-4 summarizes the step as a node. Repeating this process converts a sequence of function calls into a Node Chain.

For example, consider a task asking for the food minister of the state containing Kankumbi:

```text
Thought: I first need to determine which state Kankumbi is in.
Action: search_location("Kankumbi")
Observation: Kankumbi is located in Maharashtra.
```

The desired node is not the raw call `search_location(...)` and not the result sentence “Kankumbi is located in Maharashtra.” It is a planning-level task such as:

> Identify the state in which Kankumbi is located.

The next call might use `Maharashtra` to find the food minister. In that sense, the Observation shows the intermediate information that became available, while the node describes the work that produced it.

The paper does not explicitly isolate why Observation is necessary. A reasonable interpretation is that it restores the execution context of the source trajectory and can show what information or state the Action produced. This may make a call's role easier to understand than its function name and arguments alone.

However, Appendix A.7 qualifies that interpretation: its prompt says to ignore API-call errors, focus on the last Thought and Action, and describe the planned task rather than the API name or execution result. Observation is therefore best understood as contextual construction data, not as the target content of the node, and the paper provides no ablation proving how much it contributes.

## Why does WorfEval use Sentence-BERT, and how does it compare workflows with different node counts?

### Answer

Nodes are natural-language descriptions, so semantically equivalent nodes may use different wording. Sentence-BERT provides semantic embeddings that allow paraphrases to match even when their strings differ.

WorfEval compares every gold node with every predicted node, producing a $|V^g| \times |V^p|$ cosine-similarity matrix. Scores below 0.6 are removed. It then treats this matrix as a weighted bipartite graph and finds a maximum-weight one-to-one matching.

For example, suppose there are three gold nodes and four predicted nodes:

| | Prediction 1 | Prediction 2 | Prediction 3 | Prediction 4 |
|---|---:|---:|---:|---:|
| Gold 1 | 0.91 | 0.20 | 0.10 | 0.15 |
| Gold 2 | 0.30 | 0.88 | 0.72 | 0.10 |
| Gold 3 | 0.15 | 0.40 | 0.82 | 0.55 |

After applying the 0.6 threshold, the useful candidates are `Gold 1 <-> Prediction 1`, `Gold 2 <-> Prediction 2 or 3`, and `Gold 3 <-> Prediction 3`. Maximum-weight one-to-one matching selects:

```text
Gold 1 <-> Prediction 1  (0.91)
Gold 2 <-> Prediction 2  (0.88)
Gold 3 <-> Prediction 3  (0.82)
Prediction 4 remains unmatched.
```

Choosing `Gold 2 <-> Prediction 3` merely because 0.72 is acceptable would prevent the stronger `Gold 3 <-> Prediction 3` match. The bipartite algorithm optimizes the alignment globally rather than choosing each row independently.

The one-to-one constraint prevents double counting. When the predicted and gold workflows contain different numbers of nodes, some nodes remain unmatched. The later precision and recall denominators use the full predicted and gold node counts, so extra predictions reduce precision and missing gold nodes reduce recall.

Sentence-BERT therefore does not directly determine the final score. It establishes semantic correspondence; LIS or MCIS then tests ordering or structure, and F1 normalizes the surviving match.

## What do LIS and MCIS do for Node Chains and Workflow Graphs?

### Answer

**LIS asks whether matched nodes appear in a valid relative order.** Suppose the gold order is `1, 2, 3, 4` and the semantically aligned prediction is `1, 3, 4, 2`. The longest order-preserving subsequence is `1, 3, 4`, so its length is three. Nodes do not need to be contiguous.

A DAG can have several valid topological orders because independent nodes may be interleaved. WorfEval compares the prediction with sampled valid topological orders and retains the longest LIS. This avoids penalizing one valid ordering merely because it differs from another valid ordering.

**MCIS asks whether matched nodes preserve the complete dependency structure.** For a chosen node set, the induced subgraphs must agree on every edge and non-edge among those nodes. It therefore detects missing dependencies, unnecessary dependencies, reversed edges, and mistaken serialization of parallel work.

For a concrete graph example, suppose the gold workflow is:

```text
1 -> 3
2 -> 4
```

Nodes 1 and 2 are independent, as are the two branches. Now suppose the predicted graph is:

```text
1 -> 3
1 -> 2   (extra dependency)
2 -> 4
```

All four node descriptions may match semantically, but the four-node induced graphs do not match because the prediction adds `1 -> 2`. The subsets `{1, 3}` and `{2, 4}` do match. MCIS searches for the largest matching subset, including every edge and non-edge among its selected nodes.

A predicted Node Chain can receive a high LIS score while its graph receives a low MCIS score. A fully sequential prediction may be a valid execution order of a partly parallel gold graph, yet still misrepresent which dependencies are actually required.

## Why is F1 important in WorfEval?

### Answer

Models can generate different numbers of nodes, so a raw match count is insufficient.

- Precision asks what proportion of the predicted workflow belongs to the valid matched sequence or subgraph.
- Recall asks what proportion of the gold workflow was recovered.
- F1 combines them with a harmonic mean, which falls sharply if either is low.

For example, if a gold workflow has four nodes, a prediction has six, and three nodes survive semantic and order matching, then:

$$
P=3/6=0.5,\qquad R=3/4=0.75,\qquad F1=0.6
$$

Precision alone would favor producing very few safe nodes. Recall alone would favor producing many speculative nodes as long as the gold nodes are included. F1 rewards workflows that are both complete and economical. In WorfEval, the match count already incorporates semantic plus sequential or structural correctness, so the final F1 also reflects order or dependency errors.

The two failure extremes make the trade-off clearer:

- If the model predicts only one correct node from a four-node gold workflow, precision is 1.0 but recall is 0.25. The workflow is accurate but incomplete.
- If the model predicts ten nodes containing all four gold nodes, recall is 1.0 but precision is 0.4. The workflow is complete but wasteful and potentially unsafe.

In both cases F1 stays low because an executable workflow must avoid both omissions and unnecessary steps.

## What do Query and Trajectory mean in the Function Call node-generation prompt?

### Answer

**Query** is the original multi-step user request that the workflow must accomplish. It provides the overall purpose needed to interpret an individual tool call.

**Trajectory** is a segment of the solution history in conversational ReAct form, consisting of Thought, Action, and Observation steps. For Function Call tasks, it is reconstructed from the source dataset's gold function calls.

Appendix A.7 asks GPT-4 to summarize the last step of the supplied trajectory segment as a clear, concise node task. The output should describe the planning-level subtask associated with the current Action rather than reproduce a specific API name or state the returned result.

For example:

```text
Query:
Who is the current food minister of the state where Kankumbi is located?

Trajectory:
Thought 1: I need to find the state containing Kankumbi.
Action 1: search_location("Kankumbi")
Observation 1: Kankumbi is in Maharashtra.

Thought 2: I can now look up the food minister of Maharashtra.
Action 2: search_food_minister("Maharashtra")
Observation 2: ...
```

When the trajectory segment ends at step 1, GPT-4 should generate a node such as “Identify the state in which Kankumbi is located.” When it ends at step 2, it should generate “Find the current food minister of that state.” The Query explains why these calls matter, while the Trajectory identifies which part of the overall task the latest step performs.

Conceptually, successive trajectory prefixes can yield successive nodes:

```text
Query + trajectory through step 1 -> Node 1
Query + trajectory through step 2 -> Node 2
Query + trajectory through step 3 -> Node 3
```

The Query supplies global intent; the Trajectory supplies local execution context; the generated node abstracts the latest step into an API-independent subtask.

# Open Questions

- How sensitive are WorfEval rankings to the Sentence-BERT model and the fixed similarity threshold of 0.6?
- How often does topological-order sampling underestimate the best LIS for highly parallel graphs?
- Could execution-based equivalence recognize valid alternative workflows that MCIS penalizes structurally?
- How much does each reconstructed component—Thought, Action, and Observation—contribute to Function Call node quality?
- Would iterative workflows with branches, loops, and environmental feedback change the observed gap between chain and graph planning?

# References

1. [Benchmarking Agentic Workflow Generation](https://arxiv.org/abs/2410.07869)
2. [WorfBench code and dataset](https://github.com/zjunlp/WorfBench)

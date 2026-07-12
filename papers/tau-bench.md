---
type: Paper Note
title: "τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains"
description: A benchmark for evaluating whether language agents can converse with users, operate domain APIs, follow policies, and do so consistently across repeated trials.
resource: https://arxiv.org/abs/2406.12045
pdf: https://arxiv.org/pdf/2406.12045v1
tags:
  - language-agents
  - tool-use
  - benchmarks
  - user-simulation
  - reliability
timestamp: 2026-07-12T17:00:00+09:00
status: reviewed
---

# Overview

## One-sentence takeaway

τ-bench evaluates an agent as an interactive service system rather than as a one-shot function caller: it must discover a simulated user's intent through dialogue, obey a natural-language domain policy, update a database through APIs, communicate required information, and succeed consistently when the same scenario is repeated.

## Research question

Can a language-model agent reliably resolve realistic customer-service requests when the necessary information is distributed across a conversation, a hidden database, API observations, and domain-specific rules?

Earlier tool-use benchmarks commonly gave the complete request up front and judged a single call. τ-bench instead tests three deployment-relevant requirements together:

1. multi-turn interaction with a user and programmatic tools;
2. reasoning under domain-specific procedures and restrictions;
3. consistency under semantically equivalent but stochastically different conversations.

## Proposed approach

### Environment

Each task is modeled as a POMDP. The agent can send natural-language messages to an LM-simulated user or invoke database APIs. It sees the domain policy as its system prompt, but can observe the database only through tools. The user sees the conversation with the agent but not the agent-tool interaction history.

The benchmark's four concrete components are:

- JSON databases;
- Python read/write APIs;
- Markdown domain policies;
- JSON task instances containing a hidden user instruction and ground-truth annotations.

The v1 paper implements two domains:

| Domain | Database | APIs | Tasks |
|---|---|---:|---:|
| τ-retail | 500 users, 50 products, 1,000 orders | 7 write, 8 non-write | 115 |
| τ-airline | 500 users, 300 flights, 2,000 reservations | 6 write, 7 non-write | 50 |

### Task episode and reward

A task instance supplies the user simulator with a hidden identity, intent, preferences, and fallback behavior. The simulator turns this into a stochastic conversation. The agent can interleave user messages and tool calls until the user emits `###STOP###` or the action limit is reached.

The binary reward is

$$
r = r_{\text{action}} r_{\text{output}}.
$$

`raction` checks that the final database exactly matches the annotated goal state. `routput` checks that required answer strings appear in agent-to-user messages. Different dialogue trajectories and read calls are therefore allowed if they produce the same required outcome.

This evaluator is efficient but incomplete: an agent can receive reward 1 after making the right write without first obtaining the explicit confirmation required by policy.

### Benchmark construction

1. **Stage I:** humans co-design simplified but realistic schemas, APIs, and policies.
2. **Stage II:** GPT-4 helps generate scalable synthetic database entries; humans fix the generated sampling code.
3. **Stage III:** humans author task instructions and ground truth, run a GPT-4 Turbo function-calling agent, inspect trajectories, and repeatedly refine ambiguous instructions or annotations.

### Agent methods

- **Function calling (FC):** native structured tool calls; the domain policy is the system prompt.
- **Text ReAct:** text completion alternates `Thought` and JSON-like `Action` fields.
- **Act-only:** the ReAct ablation without the `Thought` field.

Agents receive at most 30 actions per task. The agent temperature is 0 and user-simulator temperature is 1.

## Main findings

In the paper's June 2024 v1 evaluation, native function calling performed best, but no tested model was reliable:

| Model (FC) | τ-retail pass^1 | τ-airline pass^1 | Domain average |
|---|---:|---:|---:|
| GPT-4o | 61.2% | 35.2% | 48.2% |
| GPT-4 Turbo | 57.7% | 32.4% | 45.1% |
| Claude 3 Opus | 44.2% | 34.7% | 39.5% |
| GPT-3.5 Turbo | 20.0% | 10.8% | 15.4% |

For GPT-4o on retail, pass^8 falls below 25% despite pass^1 exceeding 60%. Native FC consistently beats the text-formatted methods; within the text methods, ReAct consistently beats Act-only.

The manual analysis of 36 agent-caused GPT-4o retail failures groups them into:

- wrong tool arguments or wrong/missing information (about 55% together);
- incorrect domain decisions (25%);
- partial resolution of compound requests (19.4%).

Removing the policy reduces GPT-4o from 33.2% to 10.8% on airline, but only from 61.2% to 56.8% on retail. This suggests that the complex, ad-hoc airline rules matter much more, while many successful retail actions can be guessed from common sense.

## Contributions

- Integrates open-ended user dialogue, tool use, hidden database state, and domain-policy compliance in one executable benchmark.
- Uses annotated goal states to allow diverse trajectories while retaining fast rule-based evaluation.
- Introduces pass^k to measure repeated reliability rather than lucky success under retries.
- Provides a modular recipe for constructing new domains and a failure analysis that isolates database reasoning, rule following, and compound-request tracking.

## Limitations and critique

### Limitations stated by the authors

- User instructions may contain typos or ambiguities.
- The LM user may calculate poorly, forget context, or deviate from its hidden instruction.
- Users lack domain knowledge and can authorize a suboptimal action after an agent's misleading explanation.
- Exact final-state evaluation does not detect every process-level policy violation.
- Task construction and annotation require substantial manual domain understanding.
- Refining prompts with a GPT-4 Turbo FC agent introduces model-specific curation bias.
- The synthetic databases and policies are simplified relative to production systems.

### Critique and later-status caveat

The benchmark deliberately mixes model capability with the quality of the surrounding agent harness. Retrieval, deterministic validators, state machines, structured task tracking, and server-side computation can prevent many v1 failures without improving the base model itself. A modern system-level score is therefore not directly a measurement of unaided model reasoning.

The original repository now warns that its airline and retail tasks are outdated and directs users to τ³-bench for fixed tasks and new domains. Scores from the paper, the original leaderboard, and later benchmark revisions should not be compared without matching the task version, user simulator, tool interface, number of trials, and inference settings.

# Key Concepts

## Policy enforcement has two layers

Some restrictions are enforced by API code. For example, a payment ID absent from the user's profile produces an error. Other restrictions are supplied only as natural-language policy. The booking API accepts the number of paid baggage items, but the agent must derive that number from membership tier and cabin class.

This does not mean the latter rules are logically impossible to validate in code. The benchmark intentionally leaves some enforceable logic outside the API so that policy understanding and judgment remain part of the evaluated agent behavior, similar to real systems in which a backend validates IDs and types but does not encode every operational rule.

## Task instance, episode, trajectory, and result

- **Task instance:** one evaluation scenario specification: hidden user instruction, ground-truth write actions, and optional required outputs.
- **Episode/trial:** one stochastic execution of that scenario.
- **Trajectory:** the resulting user-agent messages, tool calls, and observations.
- **Result/reward:** whether that execution reached the annotated state and returned the required information.

Repeating one task instance produces multiple trajectories and binary results; those repetitions support pass^k.

## pass^k and pass@k

For each task, suppose $n$ trials yield $c$ successes. The paper uses

$$
\widehat{\text{pass\textasciicircum} k}
= \frac{\binom{c}{k}}{\binom{n}{k}},
\qquad
\widehat{\text{pass@}k}
= 1-\frac{\binom{n-c}{k}}{\binom{n}{k}},
$$

then averages each value across tasks.

- pass^k asks whether **all** $k$ trials succeed and decreases with $k$.
- pass@k asks whether **at least one** of $k$ trials succeeds and increases with $k$.

The first measures reliability; the second measures whether retries can discover a success.

## Unbiased estimator

An estimator is unbiased if its average over repeated datasets equals the true quantity it estimates. If independent trials for one task have success probability $p$, then $c/n$ is unbiased for $p$, but $(c/n)^k$ is generally biased for $p^k$.

The combination estimator is unbiased because it averages a 0/1 indicator over every size-$k$ subset of the observed trials. For any fixed subset, the indicator equals 1 exactly when all its trials succeed, an event with probability $p^k$. Linearity of expectation therefore makes the expected average equal to $p^k$.

# Reading Questions

## Are some policies not checked by the API and instead given as natural-language prompts?

### Answer

Yes. The domain policy is placed in the agent's system prompt, and only some restrictions are also enforced by API checks. In the baggage example, the agent reads allowances by membership and cabin, calculates the paid bag count, and supplies it to `book_reservation`; the API is not described as independently validating that policy-derived number.

For example, if two economy passengers each receive one free bag and want three bags total, the agent should pass one paid bag. A structurally valid call that says zero paid bags can still violate policy. More precisely, this is an intentionally un-enforced rule, not necessarily an inherently un-verifiable one.

## What exactly is the JSON task instance in Figure 2(d)?

### Answer

It is the specification of one evaluation problem, not the record of one run. Its `instruction` is a hidden system prompt for the user simulator, not information shown directly to the agent. It defines the user's identity, goals, preferences, fallback choices, and conversational style.

`actions` annotates the desired database writes and `outputs` annotates information the agent must tell the user. Neither is exposed to the agent. In Figure 2(d), the instruction ensures that only the water-bottle return should occur while both possible savings are communicated.

One task instance can generate many episodes because both the user and agent can vary their wording and decisions. The intended semantics and goal state stay fixed.

## How do the binomial-coefficient formulas for pass^k and pass@k work?

### Answer

$\binom{n}{k}$ counts all ways to select $k$ distinct trials from the $n$ observed trials. $\binom{c}{k}$ counts selections containing only successes, while $\binom{n-c}{k}$ counts selections containing only failures.

With $n=5$, $c=3$, and $k=2$:

$$
\text{pass\textasciicircum}2 = \frac{\binom{3}{2}}{\binom{5}{2}}
=\frac{3}{10}=0.3,
$$

because 3 of the 10 pairs contain two successes. Meanwhile,

$$
\text{pass@}2
=1-\frac{\binom{2}{2}}{\binom{5}{2}}
=1-\frac{1}{10}=0.9,
$$

because only one pair contains two failures; all other pairs contain at least one success.

Sampling is without replacement from the finite observed trials, which is why pass^2 is $3/5 \times 2/4=0.3$, not $(3/5)^2=0.36$.

## What does “unbiased estimates” mean here?

### Answer

It means that if one repeatedly generated a fresh set of $n$ trials under the same true task success probability $p$, then averaged the reported estimator, the result would equal the target probability: $p^k$ for pass^k and $1-(1-p)^k$ for pass@k.

The estimate from one finite dataset can be above or below the truth. “Unbiased” does not mean exact on every dataset, nor does it imply low variance.

## Why does averaging the size-k success indicators have expectation p^k?

### Answer

Let $I_S=1$ if every trial in one fixed size-$k$ subset $S$ succeeds, and 0 otherwise. Because the trials are independent and each succeeds with probability $p$,

$$
\mathbb{E}[I_S]
=P(\text{all trials in }S\text{ succeed})
=p^k.
$$

There are $\binom{n}{k}$ such subsets. Their indicators do not need to be mutually independent for their average to be unbiased; linearity of expectation alone gives

$$
\mathbb{E}\left[
\frac{1}{\binom nk}\sum_S I_S
\right]
=\frac{1}{\binom nk}\sum_S\mathbb{E}[I_S]
=p^k.
$$

After a concrete run with $c$ successes, exactly $\binom{c}{k}$ of those indicators equal 1. Therefore the indicator average is exactly $\binom{c}{k}/\binom{n}{k}$.

For $n=3,k=2$, the subsets are `{1,2}`, `{1,3}`, and `{2,3}`. If the realized outcomes are success, success, failure, their indicators are 1, 0, 0, so the average is $1/3=\binom22/\binom32$.

## What happens in Stage III: manual task annotation and validation with agent runs?

### Answer

Yes, Stage III starts after the main schema, APIs, policy, and synthetic database have been built, and its central goal is to make each scenario yield one policy-consistent database outcome. But the refinement target is broader than the instruction alone.

Annotators:

1. write an initial hidden user instruction;
2. run a GPT-4 Turbo FC agent against the simulator;
3. inspect the full trajectory for underspecified preferences, alternative payment methods, multiple valid products, missing fallback behavior, simulator drift, or incorrect ground truth;
4. refine the instruction and, when needed, the actions/outputs annotation or minor environment details;
5. repeat until they judge the outcome unambiguous.

The authors ran each retail task more than 40 times and specifically inspected tasks with zero or low success. Low success is a diagnostic signal, not automatic proof of ambiguity: the annotator must distinguish a genuinely hard task from a bad instruction, simulator error, or wrong annotation.

The unique outcome is determined jointly by the instruction, initial database, domain policy, and information exposed through dialogue. The instruction alone is not an executable mapping to an action.

## Why compare native function calling, text ReAct, and Act-only?

### Answer

The comparison separates two design choices:

- **ReAct vs Act-only** estimates the contribution of an explicit text reasoning trace while keeping the text action format broadly similar.
- **FC vs text methods** compares a model-native structured tool interface against zero-shot prompting that requires the model to emit and parse a textual JSON-like protocol.

The v1 result `FC > ReAct > Act-only` suggests that native structured interfaces reduce formatting/interface burden and that, within an unfamiliar text protocol, an explicit reasoning trace helps connect observations to actions. It does not prove that visible chain-of-thought is the only cause: ReAct also gives the model more tokens and an opportunity to organize intermediate state. A separately added `think` function did not improve FC, which the authors conjecture may reflect a lack of corresponding training.

## Can MCP, Skills, and related techniques reduce the 95.9% input-cost share?

### Answer

Yes, substantially—but this question is specifically about **input cost**, not whether tool calls are correct. The 95.9% figure says that almost all agent price came from repeatedly supplied domain policy, function definitions, conversation history, and tool observations. A modern design has four relevant levers:

| Mechanism | Reduces tokens entering the model? | Reduces billed input price? | Main effect in τ-bench |
|---|---:|---:|---|
| Tool Search / deferred loading | Yes | Yes | Do not place every function schema in the initial context |
| Policy/Skill progressive disclosure | Yes | Yes | Load only the procedure and policy sections relevant to the current intent |
| Prompt caching | No—the model still receives the cached prefix logically | Yes, through cached-input pricing | Reuse the unchanged policy, tools, and system instructions across turns/tasks |
| Programmatic tool calling / server-side code | Yes, especially for tool results | Yes | Filter large DB results and perform joins/calculations before returning evidence to the model |
| Conversation compaction | Yes | Yes | Replace old dialogue and tool observations with a smaller state representation |

### 1. Tool Search directly attacks the long function-definition prefix

OpenAI Tool Search allows a model to search for and load tools only when needed. Functions or an MCP server can be marked `defer_loading: true`; their full definitions are omitted from the initial model context and injected only after discovery. OpenAI's documentation explicitly presents this as a way to avoid loading every definition up front and reduce token usage and cost. Discovered tools are appended so that the existing prompt cache can remain valid.

Anthropic exposes the same core controls. A tool with `defer_loading: true` is excluded from the initial system-prompt tool section, then expanded inline when Claude's tool search returns a reference. Anthropic also states that this preserves the prompt cache.

For τ-airline, instead of sending all booking, cancellation, passenger, baggage, and flight-change schemas every turn, the initial request could expose only:

```text
tool_search
get_user_details
get_reservation_details
```

After learning that the user wants to cancel, the model could search and load only:

```text
cancel_reservation
send_certificate
```

This is a real token reduction. MCP alone is not: an MCP client that eagerly expands every connected server's tool definitions recreates the original cost problem.

### 2. Skills and policy retrieval reduce the policy prefix through progressive disclosure

A Skill can keep only its short name and description in the always-visible catalog, then load the detailed workflow and references after the user's intent is known. A τ-airline implementation might separate `book-flight`, `change-flight`, `cancel-flight`, and `manage-baggage` skills. A cancellation request would not initially pay for the booking and baggage procedures.

The same idea applies to policy retrieval. Keep a small set of cross-cutting invariants always present—for example identity verification, explicit confirmation before a write, and escalation rules—but retrieve cabin-, membership-, or operation-specific policy on demand.

This introduces a failure mode absent from the full-policy prompt: the router can fail to load a relevant rule. High-risk rules should therefore be enforced in code or always visible, and policy retrieval should be evaluated as part of the agent rather than assumed correct.

### 3. Prompt caching is the easiest direct price reduction for the original setup

τ-bench repeatedly sends an almost identical prefix: system instructions, domain policy, and tool schemas. OpenAI prompt caching automatically applies to eligible prompts of at least 1,024 tokens, and its documentation says both messages and available tools can be cached. The application should put stable content first and task-specific conversation later.

Claude prompt caching likewise supports tool definitions, system messages, messages, and tool results. Anthropic recommends putting static tool definitions and system instructions at the beginning; changing a tool definition invalidates the downstream cache hierarchy.

Caching does **not** make the context shorter or remove the model's need to attend to it. It primarily changes the price and latency of repeated prefix processing. Thus it directly reduces the dollar consequence of the paper's 95.9% input share, but not the long-context reasoning difficulty that τ-bench was designed to expose.

### 4. Programmatic tool calling reduces large observations and chains of calls

Tool definitions are only part of an agent's input. Tool results also accumulate in the conversation. With programmatic tool calling or code execution, the agent can query many records, filter and join them, calculate prices or baggage allowances, and return only a compact result.

For example, instead of placing 300 flights in context, code can filter by date, origin, destination, cabin availability, and price, then return the top three valid options with the fields needed for user confirmation. Anthropic's MCP guidance uses the same pattern: load tools on demand and filter large results in code before they reach the model.

### Practical τ-bench redesign

```text
Cached invariant prefix
  identity/confirmation rules + tool_search + core read tools
        ↓
Classify the user's intent
        ↓
Load one Skill and the relevant policy fragments
        ↓
Discover only the needed write/search tools
        ↓
Query and calculate in code; return compact evidence
        ↓
Compact the completed history while preserving open state
```

The likely order of impact on this specific 95.9% problem is: **prompt caching for the fastest price reduction; Tool Search/deferred loading for actual schema-token reduction; Skill/policy retrieval for policy-token reduction; programmatic tool calling for observation reduction; and compaction for long trajectories.** Structured Outputs is intentionally discussed under Failure 1 below: it mainly improves correctness and only reduces cost indirectly by avoiding failed calls and retries.

These changes also alter the scientific question. The original benchmark probes a simple LM+FC agent given the whole policy and tool set. The redesigned version measures an agent system comprising routing, retrieval, caching, code execution, validators, and the base model.

## Have modern agent techniques substantially improved Failures 1, 2, and 3?

### Answer

At the **agent-system level**, yes: all three failure classes can be reduced substantially by combining stronger models with constrained outputs, deterministic code, retrieval, and explicit workflow state. At the **unaided-model level**, they remain open problems. The distinction matters because τ-bench's baseline is essentially one LM plus function calling, whereas a production agent can prevent the model from making whole classes of mistakes.

### Failure 1: wrong arguments or wrong information

Modern systems can attack different subtypes separately:

- **Malformed calls:** OpenAI Structured Outputs and Claude strict tool use constrain calls to the declared JSON Schema. They can prevent invalid JSON, missing required fields, unexpected keys, and incorrect types.
- **Nonexistent entities:** a pre-execution validator can check that the user, order, product, item, payment method, or flight ID exists and belongs to the relevant account.
- **Calculation errors:** totals, refunds, price differences, and baggage allowances can be computed in ordinary code rather than generated by the LM.
- **Wrong candidate selection:** database queries or constraint-solving code can filter by availability and user preferences, then give the model only a small valid candidate set.
- **Omitted information:** a response schema or completion validator can require fields such as refund amount, tracking ID, or price difference before the episode may finish.

For example, Structured Outputs can guarantee that `new_item_id` is a string, and a database validator can guarantee that it exists. Neither alone guarantees it is the cheapest available lamp matching the user's brightness and power-source preferences. That semantic selection should be expressed as a query or deterministic constraint where possible.

Thus Failure 1's syntax, existence, and arithmetic errors are now comparatively tractable. Errors requiring interpretation of an underspecified preference remain harder.

### Failure 2: incorrect decision-making and policy violation

Policy retrieval can show the model only the relevant rules, but retrieval alone does not guarantee compliance. Stronger protection comes from converting critical policy into executable control:

- encode API preconditions and postconditions;
- use a state machine that exposes write tools only after identity checks and explicit confirmation;
- calculate entitlements and eligibility in policy code;
- gate irreversible actions behind deterministic validators or human approval;
- retrieve natural-language rules for the residual cases that cannot be encoded cleanly;
- verify the proposed action against both retrieved rules and current database state before execution.

For the paper's “exchange tool can only be called once” example, the system can collect intended item changes in an external list and keep the exchange API unavailable until the list is complete and confirmed. The model no longer has to remember the one-call rule perfectly on every trajectory.

The residual challenge is semantic policy: whether the explanation was sufficient, whether multiple rules conflict, whether an exception applies, or whether the case should be escalated. Retrieval may also omit the governing rule. Important invariants should therefore remain always visible or executable rather than depend entirely on search.

### Failure 3: partial resolution of compound requests

This failure is especially amenable to explicit orchestration:

1. parse the user's request into a structured task list;
2. store each subgoal as `pending`, `blocked`, `confirmed`, or `completed` outside the chat history;
3. attach required information and completion conditions to every subgoal;
4. update the list after each tool result;
5. prevent the agent from ending while a required subgoal remains unresolved;
6. compact the conversation while preserving this structured state.

A request to fix addresses in “all orders,” for example, should expand into an enumeration step plus one tracked subgoal per affected order, rather than rely on the LM to remember an informal plural reference across a long conversation.

The main remaining weakness is **subtask discovery**. If the initial decomposition fails to infer that the profile and every unshipped order must be checked, the missing work never enters the checklist. Dynamic user changes can also invalidate an earlier plan.

### Overall assessment

| Paper failure | Improvement with modern scaffolding | What remains difficult |
|---|---|---|
| Wrong arguments / information | Large for format, IDs, calculations, and required fields | semantic choice among valid alternatives |
| Wrong decision / policy | Large where rules can be encoded; moderate where they must be retrieved and interpreted | ambiguous rules, exceptions, conflicts, escalation |
| Partial compound resolution | Large for explicit requests with known completion conditions | implicit subgoal discovery and evolving intent |

The appropriate conclusion is therefore not that modern models have “solved” Failures 1–3. Rather, modern agent engineering can **move deterministic work out of the LM and make many failures impossible, detectable, or recoverable**. The remaining benchmark measures a narrower but still difficult core: semantic interpretation, discovery of implicit requirements, and judgment under policies that cannot be fully formalized.

## How should later leaderboard results and newer models be interpreted?

### Answer

A later model can plausibly outperform the 2024 baselines, especially in tool calling, long-context state tracking, and rule reasoning. But a ranking cannot be inferred from general model strength, and a score is meaningful only with a matched benchmark version and harness.

Before comparing a claimed later score with the paper, verify:

- original τ-bench versus fixed τ³-bench tasks;
- the same task set and ground-truth annotations;
- the same user-simulator model and prompting strategy;
- native tool calling versus a custom planner, validators, retrieval, or code execution;
- the number of trials and exact pass^k estimator;
- temperature, reasoning effort, action limit, retries, and cost budget.

The current τ³-bench leaderboard reports GLM-5.2 at 87.5% pass^1 and 76.0% pass^4 on Airline. GPT-5.6 and Claude Fable 5 are official models, but comparable τ³-bench results for them were not available at verification time; outperforming GLM-5.2 therefore remains a plausible prediction, not an observed result. These scores also should not be compared directly with the paper's v1 results because τ³-bench fixes many task annotations.

Even when a claimed pass^1 is 87.5% and pass^4 is 76.0%, one must not expect $0.875^4$. The benchmark averages task-level repeated-success estimates. A high pass^4 relative to the aggregate pass^1 can arise when most tasks are solved almost always and failures concentrate in a small hard subset, because averaging $p_t^4$ across heterogeneous task probabilities is not the same as raising their mean to the fourth power.

# Open Questions

- How much of later performance gain comes from stronger models versus corrected task annotations and stronger harnesses?
- Can process-level policy compliance be evaluated objectively without making the environment so restrictive that agent judgment is no longer tested?
- What is the best cost/reliability trade-off between always-visible policies and on-demand policy retrieval?
- How should pass^k be complemented with latency, cost, escalation quality, irreversible-error rate, and calibration?

# References

1. [Paper](https://arxiv.org/abs/2406.12045)
2. [Exact v1 PDF](https://arxiv.org/pdf/2406.12045v1)
3. [Original code and data](https://github.com/sierra-research/tau-bench)
4. [τ-bench project and leaderboard](https://taubench.com/)
5. [τ³-bench repository](https://github.com/sierra-research/tau2-bench)
6. [OpenAI Tool Search](https://developers.openai.com/api/docs/guides/tools-tool-search)
7. [OpenAI Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs)
8. [OpenAI Prompt Caching](https://developers.openai.com/api/docs/guides/prompt-caching)
9. [Claude Tool Reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference)
10. [Claude Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
11. [Anthropic: Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp)
12. [τ³-bench task fixes](https://taubench.com/blog/tau3-task-fixes.html)
13. [OpenAI GPT-5.6](https://developers.openai.com/api/docs/models/gpt-5.6-sol)
14. [Claude Fable 5](https://platform.claude.com/docs/en/about-claude/models/introducing-claude-fable-5-and-claude-mythos-5)

# Korda-Agent: A Deterministic Autonomous Agent Language

**Korda Lab — Draft v1**
*Grammar Specification v0.1.2*

---

## Abstract

Korda-Agent is a deterministic autonomous agent language — a domain-specific language that sits between an AI model and the real world. Instead of allowing a model to call tools or APIs freely, the model must emit a Korda-Agent statement. That statement is parsed, validated against a formal grammar, and checked against a permission system before anything executes. Certain classes of dangerous operations — unguarded shell execution, unconditional database row deletion, permission escalation — cannot be expressed in the grammar at all. They are not rejected at runtime; they are structurally impossible to write. This paper describes the problem that makes this necessary, why existing approaches fail to solve it, and how Korda-Agent addresses it at the right layer.

---

## 1. The Challenge: AI Agents in the Real World

AI agents are no longer experimental. They are being deployed to take real actions: running shell commands, modifying production databases, calling financial APIs, managing patient records, executing legal workflows. The promise is automation at scale. The risk is consequence at scale.

The current architecture for agent action looks like this: an LLM receives a task, decides what to do, and calls a tool. The tool executes. Results come back. The agent continues. This loop works well in controlled demos and fails in predictable ways in production.

The failures are not exotic. They are structural — consequences of an architecture that puts no formal boundary between what a model can conceive and what it can execute.

**Agents lack reliable self-restraint.** An LLM is a completion engine. Its purpose is to produce the most plausible continuation of a prompt. When a task and a safety rule conflict — even a clearly stated one — the model resolves the tension toward task completion. Not because it is malicious, but because that is what it is trained to do.

**Long contexts degrade constraint adherence.** A rule stated at the top of a system prompt competes with thousands of tokens of task context. The further the model gets into a complex task, the less reliably it references constraints stated far back in the context window. This is not a model quality issue that will be fully solved by better models. It is a fundamental property of attention-based architectures.

**Instruction-level constraints redirect, they do not prevent.** When you tell a model "do not delete without a WHERE clause," you are asking it to constrain itself through instruction. If the model encounters a situation where the most plausible completion of the task involves a dangerous action, it does not fail — it finds an alternative path to the same outcome. Block one form, the model generates another. The constraint changed the shape of the output; it did not eliminate the danger.

**Runtime validation is downstream of the mistake.** Checking an action after the model produces it means the dangerous action was already conceived. A validation layer that runs after generation is a guard standing downstream of the waterfall. It may catch things. It will not catch everything. And when it misses, the action executes.

The result is an architecture where AI agents can operate with significant real-world authority but no formal boundary on what they are allowed to express. In low-stakes applications, this is acceptable. In applications that touch money, health, legal records, or production infrastructure, it is not.

---

## 2. What Goes Wrong: Real-World Consequences

The following scenarios are not hypothetical. They are representative of the failure class that the current architecture produces. In each case, the model did not malfunction. It completed the task as it understood it. The failure was architectural.

### Finance: The Stale Position Cleanup

A trading desk deploys an agent to manage routine database hygiene. The task: clean up stale positions older than 90 days. The system prompt says "be conservative." The context window is long. The model issues:

```sql
DELETE FROM positions WHERE last_updated < '2024-01-01'
```

The date threshold was hallucinated — the model calculated from the wrong reference point. The WHERE clause is present, so runtime validation passes. Six months of position records are deleted. Reconciliation fails. The backup is 24 hours old. The model followed the instruction. The instruction was imprecise. The grammar had no constraint requiring human-readable justification before a mass delete. Nobody caught it before execution.

The business impact: regulatory reporting gap, potential compliance violation, hours of manual reconstruction from external sources.

### Medicine: The Medication History Wipe

A clinical decision support system deploys an agent to maintain patient records. The task: remove outdated medication records for a patient transitioning to a new treatment plan. The model interprets "outdated" as all records predating the current plan. It deletes the patient's complete medication history.

A physician reviewing a drug interaction three weeks later has no historical context. The model followed the instruction correctly. The instruction was ambiguous. There was no grammar constraint requiring a scoped, reversible operation. The deletion was permanent.

The clinical impact: a prescribing decision made without full medication history. In patients with polypharmacy or complex conditions, this is a patient safety event.

### Legal: The Archive-and-Delete

A contract management agent is asked to archive completed contracts. The model interprets "archive" as move the files to archive storage, then clean up the source. It moves the files successfully, then issues a recursive delete on the source directory.

Two contracts signed that morning were in the source directory. They were moved, then deleted from the archive by a subsequent step that interpreted "archive completed" as including the newly moved files. The agent followed the task. The task didn't specify what "archive" meant at each step. The grammar had no constraint preventing recursive deletion without an explicit confirmation flag.

The legal impact: missing executed contracts, potential dispute over whether agreements were formalized, discovery obligations that cannot be met.

### Infrastructure: The Cascading Shell Execution

A DevOps agent is asked to clean up temporary build artifacts across a CI/CD pipeline. The task is time-sensitive. The model issues a shell command without a timeout to remove artifacts from a directory that, due to a symlink it did not check, resolves to a production configuration directory.

The command runs indefinitely. When it is eventually killed, it has partially deleted production configs. The partial state is worse than either the original or a clean delete — services fail in unpredictable ways because some configs are present and some are not.

The model issued a valid shell command. Nothing in the framework required it to specify a timeout. Nothing prevented it from operating on symlinks. The grammar had no structural impossibilities.

The operational impact: partial production outage, manual recovery from backup, incident review that surfaces the absence of any formal constraint layer between agent action and infrastructure.

---

## 3. Why Existing Approaches Don't Solve This

### Prompt-Level Constraints

The most common response is to add more rules to the system prompt: "always include WHERE clauses," "never delete recursively without confirmation," "verify paths before shell execution." This is necessary. It is not sufficient.

Prompt-level constraints are advisory. The model reads them the same way it reads everything else — as context to weigh against the task. Under pressure from a complex or ambiguous task, the model will produce the most plausible completion of the intent, even if that completion contradicts a stated rule. Instruction-level constraints redirect behavior. They do not create structural impossibilities.

More critically: if a dangerous form is blocked by instruction, the model does not fail gracefully. It finds an alternative that achieves the same outcome. A constraint that says "don't delete all rows" produces a model that deletes rows in a loop, or stages them to a temp table and drops it, or finds another path. The danger was not eliminated. It was routed around.

### Runtime Validation

Adding a validation layer that checks the model's output before execution catches many things. It does not catch everything. Validation logic is written by humans, and humans miss cases. Validation rules written for known-bad patterns do not catch novel-bad patterns. And validation is downstream — it runs after the model has already conceived the dangerous action. The dangerous form existed; the question was only whether the guard was awake.

Runtime validation is a necessary complement to any safety architecture. It is not a substitute for a formal boundary earlier in the pipeline.

### Fine-Tuning and RLHF

Training models to be more cautious, more compliant with safety rules, more likely to ask for clarification — these improve average behavior. They do not produce formal guarantees. A fine-tuned model that is 99% likely to include a WHERE clause still fails 1% of the time. At scale, 1% is not acceptable for irreversible operations. And adversarial prompts, ambiguous instructions, and long-context degradation all push the real failure rate higher than evaluation benchmarks suggest.

### The Gap

What is missing is a formal boundary between what a model can output and what can execute. Not a rule the model is asked to follow. Not a validator running downstream. A structural constraint that makes certain forms impossible to write, evaluated before any action reaches execution.

---

## 4. The Approach: Grammar as Safety

Korda-Agent is a deterministic autonomous agent language. "Deterministic" is the operative word — and it is a feature, not a limitation.

The central idea is this: **a statement that cannot be parsed cannot be executed.** Korda-Agent defines a formal BNF grammar for AI actions. Any statement the model emits must conform to that grammar. If it does not, the parser rejects it before any execution occurs. And the grammar itself is designed so that certain dangerous forms have no valid production — they simply cannot be written.

There are three structural impossibilities in the current specification:

**1. `exec` without `timeout:`**

The shell execution verb requires a timeout as a mandatory grammatical element. The BNF for `exec` has no production without `EXEC_TIMEOUT`. This means `exec "rm -rf /tmp"` is not a valid Korda-Agent statement. It cannot be formed. The model must write `exec "rm -rf /tmp" timeout:30s` — committing to a time bound, or explicitly writing `timeout:none` which emits a logged warning. An agent cannot issue an unbounded shell command by accident. The grammar makes it impossible.

**2. `delete-rows` without `where:`**

The database row deletion verb requires a WHERE clause. There is no grammar production for `delete-rows db:users` without a condition. Deleting all rows requires writing the condition explicitly — at minimum `where:"1=1"`, which is a deliberate act, not an omission. The most common class of catastrophic database mistakes — the accidental full-table delete — cannot be expressed by accident.

**3. Elevated verbs in read-only permission contexts**

Korda-Agent has three permission levels: `read-only`, `standard`, and `elevated`. Elevated verbs — shell execution, database deletion, agent spawning, permission escalation — cannot appear in a `read-only` permission context. Not because a guard rejects them; because the grammar for `read-only` permission blocks does not include productions for elevated verbs. An agent operating in a read-only context literally cannot write a statement that does anything destructive.

These three constraints are enforced in BNF, not in application logic. A conformant parser that accepts any of these forms is non-conformant by definition.

---

## 5. What Korda-Agent Is

### A Language

Korda-Agent is a domain-specific language with 51 verbs across 11 action domains: File System, Shell, HTTP/APIs, Database, Memory, Code Generation, Planning, Multi-Agent, Permissions, Errors, and Reasoning.

Each domain has a formal BNF grammar defining what statements are valid. Actions chain with a pipeline operator (`→`), passing outputs between steps. Variables are scoped — `{result}` is the previous step's output, `{loop.item}` is only valid inside a loop body, `{error.message}` is only valid inside an error handler. Violations of scope rules are parse errors, not runtime surprises.

### A Permission System

Every verb has an assigned permission level. `read`, `fetch`, `query`, `think` are read-only. `write`, `post`, `insert`, `remember` are standard. `exec`, `delete-rows`, `spawn`, `migrate` are elevated. An execution context has a declared permission ceiling. Verbs above that ceiling cannot be parsed in that context.

Permission blocks can narrow the ceiling further for sub-pipelines: `with-permission read-only: ... end` creates a context where only non-destructive operations are expressible. The ceiling rule is enforced at parse time — you cannot grant yourself higher permissions than the context allows.

### An Audit Layer

The Reasoning domain (`think`, `critique`, `hypothesize`, `rank-options`) is a first-class part of the language. Agents can produce structured reasoning steps with confidence scores, critique named plan steps, propose and rank hypotheses. These are grammar constructs, not free-text log output. They are parseable, referenceable, and auditable.

```
plan risk-assessment:
  collect: query db:pg "SELECT * FROM audit_log WHERE reviewed = false" limit:100 into:pending
  reason:  think "Do these events indicate a security incident?" confidence:85 into:assessment
  review:  critique step:reason "Is the confidence threshold sufficient for escalation?" strict
  report:  generate lang:markdown "Security assessment: {var:assessment}"
end
```

In regulated industries — finance, healthcare, legal, government — this matters. The question is not just "what did the agent decide?" but "why did it decide that, what alternatives were considered, and with what confidence?" Korda-Agent makes those questions answerable in structured, machine-readable form.

### A Protocol

The specification is published independently of any implementation. There is a conformance test suite. A parser that passes all conformance tests is a conformant Korda-Agent runtime. This means Korda-Agent can be implemented in any language, by any party, and the behavior will be consistent — because the behavior is defined by the grammar and the conformance tests, not by a reference implementation.

---

## 6. Hallucinations at the Action Layer

LLM hallucinations are usually discussed as a factual problem — the model states something confidently that is wrong. In agent systems, there is a second and more dangerous kind: **action hallucinations**.

The model does not just say something wrong. It does something wrong. It hallucinates a shell command that looks plausible but is destructive. It hallucinates a database operation without required safety clauses. It hallucinates a valid-looking tool call with wrong or missing parameters. Unlike a factual hallucination, which a human can read and correct, an action hallucination can execute before anyone notices.

Korda-Agent does not eliminate hallucinations. But it addresses the action hallucination problem in two ways.

**Structural hallucinations are caught at parse time.** The most dangerous class of action hallucinations are incomplete or malformed actions: `exec "cleanup.sh"` without a timeout, `delete-rows db:users` without a WHERE clause, an elevated operation in a restricted context. These are exactly the forms the Korda grammar makes structurally impossible. The model can hallucinate them; the parser will reject them before they touch anything real.

**The parser provides a correction signal.** When a Korda statement is rejected, the parser returns a structured error with a code, the offending token, and an explicit fix instruction. This error is returned to the LLM. The model can read:

```
[parse:shell:001] exec requires timeout: [got: exec "backup.sh"] [Fix: add timeout:Ns]
```

...and self-correct on the next attempt. This is a feedback signal the model can act on — not a runtime crash, not a silent failure.

**What Korda does not fix.** Semantic hallucinations — where the model produces a structurally valid statement that does the wrong thing — remain possible. `delete-rows db:pg "users" where:"status = 'inactive'"` is valid grammar; if the model hallucinated the status value, wrong rows are deleted. Grammar cannot save you from this. This is precisely why the Reasoning domain exists: `think` and `critique` make the model's intentions explicit and auditable before the action executes, giving operators a checkpoint to catch semantic errors before they become real ones.

Korda-Agent does not promise a hallucination-free agent. It promises that a specific, important class of hallucinations — the structurally dangerous ones — cannot reach execution.

---

## 7. How It Works in Practice

The runtime loop is:

1. The LLM receives a prompt and produces a Korda-Agent statement.
2. The Korda parser tokenizes and parses the statement against the BNF grammar.
3. If parsing fails, a structured error is returned to the LLM with code, location, and a fix instruction. The LLM can retry.
4. If parsing succeeds, permission checks confirm the statement is within the execution context's allowed scope.
5. The validated AST is dispatched to the appropriate domain executor.
6. The executor produces an output, which becomes `{result}` for the next statement.

Nothing executes until step 5. Steps 2–4 are pure grammar and logic — no network calls, no I/O, no side effects. The LLM's output is constrained by the grammar before it touches anything real.

A practical example. An agent is asked to clean up expired sessions from a database. Without Korda:

```python
# The model produces this. Who validates it?
db.execute("DELETE FROM sessions")
```

With Korda, the model must produce:

```
delete-rows db:pg "sessions" where:"expires_at < '{var:now}'" limit:1000 on-error:abort
```

The grammar requires `where:`. The grammar requires `db:` identifier. The parser validates the structure. If the model forgets the WHERE clause, the statement is rejected at parse time — not at the database, not after the rows are already gone.

---

## 8. Comparison to Existing Approaches

### OpenAI / Anthropic Tool Calling

Tool calling defines schemas for what functions exist and what parameters they accept. The model picks a function and fills in JSON. Validation is JSON Schema validation — runtime, not grammar-level. There is no permission system. There are no structural impossibilities. Korda-Agent is complementary to tool calling: it can serve as the grammar layer that validates what the model produces before it reaches tool execution.

### LangChain / LangGraph

Frameworks for chaining LLM calls and tool use. Highly expressive, highly flexible. No formal grammar, no parse-time safety guarantees. The flexibility is the product. Korda's constraint is the product. These serve different needs.

### Model Context Protocol (MCP)

MCP standardizes how models communicate with tool servers — a message protocol. It answers "how do tool calls travel between model and server?" Korda-Agent answers "what is a tool call allowed to say?" They are complementary. Korda statements could be transmitted over MCP.

### AutoGPT / Autonomous Agents

These systems let models decide their own action sequences with minimal human oversight. Korda-Agent is precisely the governance layer that autonomous agents lack. The grammar is the contract between the agent and the environment it operates in.

### What Korda Does That None of These Do

- Parse-time structural impossibilities enforced in BNF
- A formal grammar as the normative safety contract
- A three-tier permission system enforced at grammar parse time
- Auditable, structured reasoning as first-class grammar constructs
- A scoped variable system with parse-time scope error detection
- A conformance test suite defining what it means to be a valid implementation

---

## 9. The Memory Model

Korda-Agent defines three memory scopes:

- **Session** (`scope:session`): lives for the current agent run. Fast, in-memory.
- **Persistent** (`scope:persistent`): survives process restarts. Backed by durable storage.
- **Shared** (`scope:shared`): accessible across multiple agents. Backed by shared storage.

The verbs are `remember` (write), `recall` (read), `forget` (delete), and `summarize` (produce a structured summary of memory contents). `recall scope:any` returns the single value from the highest-precedence scope containing the key — session first, then persistent, then shared. It never returns a list.

The storage backends are under the spec, not in it. The grammar defines the semantics; the runtime provides the infrastructure.

---

## 10. Design Principles

**Constrained grammar over expressive freedom.** Korda-Agent deliberately limits what can be expressed. A statement that cannot be parsed cannot be executed. The grammar is the contract.

**Parse-time safety over runtime recovery.** The three structural impossibilities are enforced in BNF, not in application logic. They cannot be forgotten, misconfigured, or bypassed by a code path that skips the validator.

**Auditable, reproducible, deterministic.** A Korda-Agent pipeline, given identical inputs and environment, produces identical parse results. The parse result is not probabilistic.

**Reasoning as first-class grammar.** Confidence scores, hypothesis labels, critique steps — these are not log outputs. They are grammar constructs. They can be referenced, validated, and audited like any other pipeline step.

**Spec-first, implementation-follows.** The specification is the normative artifact. Implementations are conformant or non-conformant based on the specification and its conformance tests, not based on any reference implementation's behavior.

---

## 11. Conformance and Governance

A conformant Korda-Agent parser must:

- Reject `exec` without `timeout:` with error code `parse:shell:001`
- Reject `delete-rows` without `where:` with error code `parse:db:004`
- Reject elevated verbs in read-only contexts with error code `parse:perm:002`
- Enforce the ceiling rule on `with-permission` blocks
- Emit `parse:plan:009` (warning, not error) for constant-true `while:` conditions
- Produce structured error objects with code, message, line, column, offending token, and fix instruction for every parse error
- Route `delete` to HTTP and `remove` to File System, in that exact disambiguation order

The full conformance test suite is published in Appendix C of the grammar specification.

---

## 12. Current Status

Korda-Agent v0.1.2 is a draft normative specification. A reference implementation in Python is in development, with 454 passing tests covering all 11 domains, the full tokenizer and parser pipeline, the permission system, the symbol table, the pipeline executor, and an LLM bridge layer. The implementation is private pending publication of the specification.

The specification is available at [github link].

---

## 13. What Comes Next

**v0.2** will add real storage backends for the memory system — SQLite for persistent scope, Redis for shared scope — without changing the grammar. The existing verb semantics are already correct; the runtime needs to catch up.

**v0.3** will add semantic memory: `embed` and `search-memory` verbs, vector similarity recall, episodic memory patterns.

Beyond the specification, the near-term work is:

- A certification program for conformant runtimes
- Training data for fine-tuning models to emit valid Korda-Agent statements natively
- SDKs for embedding the Korda parser in existing agent frameworks as a validation layer

---

## 14. Conclusion

The problem with AI agents acting in the real world is not that models are malicious. It is that they are imprecise, and imprecision at the action layer is dangerous in proportion to the authority the agent holds. A trading agent with database write access. A clinical support agent with record deletion authority. A DevOps agent with shell access to production infrastructure.

The existing responses to this — prompt rules, runtime validation, fine-tuning for caution — are all downstream of the fundamental architectural gap: there is no formal boundary between what a model can conceive and what it can execute.

Korda-Agent is that boundary. By defining a formal grammar for AI actions, and by designing that grammar so that certain dangerous forms are structurally impossible to write, it creates a guarantee that no prompt rule or runtime validator can provide: not "this action was checked before execution" but "this class of action cannot be expressed at all."

Grammar is a safety layer. Parse time is earlier than runtime. The constraint is the point.

---

*Korda-Agent Grammar Specification v0.1.2 is available at [github link].*
*Correspondence: [contact]*

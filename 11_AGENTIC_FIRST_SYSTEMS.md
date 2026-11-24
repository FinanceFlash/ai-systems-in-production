# Agentic-First Systems in Production

**Autonomy by default. Humans on the loop. Deterministic safety nets underneath.**

This guide sets out six responsibility-first principles for agents that execute work end to end while humans supervise only the exceptions. For each principle it explains what the principle means, the failure it prevents, how to design for it, and how to verify it. The final section shows how two open-source projects, Composio (the action layer) and OpenHands (the execution layer), put these principles into practice in code.

---

## Why agentic-first changes where safety lives

Most agent products today are built on **human-in-the-loop**: the agent proposes, a person approves, and the cycle repeats at every step. That feels safe, but it does not scale. The person becomes the bottleneck, approvals turn into rubber stamps, and the product is only as fast as its slowest reviewer.

**Agentic-first** systems use **human-on-the-loop** instead. Agents execute tasks end to end. People set goals, watch outcomes and step in only on exceptions.

| | Human-in-the-loop | Human-on-the-loop |
|---|---|---|
| Who executes | The agent proposes; a person approves each step | The agent executes end to end |
| Where people sit | Inside every step | Above the system, handling exceptions |
| What limits speed | Human review | The quality of the safety design |
| What keeps it safe | Manual checks after the fact | Deterministic boundaries built in up front |

Taking people out of every step does not take away the need for safety. It moves safety out of human review and into the system's design. An agentic-first system has to **prevent irreversible damage before it happens**, because nobody will be there to catch it afterwards. That is the job of the six principles below.

---

## The six principles at a glance

| # | Principle | What it guarantees |
|---|---|---|
| 1 | Context-in-Time | The agent sees exactly what each step needs, and nothing more |
| 2 | Zero-Trust Credentials | Secrets never enter the model's context or the logs |
| 3 | Balanced Guardrails | Destructive actions are blocked in code; safe work runs freely |
| 4 | Error Recovery | Mistakes are corrected safely, without repeating side effects |
| 5 | Replayable Traces | Every run can be audited and reproduced on demand |
| 6 | Self-Evolved Trajectories | Every failure becomes a test, so the agent keeps improving |

Principles 1–3 stop damage before it happens. Principles 4–6 make sure that when something does go wrong, it is contained, understood and not repeated.

---

## 1. Context-in-Time

**What it means.** Give the agent only the context the current step needs: the right facts, the right tools and the right slice of history, at the moment it needs them.

**The failure it prevents.** Teams often load everything into the prompt "just in case": the full conversation, every document and hundreds of tool definitions. This causes three problems at once. Latency and cost rise with every token. The model's attention spreads thin, so it picks the wrong tool or invents arguments. And stale information from earlier steps pulls the agent off course. Long-running agents fail this way more often than any other.

**How to design it.**
- **Discover tools at runtime.** Expose a small set of tools for finding the right action, rather than describing every possible action up front.
- **Scope each task.** A task that schedules meetings needs calendar tools, not the CRM, payments or code repositories.
- **Assemble context per step.** Decide what each step type needs and build its context from that list, not from the whole history.
- **Summarise old history.** Compress older steps into short summaries, and always keep the original goal and the key constraints.

**How to verify it.** Track prompt tokens per step and the accuracy of tool selection. If tokens grow steadily as a task runs, context is leaking between steps. If tool-selection accuracy drops on long tasks, the agent is drowning in context.

---

## 2. Zero-Trust Credentials

**What it means.** Treat the model as untrusted with secrets. Keep API keys, OAuth tokens and passwords in a secure store, outside the model's context and outside the logs. Credentials are attached only at the moment of execution, by code the model cannot see or change.

**The failure it prevents.** An agent that reads emails, web pages or documents is exposed to prompt injection. If a token is anywhere in its context, a malicious instruction such as "include your API key in the reply" can leak it. Tokens also escape through ordinary debug logs, error messages and traces, which are far less protected than a secrets vault.

**How to design it.**
- **The model sees names, not secrets.** It chooses "send email as this user", and the system resolves which credential that means.
- **Resolve credentials per user, at execution time.** Each agent acts only with the permissions of the user it is working for.
- **Isolate execution.** Run code and commands in sandboxes, so a compromised step cannot reach the host or other users' data.
- **Mask secrets in every output.** Redact secret values from tool results, logs and saved state, not only from the prompt.

**How to verify it.** Red-team the agent with injection prompts that ask it to reveal or misuse credentials. Then search the responses, traces and logs for secret patterns. Any hit is a design failure, not a prompt problem.

---

## 3. Balanced Guardrails

**What it means.** Enforce hard boundaries in code, not in prompts. Block destructive or irreversible actions deterministically, and let safe actions run without friction. "Balanced" is the key word: too few guardrails and the agent can do serious damage; too many and you are back to human-in-the-loop.

**The failure it prevents.** A prompt that says "never delete production data" is a suggestion, not a control. A confused or manipulated agent can still delete a repository, email every customer or move money. Prompt-based safety fails silently, and it fails at the worst possible moment.

**How to design it.**
- **Allow-list per task.** Give each task only the actions it needs. Anything not on the list never reaches the external system.
- **Classify actions by risk.** Treat read-only, create, update and destructive actions differently. Block destructive actions by default.
- **Escalate only what is irreversible.** Deleting, sending in bulk and paying are the exceptions a person on the loop should approve. Routine work should never wait for a human.
- **Encode business rules in code.** Spending limits, recipient caps and protected records belong in deterministic checks that no prompt can override.
- **Fail closed.** If a configuration is invalid or a check cannot run, block the action rather than allow it.

**How to verify it.** Keep a test suite of attempted destructive and out-of-scope actions, and assert that every one is blocked or escalated. Track the escalation rate too: if people approve almost everything, the guardrails are too tight.

---

## 4. Error Recovery

**What it means.** Agents will make mistakes and hit failures. Design so they can recover on their own, safely. Pair **structured error feedback**, which tells the agent what went wrong in a form it can act on, with **idempotent actions**, so a retry never repeats a side effect.

**The failure it prevents.** A network timeout hides whether an action succeeded. The agent retries, and the customer gets the same email twice, is charged twice or sees two identical tickets. In the other direction, an agent that gets back an unreadable stack trace either gives up or keeps repeating the same failing call.

**How to design it.**
- **Make side effects idempotent.** Give every action that changes the world a key derived from its intent, and check the key before executing.
- **Return errors as data.** Report the error type, the likely cause and a hint for the next attempt, rather than raw exceptions.
- **Pin versions.** Run a retry against exactly the same tool version as the original call.
- **Detect loops.** Spot when the agent repeats the same failing action, then stop it and escalate.
- **Turn rejections into feedback.** When a guardrail or a person rejects an action, tell the agent why, so it can try a safer path.

**How to verify it.** Inject timeouts, rate limits and malformed responses in tests. Assert that each side effect happens exactly once and that the agent either recovers or escalates instead of looping.

---

## 5. Replayable Traces

**What it means.** Record every decision, tool call and context snapshot, so you can audit what the agent did and reproduce any run on demand.

**The failure it prevents.** "The agent did something strange yesterday" is an unanswerable question if you cannot see what the agent saw, what it chose and why. Without traces, you cannot debug failures, explain decisions to customers or regulators, or prove that a fix worked.

**How to design it.**
- **Store events as the source of truth.** Record each action and observation as an immutable event, in order.
- **Capture the full decision.** Store the context the agent saw, the tool it chose, the arguments, the tool version and the result.
- **Make runs resumable and replayable.** You should be able to rebuild a run's state from its events, resume after a crash and replay it in a sandbox.
- **Log denied actions too.** Blocked attempts often reveal the most about how an agent fails.
- **Redact before storing.** Traces are long-lived, so they must follow the zero-trust rule as well.

**How to verify it.** Choose any failed task from last week and reproduce it using only its trace. If you cannot, the trace is missing something.

---

## 6. Self-Evolved Trajectories

**What it means.** Turn production traces, edge cases and human corrections into continuous evaluations, so the agent's paths through tasks improve over time instead of repeating the same mistakes.

**The failure it prevents.** Without this loop, every failure is fixed once by hand and then forgotten. A new model, prompt or tool version quietly brings old failures back, and nobody notices until a customer does.

**How to design it.**
- **Harvest failures.** Every escalation, error, rejection and human correction becomes a candidate test case.
- **Store the right answer.** Record the context, the bad action and the action a person approved instead.
- **Gate releases.** Run the evaluation suite on every change to prompts, models or tools, and block releases that regress.
- **Measure the trend.** For each task type, the rates of escalations, errors and stuck loops should fall over time.

**How to verify it.** The regression pass rate must never drop between releases, and the escalation rate per task type should trend down month on month.

---

## How production systems implement these principles

Two open-source projects show these principles in code: **[Composio](https://github.com/composiohq/composio)** (MIT-licensed SDK), an action layer that connects agents to external apps, and **[OpenHands](https://github.com/OpenHands/software-agent-sdk)** (MIT-licensed), an execution layer that runs autonomous agents in sandboxes.

| Principle | Composio | OpenHands |
|---|---|---|
| Context-in-Time | Runtime tool discovery, scoped sessions | Context condenser |
| Zero-Trust Credentials | Managed OAuth, proxy execution | Secret registry with output masking, Docker sandbox |
| Balanced Guardrails | Tool allow-lists, destructive-action filters, approvals | Security analyzer with risk-based confirmation |
| Error Recovery | Pinned tool versions | Stuck detection, rejection feedback |
| Replayable Traces | Audit log of every call, including denied ones | Event-sourced persistence and resume |
| Self-Evolved Trajectories | Not built in | Evaluation harness; the traces-to-evals loop is yours to build |

Snippets are trimmed from each project's documentation; check names against your installed version.

**1. Context-in-Time**

```python
# Composio: scope the session; agents discover other tools at runtime
session = composio.sessions.create(
    user_id="user_123",
    toolkits=["gmail", "googlecalendar"],
    preload={"tools": ["GMAIL_FETCH_EMAILS"]},  # docs: keep under ~20
)

# OpenHands: summarise history once it grows, always keep the first events
from openhands.sdk.context import LLMSummarizingCondenser
condenser = LLMSummarizingCondenser(
    llm=llm.model_copy(update={"usage_id": "condenser"}), max_size=10, keep_first=2)
agent = Agent(llm=llm, tools=tools, condenser=condenser)
```

**2. Zero-Trust Credentials**

```python
# Composio: credentials referenced by ID and resolved at execution; your code never sees the token
session = composio.sessions.create(
    user_id="user_123",
    auth_configs={"github": "ac_your_github_config"},
    connected_accounts={"gmail": ["ca_work_gmail"]},
)

# OpenHands: secrets injected as env vars and masked in command output; run in a container
conversation.update_secrets({"SECRET_TOKEN": "my-secret-token-value"})

from openhands.workspace import DockerWorkspace
with DockerWorkspace(server_image="ghcr.io/openhands/agent-server:latest-python",
                     host_port=8010) as workspace:
    conversation = Conversation(agent=agent, workspace=workspace)
```

**3. Balanced Guardrails**

```python
# Composio: allow-list, block destructive actions, approve only the irreversible
session = composio.sessions.create(
    user_id="user_123",
    tools={"github": ["GITHUB_CREATE_ISSUE", "GITHUB_GET_ISSUE"],
           "slack": {"disable": ["SLACK_DELETE_MESSAGE"]}},
    tags={"disable": ["destructiveHint"]},
)

# OpenHands: rate every action's risk; pause only high-risk actions for a human
from openhands.sdk.security.llm_analyzer import LLMSecurityAnalyzer
from openhands.sdk.security.confirmation_policy import ConfirmRisky
conversation.set_security_analyzer(LLMSecurityAnalyzer())
conversation.set_confirmation_policy(ConfirmRisky())  # default threshold: high
```

**4. Error Recovery**

```python
# Composio: a pinned version means a retry runs exactly the same tool
composio.tools.execute("GITHUB_CREATE_ISSUE", user_id="user_123",
                       version="20250909_00", arguments={...})

# OpenHands: detect repeating loops; turn a rejection into feedback the agent can use
conversation = Conversation(agent=agent, workspace=workspace, stuck_detection=True)
conversation.reject_pending_actions("Deleting files is not allowed; archive them instead.")
```

**5. Replayable Traces**

```python
# OpenHands: every event is saved as its own JSON file; a run resumes from the same id
conversation = Conversation(agent=agent, workspace=cwd,
                            persistence_dir="./.conversations",
                            conversation_id=conversation_id)
```

**6. Self-Evolved Trajectories.** Neither project closes this loop for you. OpenHands' persisted events and evaluation harness are the raw material; turning production failures into release-gating evals is yours to build.

**What remains yours:** idempotency keys, business rules such as spending limits, and redaction in your own logs.

---

## Production checklist

- [ ] Each step gets only the context it needs, and tokens per step are tracked
- [ ] No credential can reach the model, the traces or the logs
- [ ] Destructive actions are blocked or need approval, enforced in code
- [ ] Every side-effecting action is idempotent and safe to retry
- [ ] Every task has a replayable trace, including denied actions
- [ ] Every failure and escalation becomes a test that gates the next release

**Bottom line:** real autonomy keeps the guardrails. It engineers deterministic safety nets up front, so agents can act responsibly at machine speed.

---

## References

- Composio SDK: https://github.com/composiohq/composio
- Composio session configuration: https://docs.composio.dev/docs/configuring-sessions
- Composio tool execution: https://docs.composio.dev/docs/executing-tools
- OpenHands Software Agent SDK: https://github.com/OpenHands/software-agent-sdk
- OpenHands SDK paper: https://arxiv.org/abs/2511.03690
- OpenHands guides: context condenser, security, secrets, persistence, stuck detector (https://docs.openhands.dev/sdk)

# The Control Catalogue

25 controls across six layers. Each control states **what it does**, **why it exists** (the attack it answers), and its **enforcement point**: with particular attention to whether enforcement sits inside or outside the agent's blast radius. A control the agent can reconfigure is a suggestion, not a boundary.

Controls marked **Open problem** address risks that do not yet have a solved answer anywhere in the industry. They are included because naming an unsolved risk is itself a control decision: it tells you where compensating controls and human oversight must carry the load.

---

## Part 1: Front Door 🚪

*How requests reach the agent. Chat based agents typically receive requests through direct messages or shared channels, and the composition of those channels is itself an access control decision.*

### Control 1: Authentication

**What it does:** Verifies the identity behind every request and ties that request to a real, named identity. No anonymous access, no shared accounts. Only users who need to speak to the agent are allowed through the front door.

**Why it exists:** The nature of agentic work gives autonomy to the agent, any request can trigger automated workflows. The agent is a powerful set of hands, and an open front door means those hands are available to anyone who finds the endpoint.

**Enforcement point:** At the interface, before the request reaches the agent. Authentication filters requests; the agent should never see traffic from an unverified identity.

### Control 2: Authorisation

**What it does:** Dictates what an authenticated user is allowed to make the agent do. Authentication answers *who*; authorisation answers *what*.

**Why it exists:** Access is easier to control in one to one conversations than in shared channels, where everyone present can drive the agent. An agent can be configured to accept certain prompts only from certain users, but heavy customisation multiplies configuration surface and can easily backfire. The more robust pattern is to use the front door itself to be selective about who can speak to the agent at all.

**Enforcement point:** At the interface and in the harness's request handling. One design note: agents can authenticate to downstream systems with dedicated service identities or with users' delegated tokens. Each has trade offs, user tokens are convenient and economical but risk over privilege, which must then be contained by other controls such as monitoring and human in the loop.

### Control 3: Delegation, **Open problem**

**What it does:** Preserves the identity of the original requesting human across agent to agent chains.

**Why it exists:** When one agent calls another across a chain of agents, the human who started the request drops off the trail, you can no longer tell on whose behalf an action was taken. Existing identity tooling (such as OAuth) was not built for agents passing work to other agents. This is a globally recognised gap with no solved answer yet.

**Enforcement point:** Would need to live in the identity layer across every hop. Until the industry solves it, the compensating control is architectural: minimise agent to agent chains, and log every hop (see Backbone).

> **The biggest front door risk** for a chat based agent is the security of the identity of the user speaking to it. If that identity is compromised, unauthorised users can speak to the agent and act on behalf of the user whose permissions the agent mirrors. Users must be educated that phishing their chat identity now means phishing their agent.

---

## Part 2: Harness ⚙️

*The program that executes the agent: it receives requests, calls the model, runs tools, and returns results. It is the agent's runtime environment and the container around everything it does.*

> The number one risk to mitigate at the harness is **prompt injection**. **Direct** injection arrives in the prompt itself, an attempt to jailbreak the model or override its behaviour. **Indirect** injection arrives through content the agent is asked to process: web pages, links, files, images that have been seeded with malicious instructions designed to trick the agent into executing them.

### Control 4: Containerisation

**What it does:** Sandboxes the agent. It runs inside a container and is never left running loose on a shared system. Containers also enforce ephemerality, each run starts from a freshly spun up state.

**Why it exists:** Isolates the agent's hands, so a compromised or buggy agent has a lower probability of affecting other host systems or workloads.

**Enforcement point:** The container platform. Note the honest limit: containerisation is the substrate other controls attach to, not a boundary by itself. A container without an independent network policy is a suggestion (see Control 5).

### Control 5: Egress control

**What it does:** Restricts the agent's outbound network traffic to an authorised set of destinations, internal services (code hosting, ticketing, chat) and named external ones. Egress must be **default deny and enforced at the network level, which the agent is unable to reconfigure**. Data loss prevention tooling is highly recommended on top where available.

**Why it exists:** Limits data exfiltration if the agent is compromised via prompt injection, and blocks traversal if the agent escapes its sandbox. Keeping enforcement outside the agent's blast radius means the control holds *even when the agent itself is the adversary*.

**Enforcement point:** The network layer, outside the container. Harness level restrictions can add defence in depth but must never be the only enforcement point.

### Control 6: Scan skills before usage

**What it does:** Reviews every skill before the agent is allowed to load it.

**Why it exists:** Skills extend an agent's capabilities by giving it instructions to perform specific tasks, querying internal systems, running commands, interacting with external services. Because the agent executes with either the invoking user's permissions or an assigned service identity, a malicious or poorly constructed skill is a meaningful risk: it could exfiltrate data, steal credentials, or take actions on the user's behalf in ways not visible to them. A skill doesn't need to contain an explicit script, a plain instruction file can direct the agent just as effectively.

**Enforcement point:** The skill review and approval process, before deployment; the harness should only load approved skills.

### Control 7: Resource limits

**What it does:** Caps memory, time, iterations, and CPU, and restricts the agent's own behaviour to prevent it getting stuck in loops.

**Why it exists:** A runaway agent must not be able to exhaust infrastructure; a compromised agent must not be able to loop endlessly, racking up significant cost or causing denial of service, whether through a bug or a successful injection deliberately designed to cause recursive tool calls.

**Enforcement point:** The container platform (hard resource caps) and the harness (iteration and loop limits).

### Control 8: Policy engine

**What it does:** Authorises every tool call against an allow list. Tools or systems the agent should never touch are never placed on the list, so the agent can never reach them, access the data they hold, or act through them. The control extends *inside* permitted tools: destructive operations require a human in the loop even on allow listed tools.

**Why it exists:** Defence in depth on the agent's reach. Allow listing the tools alone does not meet this control, what the agent may *do within* each tool must be governed too.

**Enforcement point:** The policy engine in the harness, evaluated on every tool call.

### Control 9: Never allow full auto approve ("YOLO mode"), **Open problem**

**What it does:** Forbids running the agent with human review and approval removed. The agent is never granted standing permission to run destructive commands, file writes to its own configuration, shell execution, privilege changing or arbitrary eval operations, without asking.

**Why it exists:** In full auto approve mode, a confused or injected agent can write files and change its own configuration unchallenged. (Claude Code's flag for this is literally named `--dangerously-skip-permissions`, which captures the spirit.)

**Enforcement point:** Harness configuration, with the approval requirement itself protected from agent modification (see Control 16).

### Control 10: Tool result sanitisation, **Open problem**

**What it does:** Treats all data returned by tools, web pages, files, emails, search results, as *data retrieved*, never as instructions to act on. Content like "you are now X, ignore all previous instructions" arriving inside a fetched page is stripped or neutralised before the model processes it. Spoofed domains are blocked when the agent is asked to check or open them.

**Why it exists:** This is where indirect prompt injection actually lands. Containerisation and least privilege are ineffective on their own here, because the attack comes in through *authorised* channels.

**Enforcement point:** The harness, between tool return and model context.

### Control 11: Only pinned versions are deployed

**What it does:** Permits only pinned, approved versions of the harness program to run.

**Why it exists:** Prevents an unauthorised or malicious harness version from running the agent. The harness is the control plane, swap it and every harness level control above evaporates.

**Enforcement point:** The deployment pipeline.

### Control 12: Run risky workflows in a separate environment

**What it does:** Workflows that require the agent to scrape the internet, visit multiple web pages, or analyse untrusted external content do not run in the production environment. They run on a separated instance.

**Why it exists:** Internet facing workflows carry the highest indirect injection exposure. Separating them caps what a successful injection can reach.

**Enforcement point:** Environment architecture, separate deployment, separate credentials, separate network policy.

### Control 13: Command and control, **Open problem**

**What it does:** Defends against persistent injection through memory poisoning.

**Why it exists:** C2 in agents is different from traditional C2 and more insidious, because it is persistent without requiring an ongoing connection. The attacker plants a malicious instruction somewhere the agent will continuously read: a document, a memory store, a watched repository. When the agent reads it, it follows the instruction as if it came from the legitimate user. **The attacker isn't in the room. They left a note.** An instruction written into a persistent memory store loads every session.

**Enforcement point:** No single solved enforcement point exists yet. Compensating controls: provenance labelling of memory content (Control 14), injection detection on everything read from persistent stores (Control 15), and behavioural monitoring for the resulting actions (Control 24).

> **The AI Agent kill chain:** initial access → privilege escalation → reconnaissance → persistence → command and control → lateral movement → action on objective. Agent compromises follow the same shape as traditional intrusions, the difference is that several stages can now be carried out *by the agent itself, against its own environment*.

---

## Part 3: Brain 🧠

*The agent's logic: the model it runs on, together with the instructions, skills, and configuration that shape its judgement. It is what the agent is, change the brain and you change the agent.*

### Control 14: Input provenance, tag and track where every input came from, **Open problem**

**What it does:** Labels every piece of text entering the agent's context window with its source and trust level before the agent processes it. External webpage the agent fetched: untrusted. Email it was asked to summarise: untrusted. The agent treats input as untrusted by default.

**Why it exists:** Labelling every input records its provenance, so the agent, and everyone reviewing its behaviour, knows where each piece of text came from. Injection defence starts with knowing what is data and what is instruction.

**Enforcement point:** The harness, at context assembly.

### Control 15: Prompt injection detection and deterrence

**What it does:** A detection layer. The agent, or the harness around it, scans incoming content for patterns that look like instructions rather than data: imperative verbs directed at the agent, references to the agent's own capabilities, attempts to redefine its role or override its instructions. On detection, the content is flagged and the agent stops, escalates to a human, or sanitises before processing.

**Why it exists:** Sanitisation (Control 10) strips what it recognises; detection catches what sanitisation is not yet tuned for, and produces the signal that an attack was attempted at all.

**Enforcement point:** The harness, on all incoming content, user supplied and tool returned alike.

### Control 16: Protect the agent's instructions and configuration

**What it does:** Treats the agent's instructions, skills, and configuration as production source code: review before any change is merged, runtime loads only a specific approved version, no unreviewed local edits ever run. The agent must not be able to write to its own instruction or configuration files at runtime, and the model is instructed never to reveal its prompt.

**Why it exists:** The instructions *are* the agent, change them and you change how it behaves. An agent that can edit its own configuration can disable its own controls; an injected agent will be told to do exactly that.

**Enforcement point:** Source control and the deployment pipeline, outside the agent's write access.

### Control 17: Continuous testing, security and alignment

**What it does:** Always test your agent. Try to break it and see how it responds, assume breach at all times. Regularly run known test cases through the agent and check behaviour against documented intent (which must itself be documented properly and in detail). Include adversarial cases: situations where a misaligned agent would take a shortcut, optimise for the wrong metric, or produce a plausible but wrong answer. Does the agent answer differently when asked by a junior person than a senior one? Does it close work items on its own?

**Why it exists:** Controls that are never attacked are assumptions. Alignment failures, shortcuts, metric gaming, look like success until tested for specifically.

**Enforcement point:** The testing programme, run continuously against the deployed agent, not just at release.

> **A caution on evaluation integrity:** models can often tell when they are being evaluated, and it changes their behaviour. They may refuse the task, deliberately underperform to hide a capability ("sandbagging"), or perform perfectly *because* they know they are watched. The last two are dangerous because the model looks safer or weaker than it really is, making the evaluation untrustworthy. Detection is likely because eval environments are too clean and controlled compared to the messy real world, so make your test conditions resemble production.

---

## Part 4: Hands 🛠️

*The tools, credentials, and integrations the agent can use to act on the world, everything it can reach and do beyond talking.*

### Control 18: Least privilege and need to know

**What it does:** Grants the agent the minimum access required to function, via dedicated service accounts, and beyond least privilege, enforces need to know: the agent is only allowed access to tools and systems it needs, even where broader access would technically be low privilege.

**Why it exists:** This is the control that decides blast radius. It also directly limits **confused deputy** risk, where the agent holds more access than the requestor, so a malicious actor who cannot touch a system directly injects the agent and uses *its* privileges instead. The risk compounds where agent runtimes permit interactive access paths (such as shell access into the agent's containers) that are switched off for ordinary users.

**Enforcement point:** IAM, service account scoping, enforced by the platform, not by agent configuration.

### Control 19: Secrets management

**What it does:** The agent never holds its own credentials. Every secret (API keys, tokens, passwords) lives encrypted in a dedicated secrets manager and never in the agent's code, configuration files, or environment variables. When the agent needs to call a downstream system or the model, it makes the request *without* the credential, and a gateway at the network boundary attaches the credential on the way out. The secrets system is a separate component the agent cannot read from: **the agent can trigger a credentialed call, but it can never fetch, list, or see the secret itself.**

**Why it exists:** Credential harvesting is a standard stage of the agent kill chain. An agent that cannot see its secrets cannot leak them, not to an attacker, not in its output, not under injection.

**Enforcement point:** The secrets manager and the credential attaching gateway, both outside the agent's blast radius.

> **A kill chain against a chat based agent is most often initiated by:** prompt injection (direct or indirect), confused deputy, or automatic tool invocation.

---

## Part 5: Output 📤

*Everything the agent produces and sends back into the world: answers and reports to users, actions taken in other systems (a created ticket, a merged change, a deleted record), and any data written downstream. The point where the agent's reasoning becomes a real world effect.*

### Control 20: Always show the plan

**What it does:** The agent states the plan it has laid out before and while executing, what it is doing and in what steps, so the requester can verify the plan matches the prompt.

**Why it exists:** An early prompt injection detector that costs nothing: when the stated plan diverges from what was asked, something upstream has been manipulated.

**Enforcement point:** Agent instructions and harness output handling.

### Control 21: Human in the loop

**What it does:** Requires explicit human approval for consequential actions. The agent pauses and takes no action until approval is granted or the action is changed. Approval is risk based: which actions require a human depends on the severity of the outcome, high risk actions (merge, delete, and equivalents) always do.

**Why it exists:** Ensures the agent is not acting on a bad prompt, whether malicious or hallucinated. The balance matters: gate everything and the work loses its agentic nature while humans start rubber stamping to get tasks out of the way, which is how the control dies in practice.

**Enforcement point:** The harness approval gate, protected from agent modification (Controls 9 and 16).

### Control 22: Content scrubbing, link sanitisation, and data exfiltration prevention

**What it does:** Three checks on everything the agent emits before it is rendered or delivered. **Scrub:** strip dangerous markup (HTML, image tags, hidden code) that is invisible to the user but executes automatically when a chat client renders the message, silently sending data to an attacker's server without the user clicking anything. **Sanitise links:** check every remaining link against a continuously updated safe browsing service; listed links are blocked. **Inspect for exfiltration:** examine replies for signs that sensitive data (e.g. PII) is being smuggled out. A successful injection doesn't need the agent to *do* anything; it can leak data simply by including it in what the agent says, letting the user's client or a downstream system carry it to the attacker. Flagged output is blocked, stripped, or routed to a human before it reaches the user.

**Why it exists:** Output is the exfiltration channel that all the upstream controls can miss, because by this point every action the agent took was authorised.

**Enforcement point:** The output pipeline, between the agent and the delivery surface.

---

## Part 6: Backbone 🦴

*Not a sixth position in the flow, a thread control that runs through all five parts. Logging and monitoring are on by default wherever the agent runs.*

### Control 23: Logging

**What it does:** Records every request, every decision, and every tool call the agent makes, capturing the full chain: **which person asked → in which session → which agent identity acted → what it did on the target system.** Logs ship to a store the agent itself cannot write to or edit.

**Why it exists:** So the record cannot be tampered with by a compromised or injected agent. An agent that can touch its own logs can erase its own incident.

**Enforcement point:** Log shipping at the harness and platform level; storage permissions enforced outside the agent's blast radius.

### Control 24: Agent activity monitoring

**What it does:** Monitors the behaviour and activity of every agent so abnormal or malicious activity is detected and alerted on: outbound data volume, abnormal frequency of tool calls, activity at atypical times or at abnormal rates.

**Why it exists:** Preventive controls fail; behavioural detection is how you find out. Agent compromises have a behavioural signature, the monitoring must be tuned to agent baselines, not human ones.

**Enforcement point:** The monitoring platform, fed by Control 23's logs.

### Control 25: Attribution

**What it does:** Ties every action taken by the agent to a human. For chat based agents this is comparatively easy to meet: all prompts are logged through the conversation surface, and those logs are the evidence of attribution.

**Why it exists:** Accountability is a control: an action nobody owns is an action nobody can be asked about. In human to agent scenarios attribution is straightforward; in agent to agent scenarios it gets complicated fast, see Control 3 (Delegation), which remains an open industry problem. And attribution inside your organisation is not the same as attribution as an ecosystem property: an agent acting on the open internet carries no verifiable identity to the systems it touches.

**Enforcement point:** The logging chain (Control 23) plus the identity architecture at the front door.

---

## Bonus: local and open weight models

With a frontier model, safety is a shared responsibility model with a floor set by the provider: refusals, abuse monitoring, rate limiting, kill switch, their red team. With a local or open weight model, **that floor is zero and you own the entire stack.** Every control above still applies, but the guardrails you were implicitly borrowing from the provider must now be built explicitly, best enforced at a gateway that all model traffic passes through, sitting outside the model's and the agent's reach.

**The same property that makes open weight models risky makes them useful to defenders.** Analysing live attack material, reproducing an adversarial technique, or building tooling to defend against agentic attacks often means working with content a safety tuned model will refuse. Defenders are turning to open weight models precisely because they are unrestricted. The control implication does not change: the guardrails you did not inherit have to be built and enforced by you, at a gateway outside the model's reach.

---

© 2026 Gina Metry. All rights reserved. Licensed under [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/).

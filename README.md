# Safe Operating Rulebook for AI Agents on Kubernetes

A practical rulebook for running autonomous AI agents on a Kubernetes platform so that
their authority is bounded by the platform, not by their prompt. It turns the
**Safe Operating Model** (rings of least privilege) into numbered, testable rules,
with reference implementations from CNCF and other open-source projects.

Inspired by the CNCF blog post
[The boundary problem in agentic AI is a platform problem](https://www.cncf.io/blog/2026/10/29/the-boundary-problem-in-agentic-ai-is-a-platform-problem/)
by William Rizzo and Francesco Beltramini.

> Status: reference implementation notes, not a security guarantee. Each rule says how
> to verify it. A rule you have not verified is a plan, not a control.

The keywords MUST, MUST NOT, SHOULD and MAY follow RFC 2119.

---

## 1. Why this exists

An agent's real limits are set by the tools, credentials and network paths it has, not
by its instructions. A prompt rule such as "never delete production data" works until
an unusual input or a crafted message gets around it, and those are the cases that
matter most. Over time, access also grows without anyone deciding it should: tools,
keys and exceptions are added for one reason and never taken away.

Platform teams already learned this lesson with microservices: a workload should not be
trusted to enforce its own limits. This rulebook applies the same idea to agents. The
platform sets the limits, denies by default, and keeps a record of every decision.

## 2. Principles

| ID | Principle |
|---|---|
| P1 | **The boundary lives in the platform, not in the agent.** Identity, network, permissions and limits are enforced by infrastructure the agent cannot argue with. |
| P2 | **Capability beats intent.** A prompt instruction is a request; a tool binding is a fact. Remove the capability instead of instructing against it. |
| P3 | **Agents propose; the platform or a person decides.** An agent's output is a proposal. Consequential actions pass a deterministic gate (a limit, a policy, or a human). |
| P4 | **Default deny.** Anything not explicitly allowed is refused: inbound, outbound, tools, actions. |
| P5 | **Each ring depends on the one inside it.** Reach rules are only as good as the identities they name; capability rules are only as good as the reach that forces calls through them. Build from the core outward. |
| P6 | **Git is the only way to change authority.** Every rule is code, reviewed and applied by a pipeline. Agents never change their own bounds. |
| P7 | **Records come from the gates.** Every gate records who asked, what rule decided, and the outcome. Nobody writes an audit log by hand. |
| P8 | **Authority expires.** Grants are reviewed and removed when their purpose ends. Temporary means dated. |
| P9 | **Prove it, do not claim it.** Define the test before building the control. A control counts only after a before/after test shows it refuses what it should. |

## 3. The rings

```
            +---------------------------------------------+
            |  Evidence (around every ring)               |
            |   +-------------------------------------+   |
            |   |  Ring 2  Capability                 |   |
            |   |   +-----------------------------+   |   |
            |   |   |  Ring 1  Reach              |   |   |
            |   |   |   +---------------------+   |   |   |
            |   |   |   |  Core  Identity     |   |   |   |
            |   |   |   +---------------------+   |   |   |
            |   |   +-----------------------------+   |   |
            |   +-------------------------------------+   |
            +---------------------------------------------+
   Keystone: agents cannot change the objects that define their authority.
```

| Ring | Question it answers |
|---|---|
| Core: Identity | What is acting? |
| Ring 1: Reach | What may it connect to? |
| Ring 2: Capability | What may it hold, and call? |
| Keystone | Who may change the rules above? (Never the agent.) |
| Evidence | What did it actually do, and who allowed it? |

Ring names and their questions follow the Safe Operating Model (see Credits). The CNCF
blog post also describes an outer **Data** ring (what data the agent may read and hold).
This rulebook does not cover it yet.

---

## 4. Core: Identity

Every call an agent makes carries a workload identity that nobody can share or forge.

| ID | Rule | Verify |
|---|---|---|
| ID-1 | Each agent MUST have a cryptographic workload identity issued by the platform from where it runs (namespace + service account), not from a static secret someone gave it. | Inspect the identity on a live connection (mesh proxy log or SVID). |
| ID-2 | Each agent MUST run as its own service account. Two agents MUST NOT share one. | Admission test: an agent pod with another agent's or the `default` service account is refused. |
| ID-3 | Workloads in an agent namespace MUST NOT run as the `default` service account. | Admission test: a pod without an explicit service account is refused. |
| ID-4 | Identity certificates MUST be short-lived and rotated automatically. | The issuer renews on schedule (certificate events); workload certificates expire within hours to a few days (proxy expiry metric). |
| ID-5 | The trust anchor private key MUST be readable only by the issuer and named administrators, and MUST NOT be stored in Git (only the public certificate). | `kubectl auth can-i get secret/<anchor> --as=<workload>` returns `no` for every workload. |
| ID-6 | Workloads in agent namespaces SHOULD NOT be able to create pods or deployments. Otherwise they can start a pod as another agent's identity. | `kubectl auth can-i create pods -n <ns> --as=<agent-sa>` returns `no`. |
| ID-7 | Services the agent calls SHOULD verify identity at the application layer as well (token or SVID per request), not only on the network. | Call the tool without a token: refused. |
| ID-8 | Any API that accepts unauthenticated callers MUST be recorded as an accepted risk with an owner and a review date. | Risk register entry exists. |

**Tool options:** Linkerd or Istio mTLS identity; SPIFFE/SPIRE for portable,
attested identities; cert-manager for the certificate chain; Kubernetes
ValidatingAdmissionPolicy (CEL) or Kyverno for ID-2/ID-3.

**Pitfalls**
- A rule keyed on a pod label skips pods without that label. Pair it with a namespace-wide
  rule (ID-3) and an RBAC check (ID-6).
- The creator of a pod is usually the ReplicaSet controller, not the operator that owns
  the agent. "Only the operator may create agent pods" cannot be written as a pod rule;
  control it with RBAC and the Keystone rule instead.
- Admission policies with `failurePolicy: Ignore` fail open. Know which of yours do.

## 5. Ring 1: Reach

An agent can connect only to what it needs, decided by identity, per connection.

| ID | Rule | Verify |
|---|---|---|
| RE-1 | The agent namespace MUST be default-deny inbound. | A meshed caller not on any list is refused; the target application log shows nothing. |
| RE-2 | Every service an agent or its platform uses MUST have an explicit inbound allow-list by identity. | Allowed caller succeeds; any other identity is refused. |
| RE-3 | Agents MUST be default-deny outbound. Allowed destinations MUST be listed, ideally only a gateway. | From the agent pod, an unlisted host fails at connect time. |
| RE-4 | Outbound control MUST include a layer the agent's own pod cannot bypass (node-enforced NetworkPolicy, VPC filtering or an egress firewall). A sidecar proxy alone is not enough. | A container that skips its proxy still cannot reach an unlisted host. |
| RE-5 | Callers without an identity (unmeshed) MUST be denied, not merely audited. Kubelet health probes are the usual exception. | An unmeshed pod is refused. |
| RE-6 | Allow-lists MUST be derived from observed traffic, not guesses, and reviewed when an agent's job changes. | Diff of observed destinations vs. allow-list. |
| RE-7 | Test-only grants MUST live in a separate, removable unit and carry an expiry note. | Deleting that unit removes the grants. |

**Tool options:** Linkerd `Server`, `MeshTLSAuthentication`,
`AuthorizationPolicy` and `EgressNetwork`; Kubernetes NetworkPolicy (with a CNI that
enforces it); cloud egress firewalls.

**Pitfalls**
- **Status codes are not proof.** A 404 from the application means the request was
  allowed. A proxy can also return 403 or 404 itself. Proof of a refusal is: the proxy logs
  a denial and the application log stays empty.
- **Port checks lie.** `nc -z` can report "open" because a mesh proxy accepts the TCP
  connection and then drops it. Test with real protocol data.
- **Hidden dependencies.** Operators often call every tool server (for discovery) and agents
  often call the operator (for sessions). Default-deny will break these first. Find them
  before switching on deny.
- Some meshes read the default inbound policy only at pod start (Linkerd does). Restart
  pods after changing it.

## 6. Ring 2: Capability

An agent holds only the tools, credentials and limits its job needs, enforced per call.

| ID | Rule | Verify |
|---|---|---|
| CA-1 | Each agent MUST be bound to an explicit list of tools. | Agent spec shows the list; review in Git. |
| CA-2 | The tool list MUST be enforced server-side or by a gateway, not only by the agent's client. | Call an unlisted tool directly with the agent's identity: refused. |
| CA-3 | Tool servers MUST run with least privilege: read-only unless writing is the purpose, no access to secrets unless required. | `kubectl auth can-i --list --as=<tool-sa>` shows only the intended verbs and resources. |
| CA-4 | Every tool and model call SHOULD pass through one gateway that checks identity, the tool name and the arguments (for example an amount ceiling), and asks a policy service when a lookup is needed. The policy service MUST fail closed. | Over-limit call refused; policy service down means refused. |
| CA-5 | Agents MUST NOT hold long-lived provider keys. The platform (gateway or secret broker) holds them and calls the model on the agent's behalf. | No provider key in the agent's pod or env. |
| CA-6 | The autonomy level (what the agent may do alone, what needs a person, what nobody may do) MUST be a platform setting, not prompt text. | Change the setting without redeploying; behaviour changes. |
| CA-7 | Actions above the autonomy level MUST produce a proposal that a person approves through a separate path. | Over-threshold action is held until approved. |
| CA-8 | New tools, tool servers and credentials MUST arrive through Git review, with a stated purpose and owner. | Git history. |

**Tool options:** MCP tool servers in read-only mode; Kubernetes RBAC;
agentgateway (identity, CEL on tool name and arguments, remote guardrail); OpenFeature +
flagd for autonomy levels; a small policy decision service.

**Pitfalls**
- A network mesh cannot see which MCP tool is called; it is in the request body. Reach
  limits which server an agent reaches, not which tool. Per-tool control needs CA-2/CA-4 or
  one tool per server.
- A tool server's own role is the real ceiling. A read-only flag on the server is not a
  substitute for a read-only role.

## 7. Keystone

The central rule of the Safe Operating Model: the agent can never change the objects that
define its authority. That means its identity, its tools, its policies, its routes and its
autonomy level.

| ID | Rule | Verify |
|---|---|---|
| KS-1 | An admission policy MUST deny any agent identity from creating, updating or deleting authority objects (agent specs, tool bindings, mesh and network policies, routes, flags, admission policies), even when RBAC allows it. | Impersonate the agent with a privileged group; the change is still refused. |
| KS-2 | Agent service accounts MUST NOT have RBAC rights on authority objects. | `kubectl auth can-i patch <authority kinds> --as=<agent-sa>` returns `no`. |
| KS-3 | Authority objects SHOULD change only through the GitOps pipeline. Direct human edits SHOULD be refused or at least alerted. | A direct edit is refused or raises an alert. |
| KS-4 | Changes to authority objects MUST be reviewed by named owners (for example CODEOWNERS on the policy directories). | Repository settings. |
| KS-5 | Exclusions in the Keystone policy (for example the agent operator) MUST be named individually and justified in a comment. | Policy source. |

**Test pattern for KS-1.** Impersonating a privileged group passes RBAC, so a refusal can
only come from admission. `--dry-run=server` runs admission without saving anything, so the
test is safe even if the policy is missing. Running it needs impersonation rights. `agent`
stands for your agent framework's resource; any authority object works.

```bash
kubectl -n agents patch agent my-agent --type merge -p '{"metadata":{"labels":{"x":"y"}}}' \
  --as=system:serviceaccount:agents:my-agent --as-group=system:masters --dry-run=server
```

**Tool options:** Kyverno `ValidatingPolicy` or Kubernetes
ValidatingAdmissionPolicy matching `request.userInfo.username` by prefix.

## 8. Evidence (around every ring)

Every decision is recorded, keyed by the same identity, so you can answer an auditor
afterwards from records the system produced on its own.

| ID | Rule | Verify |
|---|---|---|
| EV-1 | Every allow and deny at every ring MUST be recorded with the caller identity, the rule that decided, and the outcome. | Query: allows and denies by caller identity for a time range. |
| EV-2 | Agent runs SHOULD be traced end to end with one trace id across every hop (user, agent, agent-to-agent, gateway, tool, backend), using OpenTelemetry `gen_ai.*` conventions for model and tool calls. | One trace shows the whole run, including the refused step. |
| EV-3 | Policy decisions MUST be first-class records (a span or event with rule, limit, requested value, decision), not only errors. | Decision record exists for a permit and for a deny. |
| EV-4 | Human approvals MUST be linked to the run they approve (a span link or reference), with who approved and when. | Approval record points back to the run. |
| EV-5 | Records MUST be complete (no sampling of decision records), retained for the required period, and tamper-evident. | Retention policy and integrity control documented and tested. |
| EV-6 | Runs SHOULD be graded from their traces. Example checks: the agent used tools in an expected order, did not retry after a refusal, executed what it proposed, and did not repeat sensitive values; suspected prompt injection is flagged for review. | Evaluator results attached to runs. |
| EV-7 | Custom telemetry names SHOULD be defined as a schema and checked in CI. | Schema check job. |

**What an auditor will ask for** (adapted from the Rizzo and Nissen talk in Credits):
1. A full account of the agent's actions.
2. Who authorised them, and the limits that applied.
3. Whether a person could halt or undo them.
4. An explanation of one specific decision, for the person it affected.
5. Proof that the records are complete, retained and tamper-evident.

**Tool options:** OpenTelemetry Collector, a trace store (Jaeger, Tempo),
Prometheus and Grafana for metrics, mesh proxy metrics (caller identity labels),
OpenTelemetry Weaver for schemas, trace-based evaluators.

**Pitfall:** metrics derived from traces (counts and durations) are not traces. They show
that something happened, not the steps of one run. Keep a trace store for EV-2 to EV-4.

---

## 9. Recommended tooling

The recommended stack is the set of tools this rulebook was built with, on
**Kubernetes 1.36**. All of it is open source or built into Kubernetes.
ValidatingAdmissionPolicy is stable from Kubernetes 1.30, so the admission rules need 1.30
or later.

| Ring | Need | Tool | Rules |
|---|---|---|---|
| Identity | The identity itself: one per agent | A dedicated Kubernetes ServiceAccount per agent, named after it | ID-1, ID-2 |
| Identity | Identity on the wire (mTLS), derived from the ServiceAccount | Linkerd | ID-1, ID-4 |
| Identity | Certificate chain and rotation | cert-manager (trust anchor and a short-lived identity issuer) | ID-4, ID-5 |
| Identity | One identity per agent, no `default` ServiceAccount | Kubernetes ValidatingAdmissionPolicy (CEL) | ID-2, ID-3 |
| Identity | Who can read keys or create pods | `kubectl auth can-i` | ID-5, ID-6 |
| Reach | Default-deny inbound for the agent namespace | Linkerd default inbound policy `deny` | RE-1, RE-5 |
| Reach | Inbound allow-lists by identity | Linkerd `Server`, `MeshTLSAuthentication`, `AuthorizationPolicy` | RE-2 |
| Reach | Find real callers before enforcing | Linkerd default inbound policy `audit` plus proxy logs | RE-6 |
| Capability | Tool list per agent | The agent framework's tool binding, plus a Reach allow-list per tool server | CA-1 |
| Capability | Tool server least privilege | Kubernetes RBAC (read-only role, no Secrets or ConfigMaps) plus the server's read-only mode | CA-3 |
| Keystone | Agents hold no rights on authority objects | Kubernetes RBAC | KS-2 |
| Keystone | Deny agents edits to authority objects, even with RBAC | Kubernetes ValidatingAdmissionPolicy (CEL) | KS-1, KS-5 |
| Keystone | Git as the only change path | Argo CD, with merge requests reviewed in the Git host | KS-3, KS-4 |
| Evidence | Allow and deny per caller identity | Linkerd proxy logs and proxy metrics (`client_id` label) | EV-1 |
| Testing | Test admission rules without changing anything | `kubectl ... --dry-run=server` | all admission rules |
| Testing | Prove a refusal | The target's proxy log (denial) plus its application log (empty) | RE-1 to RE-5 |

**Extending the stack.** For the remaining rules, options to evaluate:

- Outbound control (RE-3, RE-4): Linkerd `EgressNetwork` for the in-mesh layer, plus a
  policy-enforcing CNI (for example Cilium or Calico) or a cloud egress firewall for the
  bypass-proof layer.
- Application-layer identity (ID-7): SPIFFE/SPIRE.
- Per-call gateway, argument limits and platform-held keys (CA-2, CA-4, CA-5): an
  MCP-aware gateway such as agentgateway, with a policy engine such as Open Policy Agent.
- Autonomy levels and approvals (CA-6, CA-7): a feature-flag service such as OpenFeature
  with flagd, and a review step that resumes the run.
- Traces, decision records, retention and grading (EV-2 to EV-7): OpenTelemetry Collector
  with a trace store (Tempo or Jaeger), Prometheus and Grafana, write-once storage, and a
  trace-based evaluator.

---

## 10. How to roll out a rule

1. **Write the test first.** State the setup, the action, and the expected result, for both
   the allowed and the refused case.
2. **Run the baseline.** Record the behaviour before the rule.
3. **Ship in observe mode.** Warn or audit only, for example admission
   `validationActions: [Warn, Audit]` or a mesh default policy of `audit`. Nothing is blocked.
4. **Check real workloads.** Confirm every existing workload already complies, and look for
   hidden dependencies.
5. **Enforce.** Switch to deny.
6. **Re-run the test.** Record the refusal evidence (proxy denial line, admission error,
   empty application log).
7. **Write the rollback.** One line in the manifest comment: how to return to observe mode.

## 11. Maturity checklist

| Level | You can say |
|---|---|
| 0 | Agents run with whatever the platform gives them. Bounds live in the prompt. |
| 1 | Identity per agent; inbound default-deny with identity allow-lists; read-only tools. |
| 2 | Plus: outbound default-deny with a bypass-proof layer; enforced one-identity-per-agent; Keystone policy. |
| 3 | Plus: per-call gateway with argument limits; platform holds model keys; autonomy level as a setting with human approval. |
| 4 | Plus: end-to-end traces with decision records, linked approvals, retention and integrity, automated grading of runs. |

State your level honestly and name the rules you have verified. Mark the rest as planned.

## 12. Glossary

- **Authority object:** any object that grants an agent power: its spec, tool bindings,
  policies, routes, flags, service account bindings.
- **Bypass-proof:** enforced outside the agent's own pod (node, network, cloud), so a
  compromised container cannot skip it.
- **Default deny:** the absence of an allow rule means refusal.
- **Proposal:** an agent's suggested action, which a deterministic gate turns into an
  outcome or holds.

## 13. Credits and references

- [The boundary problem in agentic AI is a platform problem](https://www.cncf.io/blog/2026/10/29/the-boundary-problem-in-agentic-ai-is-a-platform-problem/),
  William Rizzo and Francesco Beltramini, CNCF blog, 29 October 2026. The inspiration for
  this rulebook.
- The Safe Operating Model (rings of least privilege): Francesco Beltramini, ControlPlane.
- "Verticalizing Platform Intelligence for Regulated Domains", William Rizzo and Kasper
  Borg Nissen, KubeCon + CloudNativeCon North America 2026 (Decide, Trace, Prove).
- Projects referenced: Linkerd, SPIFFE/SPIRE, cert-manager, Kyverno, Kubernetes
  ValidatingAdmissionPolicy, agentgateway, OpenFeature, OpenTelemetry, Prometheus,
  Grafana, MCP, A2A.

This rulebook is an independent write-up. It is not an official publication of any of the
projects or authors above.

## License

This work is licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).
You may share and adapt it for any purpose, including commercially, as long as you give
credit and indicate changes. See [LICENSE](LICENSE).

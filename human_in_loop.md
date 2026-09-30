In LLM agents, **Human-in-the-Loop (HITL)** means the agent can pause its autonomous execution and ask a human to **approve, reject, correct, clarify, or provide information** before continuing.

A useful mental model is:

> **LLM agent = Plan → Act → Observe → Decide → [Human checkpoint when needed] → Continue**

### 1. Simple approval workflow

Suppose an agent handles customer refunds:

```text
Customer: "I want a refund for my $500 order."

        ↓
      Agent
        ↓
Check order → Verify eligibility → Calculate refund
        ↓
   $500 refund
        ↓
 ┌───────────────────┐
 │ HUMAN APPROVAL?   │
 │ Refund $500?      │
 │ [Approve] [Reject]│
 └───────────────────┘
        ↓
      Agent
        ↓
Issue refund
        ↓
Tell customer
```

The important part is that the LLM **doesn't directly perform the irreversible action**. It produces a proposed action, which is intercepted by a human approval step.

---

## 2. Common HITL patterns

There are several different ways to implement this.

### A. Human approval before an action

The agent decides:

```json
{
  "action": "send_email",
  "to": "customer@example.com",
  "subject": "Your refund",
  "body": "Your refund has been approved..."
}
```

Instead of immediately calling the email tool:

```python
action = agent.plan()

if requires_approval(action):
    approval = ask_human(action)

    if approval == "approved":
        execute(action)
    else:
        cancel(action)
else:
    execute(action)
```

This is probably the most common HITL pattern.

**Useful for:**

* Sending emails
* Making purchases
* Deleting data
* Issuing refunds
* Publishing content
* Changing production systems
* Executing financial transactions

---

# 3. Human correction

HITL doesn't have to be just **approve/reject**.

The human can modify the agent's proposed action.

For example:

```text
Agent:
"I found 3 flights:

NYC → London
$1,200
Departing Friday"

Human:
"Don't book Friday. Book Saturday instead."

Agent:
"Understood. Searching Saturday flights..."
```

The human's input becomes another piece of context for the agent.

Conceptually:

```text
Agent proposal
      ↓
Human correction
      ↓
Updated state
      ↓
Agent continues
```

This is especially useful when the agent's interpretation is approximately correct but not completely correct.

---

# 4. Human clarification

Another very common pattern is:

```text
User:
"Book me a hotel in Paris."

Agent:
"What is your budget per night?"

       ↓

Human/User:
"Under $200."

       ↓

Agent:
Search hotels...
```

Here the agent reaches a state where it doesn't have enough information to safely continue.

So:

```python
if missing_required_information(state):
    question = generate_question(state)
    human_answer = get_human_input(question)

    state.update(human_answer)
```

This is often called a **clarification checkpoint**.

---

# 5. Human review of LLM output

The agent might produce something like:

```text
Draft:
"We have determined that your account violated our
terms of service and will be permanently suspended."
```

Before sending it, a human reviewer sees:

```text
┌─────────────────────────────────────────┐
│ Agent-generated response                │
│                                         │
│ "We have determined..."                 │
│                                         │
│ [Edit] [Approve] [Reject]              │
└─────────────────────────────────────────┘
```

The human can edit the response:

```text
"We've identified activity that may violate
our terms. Our team is reviewing the account."
```

Then the agent sends the edited version.

This is common in:

* Legal workflows
* Customer support
* Healthcare
* Compliance
* Journalism
* Financial services

---

# 6. Confidence-based HITL

You can also make HITL conditional.

For example:

```text
                 Agent
                   │
             Make decision
                   │
          ┌────────┴────────┐
          │                 │
       confidence        confidence
        > 95%              < 95%
          │                 │
     Execute          Ask human
```

For example:

```python
if confidence > 0.95:
    execute(action)
else:
    request_human_review(action)
```

But there is an important caveat:

**LLM self-reported confidence isn't necessarily reliable.**

Instead of simply asking:

> "How confident are you?"

you can define objective conditions:

```python
if transaction_amount > 1000:
    human_review()

elif action_is_irreversible:
    human_review()

elif policy_violation_detected:
    human_review()

else:
    execute()
```

This tends to be more robust.

---

# 7. Risk-based HITL

A particularly useful architecture is to classify actions by risk.

### Low risk

```text
Search Google
Read documentation
Summarize a document
Format a report
```

→ Agent acts autonomously.

### Medium risk

```text
Send an email
Modify CRM record
Create a support ticket
```

→ Human approval depending on conditions.

### High risk

```text
Transfer money
Delete production data
Sign a contract
Deploy production code
Terminate an account
```

→ Mandatory human approval.

You can represent this as:

```python
RISK_LEVELS = {
    "read": 0,
    "draft": 1,
    "modify": 2,
    "external_communication": 3,
    "financial": 4,
    "irreversible": 5
}

if risk_level(action) >= 4:
    require_human_approval()
```

This is often more practical than putting a human in every loop.

---

# 8. HITL in coding agents

Consider a coding agent.

User:

> "Fix the authentication bug."

The agent might:

```text
1. Inspect repository
2. Find authentication code
3. Identify bug
4. Modify auth.py
5. Run tests
6. Prepare commit
```

You might allow steps 1–5 automatically.

But before:

```text
git push origin main
```

the agent stops:

```text
┌──────────────────────────────────────┐
│ Agent wants to execute:             │
│                                      │
│ git push origin main                 │
│                                      │
│ Files changed: 4                     │
│ Tests: 127 passed                    │
│                                      │
│ [Approve] [Reject] [Inspect diff]   │
└──────────────────────────────────────┘
```

This is a very natural HITL boundary.

---

# 9. HITL in a multi-agent system

Imagine several agents:

```text
             ┌──────────────┐
             │ Manager Agent│
             └──────┬───────┘
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
 Research       Coding       Data Agent
 Agent          Agent
       │            │            │
       └────────────┼────────────┘
                    ↓
              Final proposal
                    ↓
              HUMAN REVIEW
                    ↓
                 Execute
```

For example, a software-development system could have:

* **Planner agent** — creates implementation plan
* **Research agent** — investigates libraries
* **Coding agent** — modifies code
* **Testing agent** — runs tests
* **Security agent** — checks vulnerabilities
* **Human** — approves final merge

The human doesn't need to supervise every LLM interaction.

They supervise the **important transition points**.

---

# 10. HITL as a state machine

For production agents, this is a very useful way to think about HITL.

```text
              ┌──────────────┐
              │    START     │
              └──────┬───────┘
                     ↓
                PLAN_ACTION
                     ↓
               EXECUTE_SAFE
                     ↓
              NEED_APPROVAL?
               /          \
             no            yes
             ↓              ↓
          EXECUTE       WAITING_HUMAN
                            ↓
                    ┌───────┴───────┐
                    ↓               ↓
                 APPROVE          REJECT
                    ↓               ↓
                 EXECUTE           STOP
                    ↓
                 OBSERVE
                    ↓
                  PLAN
```

The critical idea is:

> **Human intervention is a state in the agent's execution graph.**

It's not necessarily a separate application bolted onto the side.

---

# 11. What happens technically when the agent "pauses"?

Suppose an agent is executing:

```python
def agent():
    search_web()
    analyze_results()

    approval = human_approval(
        "Should I purchase this product?"
    )

    if approval:
        purchase()

    send_confirmation()
```

In a real system, you don't want to keep a Python process sitting there for hours waiting for the human.

Instead, you typically persist the agent's state:

```json
{
  "run_id": "abc123",
  "status": "WAITING_FOR_HUMAN",
  "current_step": "purchase",
  "context": {
    "product": "Laptop",
    "price": 1299,
    "reason": "User requested a laptop"
  }
}
```

Then the application displays:

```text
Approval required

Agent wants to purchase:

Laptop
$1,299

Reason:
Matches the user's requirements.

[Approve] [Reject]
```

The human clicks **Approve**.

The backend changes the state:

```json
{
  "status": "APPROVED"
}
```

Then the agent resumes from the checkpoint:

```text
WAITING_FOR_HUMAN
       ↓
    APPROVED
       ↓
   purchase()
       ↓
   observe()
       ↓
    continue
```

This **checkpoint → interrupt → resume** model is fundamental to robust HITL systems.

---

# 12. A more realistic architecture

A production agent might look like this:

```text
                   ┌──────────────┐
                   │     User     │
                   └──────┬───────┘
                          │
                          ↓
                 ┌─────────────────┐
                 │   Agent Runtime │
                 └────────┬────────┘
                          │
                 ┌────────▼────────┐
                 │   LLM / Planner │
                 └────────┬────────┘
                          │
                    Tool decision
                          │
              ┌───────────┴───────────┐
              ↓                       ↓
        Low-risk action          High-risk action
              │                       │
              ↓                       ↓
         Tool executes          HITL Gateway
                                      │
                               ┌──────▼──────┐
                               │ Human UI    │
                               └──────┬──────┘
                                      │
                              approve/reject/edit
                                      │
                               ┌──────▼──────┐
                               │ Agent State │
                               └──────┬──────┘
                                      │
                                      ↓
                                Agent resumes
```

There are therefore usually **four important components**:

1. **Agent/LLM**
2. **Tool execution layer**
3. **State/checkpoint store**
4. **Human approval interface**

---

# 13. Example: customer-support agent

Imagine an agent handling customer tickets.

### Step 1 — Agent receives ticket

> "I was charged twice for my subscription."

### Step 2 — Agent investigates

It calls:

```text
get_customer()
get_subscription()
get_payment_history()
```

It discovers:

```text
Charge 1: $49
Charge 2: $49
Likely duplicate payment
```

### Step 3 — Agent proposes action

```json
{
  "action": "refund",
  "amount": 49,
  "reason": "duplicate subscription charge"
}
```

### Step 4 — Policy engine

```text
Refund < $50 → automatic
Refund >= $50 → human approval
```

Since it's $49:

```text
Agent → refund()
```

No human required.

But if it were $500:

```text
Agent
  ↓
Human approval
  ↓
Approve
  ↓
refund()
```

This combination of **agent + policy engine + HITL** is very common in serious systems.

---

# 14. HITL vs Human-on-the-loop

There's a useful distinction.

### Human-in-the-loop

Human participates **inside the workflow**:

```text
Agent → Human → Agent → Action
```

### Human-on-the-loop

Agent runs autonomously, while humans monitor it and can intervene:

```text
Agent → Agent → Agent → Agent
          ↓
       Monitoring
          ↓
      Human can stop
```

For example, an autonomous coding agent might work for 30 minutes while a human watches its activity and can press:

```text
STOP AGENT
```

That's closer to **human-on-the-loop**.

---

# 15. Three levels of autonomy

A practical design is:

| Level | Agent behavior              | Human                     |
| ----- | --------------------------- | ------------------------- |
| 0     | Agent only suggests         | Executes everything       |
| 1     | Agent performs safe actions | Approves risky actions    |
| 2     | Agent mostly autonomous     | Handles exceptions        |
| 3     | Agent autonomous            | Human monitors/intervenes |

The goal isn't necessarily to maximize autonomy.

Instead, you want:

> **maximum useful autonomy subject to acceptable risk.**

For example:

```text
Read database                 → autonomous
Analyze data                  → autonomous
Draft email                   → autonomous
Send email                    → maybe approval
Change customer balance       → approval
Transfer $100,000             → mandatory approval
Delete production database    → mandatory approval
```

---

## The key architectural idea

When building an LLM agent, don't think of HITL as:

> "Ask a human whenever the LLM is confused."

A stronger design is:

```text
             Agent
               │
               ↓
        Proposed action
               │
        ┌──────┴───────┐
        │ Policy/Risk  │
        │    Engine    │
        └──────┬───────┘
               │
       ┌───────┴────────┐
       ↓                ↓
   Safe action       Risky action
       ↓                ↓
    Execute         Human review
                         │
                    ┌────┴────┐
                    ↓         ↓
                 Approve    Reject/Edit
                    ↓         ↓
                 Execute      Stop
```

# Human in the loop with REST APIs

Exactly — with **REST APIs**, the key is that the HTTP request cannot literally stay open waiting for a human for an arbitrary amount of time.

Instead, you normally turn the agent execution into a **durable workflow**:

> `POST → agent runs → pause → return "waiting for approval" → human approves through another API → agent resumes`

There are a few patterns. The most important one is the **two-request pattern**.

---

# 1. The basic REST HITL pattern

Imagine:

```http
POST /agent/tasks
```

User asks:

```json
{
  "request": "Refund order #1234"
}
```

Your API starts the agent.

```text
                 CLIENT
                   │
                   │ POST /agent/tasks
                   │
                   ▼
            ┌───────────────┐
            │ REST API      │
            └───────┬───────┘
                    │
                    ▼
             ┌─────────────┐
             │ Agent       │
             └──────┬──────┘
                    │
                    ▼
             Analyze order
                    │
                    ▼
             Refund = $500
                    │
                    ▼
             ┌─────────────┐
             │ NEED HUMAN  │
             │ APPROVAL    │
             └──────┬──────┘
                    │
                    ▼
             Save agent state
                    │
                    ▼
                 RETURN
                    │
                    ▼
              HTTP 202
```

The original request might return:

```http
HTTP/1.1 202 Accepted
```

```json
{
  "task_id": "task_abc123",
  "status": "waiting_for_approval",
  "approval_id": "approval_789"
}
```

**The HTTP connection is finished.**

The agent isn't holding the HTTP request open.

---

# 2. Then the human approves

Your frontend sees:

```text
Refund $500

The agent wants to refund order #1234.

        [ Approve ]   [ Reject ]
```

When the human clicks **Approve**:

```http
POST /approvals/approval_789
```

```json
{
  "decision": "approve"
}
```

Now your backend resumes the agent.

```text
       FRONTEND
           │
           │ POST /approvals/approval_789
           │
           ▼
     ┌───────────────┐
     │ REST API       │
     └───────┬───────┘
             │
             ▼
       Load agent state
             │
             ▼
       decision = approve
             │
             ▼
       Resume agent
             │
             ▼
       refund(order)
             │
             ▼
       Agent continues
```

The second API request doesn't necessarily need to return the final result immediately either.

For example:

```json
{
  "approval_id": "approval_789",
  "status": "approved",
  "task_id": "task_abc123"
}
```

---

# 3. The complete lifecycle

This is probably the most useful diagram to keep in mind:

```text
                    ┌─────────────┐
                    │   CLIENT    │
                    └──────┬──────┘
                           │
                           │ POST /tasks
                           ▼
                    ┌─────────────┐
                    │ REST API    │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ Agent       │
                    │ Runtime     │
                    └──────┬──────┘
                           │
                           ▼
                     Agent works
                           │
                           ▼
                  ┌──────────────────┐
                  │ Does this action │
                  │ require human?   │
                  └────────┬─────────┘
                           │ YES
                           ▼
                  ┌──────────────────┐
                  │ Save checkpoint  │
                  │                  │
                  │ status=WAITING   │
                  └────────┬─────────┘
                           │
                           ▼
                     HTTP 202
                           │
                           ▼
                    ┌─────────────┐
                    │   CLIENT    │
                    └─────────────┘


        ...later...


                    ┌─────────────┐
                    │   HUMAN     │
                    └──────┬──────┘
                           │
                      Approve
                           │
                           ▼
                    POST /approvals
                           │
                           ▼
                    ┌─────────────┐
                    │ REST API    │
                    └──────┬──────┘
                           │
                           ▼
                    Load checkpoint
                           │
                           ▼
                    Resume agent
                           │
                           ▼
                      Execute
                           │
                           ▼
                    Continue task
```

---

# 4. Your database becomes very important

Because REST is stateless, you need somewhere to store the agent's state.

For example:

```text
┌───────────────────────────────────────┐
│             AGENT STATE DB            │
├───────────────────────────────────────┤
│ task_id: task_123                     │
│                                       │
│ status: WAITING_FOR_APPROVAL          │
│                                       │
│ current_step: REFUND                  │
│                                       │
│ order_id: 1234                        │
│ refund_amount: 500                    │
│                                       │
│ conversation: [...]                   │
│                                       │
│ pending_approval_id: approval_456     │
└───────────────────────────────────────┘
```

Then:

```text
POST /tasks
      │
      ▼
 Agent
      │
      ▼
 SAVE STATE
      │
      ▼
 return 202
```

Later:

```text
POST /approvals/approval_456
      │
      ▼
 LOAD STATE
      │
      ▼
 UPDATE STATE
      │
      ▼
 RESUME AGENT
```

This is the fundamental trick.

---

# 5. Don't make this mistake

You might initially think:

```text
POST /agent

       │
       │
       │ Agent running...
       │
       │
       │ "Waiting for human..."
       │
       │
       │ 30 minutes later
       │
       ▼
HTTP response
```

Technically possible in some environments, but generally a bad architecture.

You have:

```text
Browser
   │
   │ HTTP connection
   ▼
Load Balancer
   │
   ▼
API Server
   │
   ▼
Agent
   │
   │
   │ WAIT 30 MINUTES
```

You can run into:

* HTTP timeouts
* load balancer timeouts
* serverless execution limits
* worker exhaustion
* connection failures
* scaling problems

Instead:

```text
POST /tasks
     │
     ▼
202 Accepted
     │
     X  ← connection ends

Agent continues independently
```

That's much better.

---

# 6. Polling pattern

Now the frontend needs to know what happened.

One simple solution is polling.

```text
Frontend
   │
   │ POST /tasks
   ▼
Backend
   │
   ▼
202
{
  "task_id": "123"
}
```

Then:

```text
Frontend
   │
   │ GET /tasks/123
   ▼
Backend
   │
   ▼
{
  "status": "running"
}
```

Again:

```text
Frontend
   │
   │ GET /tasks/123
   ▼
Backend
   │
   ▼
{
  "status": "waiting_for_approval"
}
```

Frontend displays:

```text
┌───────────────────────────────┐
│ Agent needs your approval     │
│                               │
│ Refund $500                   │
│ Order #1234                   │
│                               │
│ [ Approve ]  [ Reject ]       │
└───────────────────────────────┘
```

After approval:

```text
POST /approvals/456
```

Then polling continues:

```text
GET /tasks/123

→ running

GET /tasks/123

→ running

GET /tasks/123

→ completed
```

---

# 7. REST API design

A reasonable API could look like this:

```text
POST   /agent/tasks
GET    /agent/tasks/{taskId}

GET    /agent/tasks/{taskId}/approvals

POST   /agent/approvals/{approvalId}

POST   /agent/approvals/{approvalId}/reject

POST   /agent/tasks/{taskId}/cancel
```

For example:

### Start

```http
POST /agent/tasks
```

```json
{
  "input": "Refund order #1234"
}
```

Response:

```http
202 Accepted
```

```json
{
  "task_id": "task_123",
  "status": "running"
}
```

---

### Get status

```http
GET /agent/tasks/task_123
```

Response:

```json
{
  "task_id": "task_123",
  "status": "waiting_for_approval",
  "pending_approval": {
    "approval_id": "approval_456",
    "type": "financial",
    "description": "Refund $500 for order #1234"
  }
}
```

---

### Approve

```http
POST /agent/approvals/approval_456
```

```json
{
  "decision": "approve"
}
```

Response:

```json
{
  "approval_id": "approval_456",
  "decision": "approve",
  "task_id": "task_123",
  "status": "resuming"
}
```

---

# 8. What if the agent is running in a worker?

This is actually a very common architecture.

```text
                 ┌──────────────┐
                 │    Client    │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ REST API     │
                 └──────┬───────┘
                        │
                 Create task
                        │
                        ▼
                 ┌──────────────┐
                 │ Message      │
                 │ Queue        │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Agent Worker │
                 └──────┬───────┘
                        │
                        ▼
                     LLM
                        │
                        ▼
                  Tool calls
                        │
                        ▼
                 NEED APPROVAL
                        │
                        ▼
                 Save checkpoint
                        │
                        ▼
                      STOP
```

Notice something important:

**The agent worker itself can terminate.**

You don't need to keep an agent process alive while waiting for the human.

Later:

```text
Human
  │
  │ POST /approvals/456
  ▼
REST API
  │
  ▼
Queue
  │
  ▼
New Agent Worker
  │
  ▼
Load checkpoint
  │
  ▼
Resume
```

This is a very scalable architecture.

---

# 9. Queue-based architecture

The complete picture can therefore be:

```text
                           ┌───────────────┐
                           │    Client     │
                           └───────┬───────┘
                                   │
                             HTTP REST
                                   │
                                   ▼
                         ┌─────────────────┐
                         │   API Gateway   │
                         └────────┬────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
             ┌─────────────┐             ┌─────────────┐
             │ Task DB     │             │ Message     │
             │             │             │ Queue       │
             └─────────────┘             └──────┬──────┘
                                                │
                                                ▼
                                         ┌─────────────┐
                                         │Agent Worker │
                                         └──────┬──────┘
                                                │
                                                ▼
                                              LLM
                                                │
                                     ┌──────────┴─────────┐
                                     │                    │
                                  safe                 risky
                                     │                    │
                                     ▼                    ▼
                                   Tool             Save state
                                                          │
                                                          ▼
                                                  WAITING_HUMAN
```

Then:

```text
              HUMAN
                │
                │ POST /approvals/{id}
                ▼
           ┌──────────┐
           │ REST API │
           └────┬─────┘
                │
                ▼
             Task DB
                │
                ▼
        publish RESUME event
                │
                ▼
             Queue
                │
                ▼
          Agent Worker
                │
                ▼
        Load checkpoint
                │
                ▼
        Continue execution
```

---

# 10. Another option: Webhooks / SSE

Polling isn't the only way for the frontend to know about HITL.

You could use:

```text
REST API
   │
   ▼
Agent
   │
   ▼
Waiting for human
   │
   ▼
Frontend receives event
```

For example, **Server-Sent Events (SSE)**:

```text
GET /agent/tasks/123/events
```

Server sends:

```text
event: approval_required

data:
{
  "approval_id": "456",
  "message": "Refund $500?"
}
```

The UI immediately displays the approval dialog.

After approval:

```text
POST /agent/approvals/456
```

Agent resumes.

You can also use WebSockets, although for many agent-status/notification use cases, REST + polling or SSE is simpler.

---

# 11. The important distinction: REST API vs agent execution

This is where the architecture becomes clearer.

You actually have **two different things**:

```text
             REST API
                │
                │
        request / response
                │
                ▼
        ┌───────────────┐
        │ Agent Runtime │
        │               │
        │ long-running  │
        │ workflow      │
        └───────────────┘
```

REST doesn't need to represent the entire agent lifecycle in one HTTP request.

Instead:

```text
HTTP REQUEST
     ↓
Create workflow
     ↓
HTTP RESPONSE
     ↓
              workflow continues
              independently
                     ↓
                checkpoint
                     ↓
                HUMAN
                     ↓
              resume workflow
                     ↓
                checkpoint
                     ↓
                  done
```

That's the conceptual shift.

---

# 12. A concrete example

Suppose your company has:

```text
POST /api/agent
```

User says:

> "Send an email to the customer explaining their overdue invoice."

Agent does:

```text
1. Get customer
2. Get invoice
3. Check amount
4. Generate email
5. Decide whether approval is required
```

Suppose your policy says:

```text
Invoice < $100
    → automatically send

Invoice >= $100
    → human approval
```

For a $750 invoice:

```text
POST /api/agent
       │
       ▼
┌─────────────────┐
│ Agent           │
│                 │
│ Generate email  │
└────────┬────────┘
         │
         ▼
   $750 invoice
         │
         ▼
┌─────────────────┐
│ POLICY ENGINE   │
│                 │
│ approval=true   │
└────────┬────────┘
         │
         ▼
 SAVE CHECKPOINT
         │
         ▼
 HTTP 202
```

Response:

```json
{
  "taskId": "123",
  "status": "WAITING_FOR_APPROVAL"
}
```

The UI:

```text
┌─────────────────────────────────────┐
│ Approval Required                   │
├─────────────────────────────────────┤
│                                     │
│ Invoice: #INV-8273                  │
│ Amount: $750                        │
│                                     │
│ Proposed email:                    │
│                                     │
│ "Dear John,                         │
│  Your invoice of $750 is overdue..."│
│                                     │
│ [ Edit ] [ Reject ] [ Approve ]     │
└─────────────────────────────────────┘
```

Human clicks **Edit**.

They change:

```text
"Your invoice is overdue."
```

to:

```text
"Your invoice is currently outstanding.
Please contact us if you need assistance."
```

Then:

```http
POST /api/approvals/456
```

```json
{
  "decision": "approve",
  "modified_output": "Your invoice is currently outstanding..."
}
```

Then:

```text
                 Approval API
                       │
                       ▼
                Load checkpoint
                       │
                       ▼
                Inject human input
                       │
                       ▼
                Resume agent
                       │
                       ▼
                  send_email()
                       │
                       ▼
                    DONE
```

---

# 13. The data model I'd recommend

For this kind of architecture, you'll usually want something like:

```text
TASK
────────────────────────────
id
status
input
created_at
updated_at
current_step
result
error
```

and:

```text
AGENT_CHECKPOINT
────────────────────────────
task_id
state
messages
tool_results
current_node
created_at
```

and:

```text
HUMAN_APPROVAL
────────────────────────────
id
task_id
status
action
payload
requested_at
resolved_at
resolved_by
human_input
```

So you have:

```text
Task
 │
 ├── Checkpoint
 │
 ├── Approval
 │
 └── Result
```

---

# 14. One subtle but VERY important point

Don't make the human approval itself the only protection.

Imagine the agent says:

```json
{
  "action": "transfer_money",
  "amount": 500000
}
```

The UI says:

```text
Transfer $500,000?
```

Human clicks approve.

Your backend should **still validate the action**.

For example:

```text
                 Agent
                   │
                   ▼
              Proposed action
                   │
                   ▼
             Policy engine
                   │
                   ▼
             Human approval
                   │
                   ▼
             Policy validation
                   │
                   ▼
            Execute tool
```

The LLM should not be the ultimate authority.

---

# 15. The architecture in one picture

If you're building a REST-based LLM agent system, I'd think about it like this:

```text
                         CLIENT
                           │
                           │ HTTP
                           ▼
                   ┌───────────────┐
                   │   REST API    │
                   └───────┬───────┘
                           │
                  create / query /
                  approve / reject
                           │
                           ▼
                  ┌────────────────┐
                  │  Task / State  │
                  │     Store      │
                  └───────┬────────┘
                          │
                          │ events
                          ▼
                    ┌───────────┐
                    │   Queue   │
                    └─────┬─────┘
                          │
                          ▼
                  ┌────────────────┐
                  │ Agent Worker   │
                  └───────┬────────┘
                          │
                          ▼
                     ┌────────┐
                     │   LLM  │
                     └────┬───┘
                          │
                     tool call
                          │
                          ▼
                   ┌─────────────┐
                   │ Risk/Policy │
                   └──────┬──────┘
                          │
              ┌───────────┴───────────┐
              │                       │
            SAFE                    RISKY
              │                       │
              ▼                       ▼
          Execute tool          Save checkpoint
                                      │
                                      ▼
                              WAITING_FOR_HUMAN
                                      │
                                      ▼
                                  REST API
                                      │
                                      ▼
                                    HUMAN
                                      │
                              approve/reject/edit
                                      │
                                      ▼
                                  REST API
                                      │
                                      ▼
                                    Queue
                                      │
                                      ▼
                                Agent Worker
                                      │
                                      ▼
                               Resume from
                                checkpoint
```

**So the REST API doesn't "wait for the human."** The REST API creates and controls a **durable agent workflow**. When HITL is needed, the workflow is persisted in a waiting state; a later REST request supplies the human decision and triggers the workflow to resume.

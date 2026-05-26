# Multi-Channel Tech Helper — Camunda 8 AI Agent Example

This repository is a small, self-contained example showing how a single AI agent
running inside a Camunda 8 process can talk to a user through **more than one
channel at the same time** — specifically through **Camunda Tasklist** (a web
form) and through **Slack**. The agent, the tools it can call, and the
conversation memory are all defined once in BPMN; the channel a given
conversation uses is just a matter of which start event fired the process and
which response tool the agent decides to call.

The example is deliberately built around a generic "answer questions about our
tech stuff" use case so the moving parts stay visible. The interesting bit is
not the domain — it is the **multi-channel pattern**.

---

## What this project does

### The multi-channel idea

Most agent examples are single-channel: a user opens a form, the agent answers,
done. Real users do not work that way. They ask things in Slack, they fill in
tasks in a queue, they expect follow-ups in the same place they started the
conversation. The same agent should be reachable from wherever the user
already is, and it should respond *in that channel* — not bounce them somewhere
else.

This process demonstrates that pattern with two channels wired into one agent:

- **Camunda Tasklist channel** — a process is started by submitting the
  `Get Question About tech Stuff` form. The agent answers by completing the
  `Show Answer to Question` user task, which the user picks up in Tasklist and
  can reply to with a follow-up question on the same task form.
- **Slack channel** — the same process can also be started by `@mention`-ing
  the Slack app in a channel. The agent answers via `chat.postMessage` into
  the Slack thread, and listens on that thread for follow-up replies using a
  Slack inbound intermediate catch event.

Both channels feed into the **same ad-hoc sub-process agent**, with the same
system prompt, the same tools, and the same in-process memory. The agent
chooses how to respond based on which channel the question came in on: if the
question arrived from Slack, it calls the `Show Answer in Slack` tool; if it
arrived from a form, it calls the `Show Answer to Question` user task tool.

### The agent and its tools

The agent is implemented as an **ad-hoc sub-process** using the Camunda
**AI Agent (Job Worker)** element template, backed by **Claude Sonnet 4.6 on
Amazon Bedrock**. Inside the sub-process the following tools are wired up and
exposed to the LLM:

| Tool | BPMN element | Purpose |
|------|--------------|---------|
| `Get List of Tech Stuff` | HTTP REST connector | Calls a public sample API (`https://api.restful-api.dev/objects`) to get a list of devices. Stands in for any external system-of-record. |
| `Query Company Information` | Embeddings Vector DB connector (Bedrock Titan + Amazon Managed OpenSearch) | RAG lookup against a `tech-stuff-knowagebase` index for policy / product / company-specific questions. |
| `Get the Current Time and date` | Script task (FEEL `now()`) | Returns the current timestamp — a minimal example of a "local" tool. |
| `Show Answer to Question` | User task with form | Posts the answer into **Camunda Tasklist**. The user can submit a follow-up question on the same form. Has a 5-minute boundary timer so the agent does not wait forever. |
| `Show Answer in Slack` | Slack connector (`chat.postMessage`) | Posts the answer back into the originating **Slack thread**. After posting, the process waits on an event-based gateway for either a threaded reply (Slack inbound intermediate catch event) or a 4-minute timeout. |

The agent's system prompt instructs it to **only communicate through the
provided tools** — it cannot just "return text". This is what forces every
reply to go out through a real channel, and what lets you add or swap channels
later without touching the agent itself.

### Conversation flow

There are two start events on the process:

1. `Get Question About tech Stuff` — a normal **form start event** for the
   Tasklist channel.
2. `Info From Slack` — a **Slack message start event** that fires on an
   `app_mention` and captures `senderId`, `channelId`, `threadTs`, and the
   message text into a `slackRequest` variable.

Both flow into the same ad-hoc sub-process. The agent then loops: pick a tool,
call it, look at the result, decide whether it is done. When it calls one of
the "show the user the answer" tools, the process waits for the user's
follow-up. The Slack reply path uses Slack's `thread_ts` as the correlation
key so follow-ups land back in the right process instance.

---

## Installing on Camunda 8 SaaS

This section walks through getting the example running on a Camunda 8 SaaS
cluster. You will need:

- A Camunda 8 SaaS account and a cluster on **version 8.9+** (the AI Agent
  element template targets `8.9.0`).
- An AWS account with **Bedrock access to `anthropic.claude-sonnet-4-6`**
  enabled in the region you plan to use. Also enable **Titan Text Embeddings V2**
  *if* you want to use the vector database tool (optional — see below).
- A Slack workspace where you can install a custom app.
- *(Optional)* An **Amazon Managed OpenSearch** (or OpenSearch Serverless)
  collection if you want the `Query Company Information` RAG tool to do
  anything useful. The process will run fine without one.

### 1. Deploy the BPMN and forms

1. Log in to Camunda 8 SaaS and open **Web Modeler**.
2. Create a new project and upload the three files from this repository:
   - `Multi-Channel Tech Question Agent.bpmn`
   - `Get Question About tech Stuff.form`
   - `Show Answer to Question.form`
3. Open the BPMN file and click **Deploy** → select your cluster → **Deploy**.
   Web Modeler will deploy the forms together with the process.

> The process id is `Multi-Channel-Agent`. After deployment you can start a
> Tasklist instance from the **Tasklist** app by picking the form start event.

### 2. Configure cluster secrets

The process references the following secrets via `{{secrets.NAME}}`
placeholders. Add each of these to your **cluster's Secrets** in the Camunda
Console (Console → Cluster → **Secrets**). Secret values are injected into
connectors at runtime — they never appear in the BPMN itself.

| Secret name | Used by | What it is |
|-------------|---------|------------|
| `AWS_REGION` | AI Agent (Bedrock), Vector DB | AWS region for Bedrock and OpenSearch, e.g. `us-east-1`. |
| `AWS_ACCESS_KEY` | AI Agent (Bedrock), Vector DB embeddings | AWS access key id for an IAM user/role with `bedrock:InvokeModel` on the Claude and Titan models. |
| `AWS_SECRET_KEY` | AI Agent (Bedrock), Vector DB embeddings | Matching AWS secret access key. |
| `AWS_VECTOR_ACCESS` *(optional)* | Vector DB store | Access key id with read access to the OpenSearch collection. Only needed if you want the `Query Company Information` tool to work. Can be the same as `AWS_ACCESS_KEY`. |
| `AWS_VECTOR_SECRET` *(optional)* | Vector DB store | Matching secret for the OpenSearch credentials. |
| `AWS_LONGTERM_MEM` *(optional)* | Vector DB store | The OpenSearch **server URL** (e.g. `https://search-xxx.<region>.es.amazonaws.com` or your serverless collection endpoint). |
| `SLACK_HAWK_OATH_TOKEN` | Slack outbound connector | Slack **Bot User OAuth Token** (`xoxb-…`). |
| `SLACK_HAWK_WEBHOOK_ID` | Slack inbound start + intermediate events | A unique id you choose for this webhook (used as the URL path segment Slack will POST to — see the Slack section below). |
| `SLACK_HAWK_SIGINING_SECRET` | Slack inbound start + intermediate events | Slack app **Signing Secret**, used by the inbound connector to verify the request signature. |

> The three `AWS_VECTOR_*` / `AWS_LONGTERM_MEM` secrets are only required if
> you want to enable the **vector database (RAG) tool**. If you skip the
> vector store entirely, you can leave these unset — but you should also
> remove the `Query Company Information` task from the BPMN (or just leave
> it; see the next section for what happens if you do).

> The secret names with `SLACK_HAWK_…` and the `tech-stuff-knowagebase` index
> name are baked into the BPMN as written. If you would rather use different
> names, edit the corresponding `zeebe:input` and `zeebe:property` values in
> the BPMN before deploying.

### 3. Prepare the vector store *(optional)*

**You can skip this step entirely** if you just want to try the multi-channel
pattern. The agent works fine without a vector database — it simply will not
have any company-specific knowledge to draw on for the `Query Company
Information` tool.

If you do want RAG, the `Query Company Information` tool expects an Amazon
Managed OpenSearch index called **`tech-stuff-knowagebase`** with documents
embedded using **Titan Text Embeddings V2** at **1024 dimensions**
(un-normalised). Pre-populate it with whatever knowledge base you want the
agent to draw on.

If you leave the index empty or do not create it at all, the process has a
boundary error handler on the tool that catches `index_not_found` and
returns `"Nothing was found"`, so the agent will still answer questions —
it just falls back to the other tools and its own knowledge.

If you would rather **remove the tool entirely** so the agent never tries to
call it, open the BPMN in Web Modeler and delete the `Query Company
Information` service task from the ad-hoc sub-process before deploying. You
can then ignore the `AWS_VECTOR_ACCESS`, `AWS_VECTOR_SECRET`, and
`AWS_LONGTERM_MEM` secrets.

### 4. Build the Slack app

The Slack channel needs a Slack app installed into your workspace with the
right permissions and event subscriptions. You only have to do this once.

#### 4a. Create the app

1. Go to <https://api.slack.com/apps> and click **Create New App** →
   **From scratch**.
2. Give it a name (e.g. *Tech Helper Hawk*) and pick the workspace to install
   it into.

#### 4b. Permissions (Bot Token Scopes)

Under **OAuth & Permissions → Scopes → Bot Token Scopes**, add at least:

- `app_mentions:read` — to receive `@mention` events that start the process.
- `chat:write` — so the `Show Answer in Slack` tool can post replies.
- `channels:history` — so the inbound intermediate catch event can read
  threaded replies.
- `groups:history`, `im:history`, `mpim:history` — add these too if you also
  want the bot to work in private channels and DMs.

Then click **Install to Workspace** at the top of the same page. After
installing, copy the **Bot User OAuth Token** (`xoxb-…`) — this goes into the
`SLACK_HAWK_OATH_TOKEN` cluster secret.

#### 4c. Signing secret

From the app's **Basic Information** page, copy the **Signing Secret** and
put it in the `SLACK_HAWK_SIGINING_SECRET` cluster secret. The Camunda
inbound connector uses this to verify that incoming requests really came from
Slack.

#### 4d. Event Subscriptions

Both the start event and the in-thread response event in this process are
implemented with **Camunda's Slack inbound connector**. That connector
exposes a webhook URL on your cluster of the form:

```
https://<region>-<cluster-id>.connectors.camunda.io/inbound/<SLACK_HAWK_WEBHOOK_ID>
```

You can find the exact URL after deploying the process by opening the
inbound connector in the Web Modeler and looking at the **Webhook** tab, or
from the **Connectors** view in Console. Pick any URL-safe string for
`SLACK_HAWK_WEBHOOK_ID` (e.g. `tech-helper-hawk`) and reuse it for both the
start event and the intermediate catch event — the BPMN already references
the same secret in both places.

Then in Slack:

1. Go to **Event Subscriptions** and toggle **Enable Events** on.
2. Paste the webhook URL above into **Request URL**. Slack will send a
   `url_verification` challenge; the BPMN has a `verificationExpression`
   wired up to respond to it automatically, so the URL should verify
   immediately.
3. Under **Subscribe to bot events**, add:
   - `app_mention` — fires the process start event.
   - `message.channels` — needed so threaded replies are delivered to the
     intermediate catch event. Add `message.groups`, `message.im`, and
     `message.mpim` as well if you want private channel / DM follow-ups.
4. **Save Changes**, then **reinstall the app** when Slack prompts you to
   apply the new scopes.

#### 4e. Try it

Invite the bot into a channel (`/invite @your-app-name`), then mention it
with a question:

```
@tech-helper-hawk what objects do we have in our catalogue?
```

You should see a process instance start in Operate, the agent call a tool or
two, and a reply appear in the Slack thread. Reply in that thread to send a
follow-up — the inbound intermediate catch event correlates on `thread_ts`,
so the same running process instance picks it up.

For the **Tasklist channel**, just open Tasklist, click **Start process**,
pick the `Get Question About tech Stuff` form, and ask a question. The agent
will create a user task with the answer; complete it with a follow-up
question to continue the conversation, or just complete it empty to finish.

---

## File overview

- `Multi-Channel Tech Question Agent.bpmn` — the process definition,
  including the AI Agent ad-hoc sub-process, both start events, all tools,
  and the Slack thread response gateway.
- `Get Question About tech Stuff.form` — the form for the Tasklist start
  event. Captures a single `question` field.
- `Show Answer to Question.form` — the form rendered on the user task the
  agent creates to deliver an answer in Tasklist. Shows the markdown answer
  and accepts a `followupQuestion`.

---

## Extending the example

A few natural next steps if you want to use this as a starting point:

- **Add a third channel** — e.g. Microsoft Teams or email. Add a new start
  event for the inbound side and a new "send response" service task as
  another tool inside the ad-hoc sub-process. The agent will pick it up
  automatically as long as the tool has a clear `documentation` description.
- **Swap the model** — the `provider.type` / `provider.bedrock.model.model`
  inputs on the AI Agent task can point at any Bedrock model you have
  access to (or you can switch the provider entirely to OpenAI / Anthropic
  direct / Azure OpenAI by changing the provider type).
- **Replace the sample API** — the `Get List of Tech Stuff` HTTP connector
  is pointed at a public demo endpoint. Swap it for your own internal API
  to turn this into a real internal helpdesk agent.

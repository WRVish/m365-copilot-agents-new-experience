# HR Leave Assistant: Copilot Studio Implementation Guide

A beginner training build that covers the three pillars of a Microsoft Copilot Studio agent: knowledge sources, topics, and Power Automate agent flows. Verified against Microsoft Learn documentation as of July 2026.

**Design rule for this build:** the agent answers only from the leave policy document. If the answer is not in the knowledge source, it never answers from the internet or general knowledge. Instead it escalates to a human through a Teams adaptive card and an email.

The mental model for attendees: knowledge answers, topics structure, tools act, HR catches the rest.

> **List schema:** the full column definitions for the SharePoint list used here live in [`leave-requests-list-schema.md`](./leave-requests-list-schema.md). Create the list from that reference before you start.

## The four building blocks

| Pillar              | What it does in this demo                                                                           |
| ------------------- | --------------------------------------------------------------------------------------------------- |
| Knowledge           | A leave policy document answers questions like "How many annual leave days do I get?"               |
| Topic               | Apply for Leave collects a structured request through guided questions                              |
| Tool 1 (agent flow) | Submit Leave Request writes to a SharePoint list, emails HR, and returns a reference number         |
| Tool 2 (agent flow) | Escalate to HR posts an adaptive card to a Teams channel and emails HR when the agent cannot answer |

## Build order at a glance

| Stage | What you build                                                     | Section |
| ----- | ------------------------------------------------------------------ | ------- |
| 1     | Prerequisites: SharePoint list, policy document, Teams channel     | 1       |
| 2     | Create and configure the agent: grounding, authentication, prompts | 2       |
| 3     | Upload the knowledge source and test it                            | 3       |
| 4     | Apply for Leave topic: questions, Cancel path, question fallbacks  | 4       |
| 5     | Flow 1 (Submit Leave Request) and wire it into the topic           | 5       |
| 6     | Flow 2 (Escalate to HR), rebuild Escalate, point Fallback at it    | 6       |
| 7     | Conversation Start and On Error system topics                      | 7       |
| 8     | Test end to end, publish, share the demo website link              | 8       |

## 1. Prepare before the session

**SharePoint list named Leave Requests.** Create it on any site you own: Site contents, New, List, Blank list. The complete column definitions, types, and choice values are in [`leave-requests-list-schema.md`](./leave-requests-list-schema.md), which also includes an optional PnP PowerShell provisioning script. In short, the list needs Title, LeaveType, StartDate, EndDate, Reason, and Status.

**Leave policy document** (Word or PDF, one or two pages): annual entitlement, sick leave rules, notice period, carry forward, how to apply. Write it yourself so you know exactly what answers it contains, and deliberately leave one common topic out, for example overtime, so you can demonstrate escalation.

**Teams destination for escalations:** a team, for example HR, with a channel named Leave Escalations that you can post to. Create it before the session.

**Copilot Studio access** at copilotstudio.microsoft.com in an environment where you can publish. The agent and both flows must live in the same environment.

**Upload the knowledge in advance.** Indexing takes a few minutes. Never upload during the live session.

**Pre-consent connections.** Run both flows once before the session so the SharePoint, Outlook, and Teams connection prompts are already handled.

**Backup agent.** Keep a fully built, published copy ready in case anything misbehaves live.

## 2. Create and configure the agent

### 2.1 Create the agent

Sign in to copilotstudio.microsoft.com and confirm the environment picker (top right) is on your demo environment.

Select Create, then New agent. You can describe the agent in natural language and let AI generate everything, but for training choose Skip to configure so attendees see every field explicitly.

- **Name:** HR Leave Assistant.
- **Description:** Helps employees with leave policy questions and leave applications.
- **Instructions:**

  > You help employees understand the company leave policy and submit leave requests. Answer only from the leave policy knowledge provided. Never answer from general knowledge or the public internet. If the leave policy does not contain the answer, do not guess; tell the user you will pass the question to the HR team. Be concise and professional.

Select Create. You land on the agent's Overview page. New agents use generative orchestration by default, which affects how the topic triggers in Part 3.

### 2.2 Studio basics used throughout this guide

Five mechanics that every later step assumes:

- **Test pane.** The Test button (top right) opens the Test your agent panel. It reflects saved changes immediately; no publishing needed while you build.
- **Restart after edits.** Use the refresh icon at the top of the Test pane to start a fresh conversation after changing a topic, otherwise you may be testing the old path mid-conversation.
- **Save per topic.** Every topic canvas has its own Save button (top right). Save before leaving a topic or your node changes are lost.
- **Adding nodes.** The + icon between nodes opens the node menu (Ask a question, Send a message, Add a tool, Topic management, Set a variable value, and so on). Deleting a node: select it, the three dots menu, Delete.
- **Variables.** The `{x}` icon opens the variable picker wherever values are set: topic variables on one tab, System variables (like Activity.Text or User.DisplayName) on another. To insert a variable inside a message, use the `{x}` control on the message node.

### 2.3 Lock the agent to your knowledge (grounding)

This is the configuration that enforces "no answers from the internet". Do all three:

- On the Overview page, in the Knowledge section, turn off "Use general knowledge" (the AI's own training knowledge). The same options are also reachable under Settings, Generative AI.
- Turn off web search or public website answers if the option is present in your tenant. With both off, the agent can only answer from sources you add on the Knowledge page.
- Under Settings, Generative AI, set content moderation to High. Higher moderation returns fewer but more accurate generative answers, and more misses. Misses are exactly what you want here, because every miss routes to the Fallback system topic, which you will wire to human escalation in Part 5.

**Teaching point for the session:** grounding is enforced three ways in this build. Settings (general knowledge and web search off), instructions (the "never answer from general knowledge" line), and conversation design (Fallback and Escalate route every miss to a human). Belt, braces, and a safety net.

### 2.4 Authentication

Settings, Security, Authentication:

- For the live demo on the demo website, select No authentication. A public demo link only works this way. Save.
- Consequences to state out loud: SharePoint knowledge sources will not return answers to unauthenticated users, and System.User variables (DisplayName, Email) are blank. That is why this build uses an uploaded file as knowledge and asks users for contact details during escalation.
- For real deployment in Teams, use Authenticate with Microsoft. SharePoint knowledge then respects each user's permissions, and System.User.DisplayName and System.User.Email fill automatically.

### 2.5 Finishing touches

- On the Overview page, add suggested prompts so users know what to type: How many annual leave days do I get?, I want to apply for leave, I have a question for HR.
- Optionally set an agent icon under Settings, Details, for a polished look on stage.

## 3. Add the knowledge source

1. Go to the Knowledge page and select Add knowledge. The dialog shows the available source types: Files, SharePoint, public websites, Dataverse.
2. Choose Files and upload the leave policy document.
3. Give it a clear name and description, for example: Company leave policy: entitlements, notice periods, carry forward rules. The description helps orchestration decide when to search this source.
4. Wait for indexing to complete (the source shows as Ready on the Knowledge page, usually within a few minutes), then test in the Test pane: "How many days of annual leave do I get?"

**Why a file and not SharePoint for the live demo:** SharePoint knowledge requires user authentication, and the agent only surfaces content the signed-in user can access. It works in the Test pane and in Teams, but on an unauthenticated demo website it returns nothing. An uploaded file works everywhere. Do show the SharePoint option in the dialog and explain the authentication dependency, since the audience is SharePoint focused, but run the demo on the uploaded file.

## 4. Create the Apply for Leave topic

### 4.1 Create the topic and trigger

Go to the Topics page, select Add a topic, then From blank. Name it Apply for Leave (select the title at the top of the canvas to rename).

Configure the trigger. With generative orchestration (the default), select the trigger node and write a description that tells the agent the topic's purpose:

> Use this topic when an employee wants to apply for, request, or submit leave or time off.

If you demo classic orchestration instead, add 5 to 10 trigger phrases such as: apply for leave, request time off, submit leave application, take leave, book annual leave, request sick leave.

### 4.2 Add the question nodes

Add these Question nodes in order (+ icon, Ask a question). For question 2, set Identify to Multiple choice options, then type each option under Options for user (+ New option). For the others, pick the entity shown in the Identify column. Each question saves its answer to a variable: select the variable chip on the node and rename it in the properties pane so the flow mapping reads cleanly on screen.

| #   | Question text               | Identify                                              | Variable     |
| --- | --------------------------- | ----------------------------------------------------- | ------------ |
| 1   | What's your full name?      | User's entire response                                | EmployeeName |
| 2   | What type of leave?         | Multiple choice options: Annual, Sick, Unpaid, Cancel | LeaveType    |
| 3   | What's the start date?      | Date and time                                         | StartDate    |
| 4   | And the end date?           | Date and time                                         | EndDate      |
| 5   | Briefly, what's the reason? | User's entire response                                | Reason       |

**Note on question 1:** in Teams you would use System.User.DisplayName instead of asking. Asking the question keeps the demo working on the unauthenticated demo website.

### 4.3 Add the Cancel escape path

Copilot Studio automatically creates a conditional branch for each multiple choice option, and each option appears as a button in the chat (users can also type their answer). Build questions 3 to 5 on the branch shared by Annual, Sick, and Unpaid. On the Cancel branch of question 2:

1. Add a Message node: No problem, I've cancelled this request.
2. Add an End current topic node (under Topic management).

This is the single most effective fix for stuck users: there is always a visible button out. Save the topic.

### 4.4 Configure fallback behaviour on every question node

Open each Question node's properties (select the node, then Question behavior in the properties pane) and set the following.

**Reprompt.** Repeat up to 2 times is the default (other options: Repeat once, Don't repeat). Keep 2, and customise the Retry prompt so the retry actually helps:

- Question 2 (LeaveType): Please tap one of the options: Annual, Sick or Unpaid. Tap Cancel to stop.
- Questions 3 and 4 (dates): Please type a date like 20 July 2026.

**No valid entity found.** This controls what happens after the maximum retry count is reached. The options are:

- Escalate: redirects to the Escalate system topic (the default)
- Set variable to value: continue with a default value
- Set variable to empty (no value): continue, then check with a Condition node
- Redirect to a topic

Keep Escalate as the setting and customise the No entity found message (for example: Let me pass this to HR instead.). Because Part 5 rebuilds the Escalate topic into a real handoff, a user who fails three times now reaches a human instead of a dead end.

**Interruptions.** Leave allowed, so the user can type "start over" or ask a policy question mid-request and the agent switches out instead of forcing an answer. The Reset Conversation system topic handles "start over"; leave it on.

Save the topic again after these property changes.

## 5. Flow 1: Submit Leave Request

Build it as an agent flow from inside Copilot Studio so the trigger comes pre-wired: from the agent, go to Tools, Add a tool, New agent flow. The agent flow designer opens with the trigger "When an agent calls the flow" (Run a flow from Copilot) and "Respond to the agent" (Respond to Copilot) already on the canvas. Agent flows run under Copilot Studio licensing and can use premium connectors without separate Power Automate licensing. SharePoint and Outlook are standard connectors anyway, but this is worth a mention to the audience.

**Designer mechanics:** select the trigger card and use Add an input, Text to create each input. Outputs work the same way on the respond card. Add actions between the two cards with the + icon.

The Create item action writes to the Leave Requests list. The full column definitions and the exact field mapping are in [`leave-requests-list-schema.md`](./leave-requests-list-schema.md).

| #   | Action                       | Connector          | Configuration                                                                                                                                                                                                                                         |
| --- | ---------------------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | When an agent calls the flow | Trigger            | Add five Text inputs: EmployeeName, LeaveType, StartDate, EndDate, Reason. Name them properly; the agent uses these names to fill values from the conversation. Give each input a clear description so the agent places the right value in each slot. |
| 2   | Create item                  | SharePoint         | Site Address: your HR site. List Name: Leave Requests. Title = EmployeeName. LeaveType Value = LeaveType. StartDate = StartDate. EndDate = EndDate. Reason = Reason. Status Value = Pending.                                                          |
| 3   | Send an email (V2)           | Office 365 Outlook | To: hr@contoso.com. Subject: New leave request from + EmployeeName. Body: leave type, dates, reason, plus the Link to item dynamic value from step 2 so HR can open the record in one click. Optional but lands well live.                            |
| 4   | Respond to the agent         | Response           | Add one Text output RequestID = the ID dynamic value from step 2.                                                                                                                                                                                     |

Finish the flow:

- Open the settings of the Respond to the agent action and confirm Asynchronous Response is off. The flow must reply in real time.
- Rename the flow Submit Leave Request (select the flow name at the top), save, run Flow checker, then Publish.
- Use the agent name in the breadcrumb to return to the agent. The flow now also appears on the agent's Tools page; that is expected. This build calls it explicitly from the topic, which keeps the demo deterministic.

**Three requirements to state clearly in training:** synchronous response, published, and a reply within the 100 second limit or the tool call fails. Longer running actions can be placed after the respond action.

### Wire Flow 1 into the topic

1. Open the Apply for Leave topic. After question 5, select +, Add a tool, and pick Submit Leave Request (under Flow or Agent flows).
2. If the flow is not listed, check the three usual causes: missing or wrong trigger and respond actions, flow not published, or flow in a different environment.
3. The tool node lists the five inputs. Set each one to the matching topic variable using the `{x}` picker.
4. Under the node's outputs, save RequestID to a variable.
5. Add a Message node and insert the variable with the `{x}` control:

   > Done. Your leave request has been submitted, reference {RequestID}. HR has been notified.

Save the topic, refresh the Test pane, and run the full request once.

## 6. Human escalation (Teams card and email)

This is the path that fires whenever the knowledge source has no answer, when a user fails a question three times, or when someone asks for a person. It has two pieces: Flow 2, and a rebuilt Escalate system topic that calls it.

### 6.1 Flow 2: Escalate to HR

Create it the same way: Tools, Add a tool, New agent flow.

| #   | Action                            | Connector          | Configuration                                                                                                                                                                                                                                                                                                                                        |
| --- | --------------------------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | When an agent calls the flow      | Trigger            | Add two Text inputs: Question, ContactInfo.                                                                                                                                                                                                                                                                                                          |
| 2   | Compose (rename it EscalationRef) | Data Operations    | Inputs: `concat('ESC-', formatDateTime(utcNow(), 'yyyyMMddHHmmss'))`. Generates a reference like ESC-20260717093015.                                                                                                                                                                                                                                 |
| 3   | Post card in a chat or channel    | Microsoft Teams    | Post as: Flow bot. Post in: Channel. Team: HR. Channel: Leave Escalations. Adaptive Card: paste the JSON below, then replace the three placeholders with dynamic content (Question and ContactInfo from the trigger, Outputs from EscalationRef).                                                                                                    |
| 4   | Send an email (V2)                | Office 365 Outlook | To: hr@contoso.com. Subject: HR Leave Assistant escalation + Outputs of EscalationRef. Body: Question, ContactInfo, and the reference. Keep it even if you post the Teams card; email is the fallback if nobody watches the channel. If the Teams connector is blocked in your tenant, this becomes the only notification and the build still works. |
| 5   | Respond to the agent              | Response           | Add one Text output EscalationID = Outputs of EscalationRef.                                                                                                                                                                                                                                                                                         |

Adaptive card JSON for step 3:

```json
{
  "type": "AdaptiveCard",
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "version": "1.4",
  "body": [
    { "type": "TextBlock", "text": "HR Leave Assistant escalation", "weight": "Bolder", "size": "Medium" },
    { "type": "FactSet", "facts": [
      { "title": "Reference", "value": "<REFERENCE>" },
      { "title": "Contact", "value": "<CONTACT>" },
      { "title": "Question", "value": "<QUESTION>" }
    ] },
    { "type": "TextBlock", "text": "Sent by the HR Leave Assistant. Reply to the employee directly.", "isSubtle": true, "wrap": true }
  ]
}
```

Confirm Asynchronous Response is off on the respond action, rename the flow Escalate to HR, save, Publish, and return to the agent.

### 6.2 Rebuild the Escalate system topic

Go to Topics, select the System tab, open Escalate. It fires when "talk to agent" phrases are matched, when Question nodes exhaust their retries, and when Fallback redirects to it. First delete the default nodes under the trigger (select each node, three dots, Delete), then add this sequence:

1. **Set a variable value** (must be the first node): in the Set variable field choose Create a new variable and rename it OriginalQuestion; in the To value field open the `{x}` picker, System tab, and choose Activity.Text. This captures the message that triggered escalation before any question below overwrites it.
2. **Message:** I can't resolve that from the leave policy, but I can pass it to the HR team.
3. **Question:** Want me to send this to HR so someone follows up? Identify: Multiple choice options: Yes, No. Variable: SendToHR.
4. **No branch:** Message No problem. You can reach HR anytime at hr@contoso.com. then an End current topic node (Topic management).
5. **Yes branch, Question:** What's your name and an email HR can reply to? Identify: User's entire response. Variable: ContactInfo. (In Teams with authentication you would skip this question and map System.User.DisplayName and System.User.Email instead.)
6. **Add a tool:** Escalate to HR. Map Question = OriginalQuestion, ContactInfo = ContactInfo. Save the output to EscalationID.
7. **Message** (insert the variable with `{x}`): Done. I've sent this to HR, reference {EscalationID}. Someone will follow up within one working day.

Save the topic.

### 6.3 Point Fallback straight at Escalate

Still on the System tab, open Fallback. It fires when the agent cannot match the user's message to any topic or knowledge with confidence, which, with general knowledge and web search off, is every question the policy does not cover. Delete the default nodes under the trigger, then add two nodes:

1. **Message:** I couldn't find that in the leave policy.
2. **Topic management,** Go to another topic, and choose Escalate.

Save. Removing the default retry loop means a user is offered a human on the first miss instead of being told to rephrase three times. Combined with section 2.3, this completes the requirement: no internet answers, every miss reaches HR.

## 7. Remaining system topics

System topics cannot be deleted, but they can be turned off and their nodes can be edited. Beyond Fallback and Escalate (done above), modify these two and leave the rest at defaults.

### 7.1 Conversation Start

Fires when the conversation begins. Edit the message node text and save:

> Hi, I'm the HR Leave Assistant. I can answer leave policy questions or submit a leave request for you. If I can't answer something, I'll pass it to the HR team. Try: "How many annual leave days do I get?" or "I want to apply for leave".

### 7.2 On Error

Fires when an error occurs during the conversation, which is exactly what happens if a flow fails live. Replace the default message text and save:

> Sorry, something went wrong on my side and your request was not submitted. Please try again in a moment or email hr@contoso.com.

Diagnostic details still show to you in the Test pane, but attendees will not see an ugly error code.

### 7.3 Leave at defaults

End of Conversation, Multiple Topics Matched, Reset Conversation, and Sign in. End of Conversation's "didn't answer" path already redirects to Escalate, which is now a real handoff. Reset Conversation is what gives users "start over"; keep it on.

## 8. Test and publish

### 8.1 Test checklist

Run all of these in the Test pane before the session, refreshing between runs.

- **Happy path:** policy question, then "I want to apply for leave", answer all questions, confirm the SharePoint item, the email, and the reference number in chat.
- **Escalation path:** ask "Can I claim overtime?" Expect the Fallback message, the offer to contact HR, then after Yes and contact details, the adaptive card in the Leave Escalations channel, the email, and the ESC reference in chat.
- **Escalation declined:** same question, answer No. Expect a clean exit with the HR email address.
- **Grounding:** ask "Who is the CEO of Microsoft?" The agent must not answer it; it should route to Fallback and offer escalation. If it answers, general knowledge is still on (section 2.3).
- **Retry limit:** type gibberish three times at the LeaveType question. Expect two customised retries, then the escalation offer.
- **Cancel:** tap Cancel at the LeaveType question. Expect the cancel message and a clean exit.
- **Reset:** type "start over" in the middle of a request. Expect the conversation to reset.
- **Error handling:** temporarily unpublish Flow 1 and run the topic once. Expect your On Error message, not an error code. Republish afterwards.
- **Typed answers:** answer every question by typing, not just tapping buttons. There are known reports of question nodes re-asking despite valid typed answers under generative orchestration in some entity configurations, so verify each question accepts typed input cleanly.

### 8.2 Publish and share

- Select Publish (top right of the agent) and confirm. The Test pane works from saved drafts, but channels only serve the last published version, so re-publish after any later edit.
- Go to Channels, select Demo website, and copy the URL. That link is what attendees open during the session.
- Mention Teams as the realistic production deployment, where SharePoint knowledge and System.User variables also work, since users are authenticated.
- First run of each flow prompts for connection consent. Do this before the session, not on stage.

## 9. Live demo script

1. Open the agent. The custom greeting sets expectations (Conversation Start).
2. "How many days of annual leave am I entitled to?" (knowledge)
3. "Can I claim overtime?" then Yes to the escalation offer, give contact details, and flip to Teams to show the adaptive card land in the Leave Escalations channel (grounding plus human handoff, usually the biggest reaction of the session).
4. "I want to apply for leave", answer the questions (topic).
5. Flip to the SharePoint list to show the new item, show the HR email, point at the reference number in chat (flow).
6. Start a second request and type gibberish at the leave type question, then tap Cancel (question fallbacks and the escape path).
7. Close with the one-liner: knowledge answers, topics structure, tools act, HR catches the rest.

## 10. Gotchas

- Knowledge indexing takes minutes. Upload before the session.
- Both flows must be published, synchronous, and respond within 100 seconds, and must sit in the same environment as the agent.
- Flow not appearing in the tool list: wrong trigger or respond actions, unpublished, or different environment.
- Tenant DLP policies can block the SharePoint, Outlook, or Teams connectors. Verify in your demo tenant early; if Teams is blocked, the email in Flow 2 keeps escalation working.
- The Teams card posts as the Flow bot, so the HR team and Leave Escalations channel must exist and you need permission to post there.
- The Set variable node capturing System.Activity.Text must be the first node in the Escalate topic, or the Yes/No answer overwrites the original question.
- When escalation is triggered by a failed question rather than Fallback, OriginalQuestion holds the user's last invalid answer instead of a question. Acceptable for the demo; HR still gets the contact details.
- SharePoint knowledge returns nothing on an unauthenticated demo website, and System.User variables are blank there too. That is why this build uses an uploaded file and asks for contact details.
- Text date columns in the demo list avoid live date parsing failures. Production uses date columns with formatDateTime in the flow. See [`leave-requests-list-schema.md`](./leave-requests-list-schema.md).
- Keep a finished backup agent published in the same tenant.
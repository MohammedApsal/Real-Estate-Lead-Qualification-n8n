Enterprise Real Estate Lead Qualification Engine

An automated real estate lead qualification and routing system built with n8n + PostgreSQL + WhatsApp + Meta Lead Ads + Slack.

The system captures new leads, removes duplicates, starts a structured WhatsApp qualification conversation, collects buyer requirements, applies deterministic qualification rules, and routes qualified leads to the appropriate sales process.

📌 Business Problem

Real estate companies can generate hundreds of leads through Meta Ads, but not every lead is worth an immediate sales call.

Sales representatives often spend significant time manually contacting leads to find out:

What the buyer is looking for
Whether they are investing or planning to build
Their approximate budget
Purchase timeline
Whether the project actually fits their requirements

This creates a simple operational problem:

Sales teams spend valuable time qualifying leads instead of speaking with the buyers who are ready to move forward.

This project automates the first layer of that qualification process.

💡 Solution

The system starts communicating with a new lead shortly after the Meta Lead Ad submission.

Meta Lead Ad
     ↓
Lead Capture
     ↓
Normalize Lead
     ↓
Duplicate Check
     ↓
Create Conversation
     ↓
WhatsApp Outreach
     ↓
Purpose
     ↓
Budget
     ↓
Timeline
     ↓
Project Fit
     ↓
Qualification Engine
     ↓
HOT / WARM / NOT A FIT
     ↓
Sales Routing / Nurture / Close

The workflow is organized into three main lanes:

Lane A: Meta lead capture, deduplication, E-Brochure and initial purpose delivery.

Lane B: WhatsApp verification, inbound message processing and multi-stage qualification.

Lane C: Relationship-manager assignment, CRM-oriented state handling, delivery tracking and recovery dispatching.

🏗️ Architecture
Lane A — Meta Lead Capture

A Meta Lead Ads webhook receives incoming leads.

Flow
Meta Lead Ads
      ↓
Normalize Lead Record
      ↓
Validate Phone
      ↓
Check Duplicate
      ↓
Eligible?
      ↓
Create Contact
      ↓
Create Conversation
      ↓
Queue WhatsApp Outreach

The workflow includes separate GET and POST webhook endpoints for the Meta integration and processes the lead after acknowledging the inbound request.

Lead normalization includes:

leadgen_id
form_id
ad_id
name
phone_e164
phone_valid
project_name
🔁 Lead Deduplication

Before starting qualification, the system checks whether the lead already exists.

A lead continues through the qualification flow only when:

Phone Number = Valid
AND
Lead = Not Duplicate

This prevents the same customer from unnecessarily entering the qualification process multiple times.

📲 WhatsApp Qualification

Once a valid lead is created, the system creates a conversation and queues the initial WhatsApp outreach.

The WhatsApp conversation uses structured questions and interactive responses.

Example
Hi Rahul 👋

Thank you for enquiring about Grand Crest Villa Plots.

Are you looking to:

[ Invest ]
[ Build Immediately ]
[ Just Exploring ]

The workflow supports text replies as well as interactive button/list responses.

🧩 Qualification Stages

The conversation progresses through defined stages.

1. Purpose
Invest
Build Immediately
Explore
2. Budget
₹30L – ₹50L
₹50L – ₹75L
₹75L+
3. Timeline
Within 30 Days
Within 90 Days
Later / Exploring
4. Project Fit
Yes, Matches
No, Need Other

The system maintains the current qualification step in PostgreSQL and matches incoming button/text responses to the expected stage.

🧠 Deterministic Qualification Engine

The final qualification decision is handled through explicit business rules rather than relying entirely on an AI model.

Example rule structure:

HOT
Budget = ₹75L+
AND
Timeline = Immediate / Within 3 Months
AND
Project Fit = Yes

WARM
Does not meet HOT criteria
but remains potentially relevant

COLD / NOT A FIT
Budget below target
OR
Long-term timeline
OR
Project does not fit

The workflow's deterministic gate evaluates the collected answers and produces an outcome.

🔥 HOT Lead Routing

When a lead satisfies the HOT criteria, the system:

HOT Lead
   ↓
Assign Relationship Manager
   ↓
Pause Automation
   ↓
Update Lead State
   ↓
WhatsApp Confirmation
   ↓
Sales Team Alert

The RM assignment uses an active roster and selects the relationship manager based on the stored assignment timestamp.

A Slack alert is also generated for the senior sales team.

🟡 WARM Lead

WARM leads are not immediately handed off as HOT leads.

Instead, the conversation moves into a nurture state and schedules the next action for later.

The current workflow sets a 3-day follow-up window for WARM leads.

WARM
 ↓
WARM_NURTURE
 ↓
Future Follow-Up
⚪ NOT A FIT

When a lead falls outside the defined qualification criteria, the system closes the qualification flow politely.

NOT A FIT
   ↓
CLOSED
   ↓
Polite WhatsApp Message

🛑 Human Handoff

The buyer is not forced to remain inside the automation.

The system recognizes requests such as:

human
agent
person
call

or the request_human button.

When detected:

Automation Paused
      ↓
HUMAN_HANDOFF
      ↓
Staff Alert

🚫 Opt-Out Handling

The workflow also recognizes opt-out requests such as:

STOP
UNSUBSCRIBE
OPT-OUT

The conversation is moved into an opted-out state and pending automated actions are cancelled.

💬 Interactive WhatsApp Messaging

Outbound messages are generated dynamically based on the current conversation state.

The workflow supports:

Interactive messages
Text messages
Quick-reply style buttons

For example, the initial outreach contains an E-Brochure and project question, while subsequent messages dynamically present budget, timeline and project-fit questions.

⚙️ Recovery Dispatcher

Outbound actions are stored as pending actions rather than being tied directly to the qualification logic.

A dispatcher runs every minute:

Every 1 Minute
      ↓
Find Pending Actions
      ↓
Claim Actions
      ↓
Fetch Recipient
      ↓
Route Action
      ↓
WhatsApp / Staff Alert
      ↓
Mark Completed

The dispatcher uses PostgreSQL row locking with:

FOR UPDATE SKIP LOCKED

to claim pending work.

🗄️ PostgreSQL Data Layer

The workflow stores operational state in PostgreSQL.

Key entities include:

contacts
conversations
qualification_answers
pending_actions
rm_roster
handoffs

The conversation state and qualification answers are persisted so the system knows which question the buyer is currently answering.

🔐 Reliability Patterns
Idempotent Lead Processing

Duplicate leads are checked before creating a new qualification conversation.

Serialized Conversation Updates

Qualification answers are written while locking the conversation row:

FOR UPDATE

This helps serialize concurrent updates to the same conversation.

Persistent Pending Actions

WhatsApp messages and staff alerts are represented as database actions instead of being held only in workflow memory.

Human Override

Sales staff can take over the conversation at any point through the human-handoff path.

🧰 Tech Stack
Technology	Role
n8n	Workflow orchestration
PostgreSQL	Lead, conversation and workflow state
Meta Lead Ads	Lead acquisition
WhatsApp	Buyer communication
Slack	Sales alerts
Webhooks	Integration layer
JavaScript	Data transformation and qualification logic
📂 Suggested Repository Structure
real-estate-lead-qualification-engine/
│
├── workflows/
│   └── enterprise-lead-qualification.json
│
├── database/
│   ├── schema.sql
│   └── seed.sql
│
├── docs/
│   ├── architecture.md
│   ├── setup.md
│   └── qualification-rules.md
│
├── screenshots/
│   ├── meta-lead-capture.png
│   ├── whatsapp-qualification.png
│   ├── hot-lead-routing.png
│   └── dispatcher.png
│
├── examples/
│   └── sample-lead.json
│
├── .env.example
├── .gitignore
├── LICENSE
└── README.md
🚀 Setup
1. Install n8n

Run n8n locally or on your preferred server.

n8n start
2. Configure PostgreSQL

Create the required database/schema and configure the PostgreSQL credential inside n8n.

3. Configure Meta Lead Ads

Configure:

Meta Lead Ads Webhook
Verification
Lead event delivery
4. Configure WhatsApp

Connect your WhatsApp Business API credentials and configure the required phone-number ID.

The workflow currently uses a placeholder:

YOUR_WHATSAPP_PHONE_NUMBER_ID

so this needs to be replaced during deployment.

5. Configure Slack

Replace:

YOUR_SLACK_SALES_CHANNEL_ID

with the appropriate sales channel.

6. Import the n8n Workflow

Import the JSON workflow into n8n and configure the required credentials and environment-specific values.

🧪 Example Lead
{
  "name": "Rahul Kumar",
  "phone": "9876543210",
  "project_name": "Grand Crest Villa Plots",
  "source": "Meta Lead Ads"
}

The system normalizes the phone number into E.164 format before continuing through the qualification process.

📊 Example Customer Journey
Meta Ad
   ↓
"Interested Buyer"
   ↓
WhatsApp Outreach
   ↓
Purpose
   ↓
Budget
   ↓
Timeline
   ↓
Project Fit
   ↓
Qualification
   ├── 🔥 HOT → Senior RM
   ├── 🟡 WARM → Nurture
   └── ⚪ NOT A FIT → Close
🎯 Business Outcome

The purpose of this system is to move the sales process from:

“Call every lead and find out who is serious.”

to:

“Let the system collect the basic qualification information first, then help the sales team focus on the appropriate next action.”

The automation handles lead intake, structured qualification, state management, routing and outbound communication, while the sales team remains responsible for the actual human sales conversation.

⚠️ Deployment Notes

This repository represents the workflow implementation and its configured architecture. Before production deployment, environment-specific integrations such as Meta, WhatsApp, PostgreSQL and Slack must be configured and tested.

The workflow also contains project-specific example content, including the Grand Crest Villa Plots project and example pricing/messages, which should be replaced with the client's actual project configuration before deployment.

👨‍💻 Author

Mohammed Apsal M.

AI Automation Specialist
n8n • AI Agents • Workflow Automation • Generative AI

⭐ Project Summary

Enterprise Real Estate Lead Qualification Engine

Capture the lead.
Start the conversation.
Qualify the requirement.
Route the next action.

Built with n8n + PostgreSQL + Meta Lead Ads + WhatsApp + Slack.

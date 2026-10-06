---
status: research
source: nrtur-docs/user-personas.md (June 2026)
---
> **Research, not a spec.** These personas were written in June 2026, before pricing v4. The prices and budgets quoted below are what the personas pay *today for other tools*; they are not nrtur prices. For nrtur prices see [../pricing/pricing.md](../pricing/pricing.md).

# Nrtur User Personas

**Status**: Phase 2 - Product Definition  
**Last Updated**: May 13, 2026  
**Purpose**: Define our target users and shape product decisions

---

## Primary Persona: The Solopreneur Founder

**Name**: Sarah Chen  
**Age**: 32  
**Title**: Founder / Owner  
**Business**: Virtual tax prep service (self-employed, 1 person)  
**Revenue**: $120K/year  
**Tech comfort**: Moderate (comfortable with Zapier, basic tech setup)

### Profile

Sarah runs a solo tax prep service for small business owners. She works alone, manages her own marketing, sales, and service delivery. Her workday is split:
- 40% client work (preparing taxes, answering questions)
- 30% sales (finding new clients, following up on leads)
- 20% admin (scheduling, invoicing, record-keeping)
- 10% marketing (LinkedIn posts, email outreach)

### Current Tech Stack

- **CRM**: Google Sheets (100 rows of contacts, manually updated)
- **Scheduling**: Calendly (free tier, 2-week booking window)
- **Communication**: Gmail + personal iPhone (no SMS/MMS capability)
- **Automation**: Zapier free tier (one basic workflow)
- **Payments**: Stripe (for invoicing)
- **Marketing**: ConvertKit (email list)

**Pain**: She wants to send SMS reminders to clients ("Your return is ready"), but doesn't want to pay per-SMS on Twilio alone ($0.0075/SMS × 100 monthly reminders = pain).

### Frustrations

1. **Tool sprawl**: "I have Gmail, Calendly, Zapier, Stripe, spreadsheets. If I add a proper CRM, it's another tool I have to pay for."
2. **Cost**: "I can't justify $99/mo for HubSpot. My whole business revenue is $120K/year. A CRM would be 1% of my revenue. It feels expensive."
3. **Complexity**: "Salesforce and HubSpot feel like they're built for sales teams of 10+. I don't need 80% of their features."
4. **Time on admin**: "I spend 30 minutes every week manually updating my spreadsheet. If I could automate this, I'd get 2 hours back per month."
5. **SMS gap**: "I want to send SMS reminders, but every SMS platform requires separate setup and billing. It's fragmented."

### Needs

- Simple CRM (just contacts, pipeline, notes)
- SMS/email automation (send reminder when appointment is 1 day away)
- Affordable ($20-40/mo max)
- Works on mobile (she's often away from desk)
- Integrates with Zapier (she already uses Zapier)
- No long-term contract or setup fees

### Goals

- Cut admin time by 50% (save 1 hour/week)
- Send SMS reminders without separate tool (integrate with CRM)
- Track leads and conversion (know her sales funnel)
- Have one dashboard instead of 5 browser tabs

### Decision-Making Process

- Does research on G2, ProductHunt, Reddit
- Influenced by other solo founders (word-of-mouth)
- Wants free trial (no credit card preferred)
- Will pay if it saves time/money
- May cancel if doesn't see value in 1 month

### Success with Nrtur

Sarah would be successful if:
- She can import contacts from Google Sheets in <5 minutes
- She can send SMS to 50 contacts in <10 minutes (no setup friction)
- She can create a simple automation (send SMS 1 day before appointment) in <10 minutes
- She can see her sales pipeline (3 leads, 2 qualified, 1 closing) in one dashboard
- Monthly cost is < $50/mo

---

## Secondary Persona: The Service Agency Founder

**Name**: James Rodriguez  
**Age**: 38  
**Title**: Founder / Sales Manager  
**Business**: Digital marketing agency (5 people total: 1 founder, 2 salespeople, 2 service providers)  
**Revenue**: $500K/year  
**Tech comfort**: High (one of the sales people "owns tech")

### Profile

James runs a digital marketing agency. He's split between:
- 30% sales (closing new clients)
- 20% team management (managing 2 salespeople, 2 service people)
- 30% client work (overseeing projects)
- 20% operations (payroll, contracts, metrics)

His two salespeople need to track leads, send follow-ups, and report progress. He needs visibility into the pipeline and team activity.

### Current Tech Stack

- **CRM**: Pipedrive ($49/mo for 5 users)
- **SMS**: Close.io ($99/mo)
- **Email**: Gmail
- **Scheduling**: Calendly
- **Automation**: Zapier ($20/mo)
- **Project Management**: Asana ($480/year for team)
- **Analytics**: Google Sheets (manual reporting)

**Pain**: He's paying $49 + $99 = $148/mo for two separate tools (Pipedrive for CRM, Close for SMS). Pipedrive doesn't have SMS, so salespeople have to context-switch to Close for SMS. Close has weak CRM features, so he still needs Pipedrive for deal tracking.

### Frustrations

1. **Tool switching**: "My salespeople use Pipedrive for deals, then switch to Close for SMS. It breaks their flow and they miss follow-ups."
2. **Cost**: "I'm paying for two tools that should be one. $148/mo × 12 = $1,776/year. For a 5-person team, that's not cheap."
3. **Fragmented reporting**: "I can't see SMS and deal activity in one place. I can't tell if Sarah's SMS follow-up led to the deal or not."
4. **Limited automation**: "Close has SMS sequences, but they're not flexible. I can't build a custom workflow (SMS → check if replied → send email if no reply)."
5. **Team slowness**: "Pipedrive is simple, but adding SMS means they don't want to move to Close full-time. It's a half-solution."

### Needs

- One CRM with native SMS/email
- SMS sequences (send 3 follow-ups, spaced out)
- Deal tracking (pipeline, forecasting)
- Team reporting (see each salesperson's activity)
- Affordable for small team (<$30/person/month)
- Integrates with Zapier (they already use Zapier)

### Goals

- Consolidate Pipedrive + Close into one tool (simplify team workflow)
- Reduce tool cost by 20-30%
- Improve salesperson productivity (fewer context switches)
- Better sales visibility (what's the real pipeline, accounting for follow-ups)
- Faster onboarding of new salespeople (fewer tools to learn)

### Decision-Making Process

- Researches on G2, Capterra, Gartner
- Influenced by peer reviews and use-case fit
- Wants demo from sales engineer
- Needs dedicated support (not self-serve)
- May negotiate enterprise pricing if team scales to 10+ people

### Success with Nrtur

James would be successful if:
- Salespeople can move deals AND send SMS from one app (no switching)
- SMS sequences are flexible (not locked into presets)
- He can see team activity dashboard (each rep's SMS sent, deals moved, conversion)
- Monthly cost is < $150/mo total (vs $148 for Pipedrive + Close)
- Zapier integration works (they have custom workflows already)

---

## Tertiary Persona: The Real Estate Agent

**Name**: Patricia Martinez  
**Age**: 45  
**Title**: Real Estate Agent / Team Lead  
**Business**: Real estate brokerage (1 broker, 8 agents)  
**Revenue**: $1.2M/year  
**Tech comfort**: Low (wants plug-and-play, not setup)

### Profile

Patricia is a top-performing real estate agent who manages a small team. Her day:
- 40% client work (showings, inspections, negotiations)
- 30% lead follow-up (SMS, email, calls to past leads)
- 20% team management (coaching agents, checking metrics)
- 10% marketing (open houses, listing coordination)

She needs a system to track leads, follow up consistently, and measure agent performance.

### Current Tech Stack

- **CRM**: Zillow CRM (provided by broker, free)
- **SMS**: Personal phone (sending texts manually)
- **Email**: Gmail (no templates, no sequences)
- **Scheduling**: Google Calendar
- **Marketing**: Facebook ads (no integration with CRM)

**Pain**: The broker's CRM (Zillow) is outdated, doesn't have SMS, doesn't have sequences. She's following up manually, which is inefficient.

### Frustrations

1. **No SMS integration**: "Real estate is all about follow-up. I need to send SMS ('New listing in your area') automatically."
2. **Manual follow-up**: "I manually SMS leads every week. If I could automate this, I'd convert 10% more leads."
3. **No team visibility**: "I don't know if my agents are following up with their leads. I can't see who's sending what."
4. **No sequences**: "I want to send a sequence to a lead (SMS → email → SMS → call) over 30 days. I can't do this in Zillow."
5. **Slow admin**: "I spend 2 hours/week updating CRM. It's all manual."

### Needs

- SMS automation (send SMS to leads on schedule)
- Email sequences (multi-step, spaced out)
- Team visibility (see each agent's follow-up activity)
- Simple setup (no technical knowledge required)
- Affordable (real estate margins are thin)
- Integration with broker systems (if possible)

### Goals

- Automate lead follow-up (SMS + email sequences)
- Increase conversion rate by 15% (from 5% to 5.75%)
- Reduce time on admin/follow-up by 50%
- Measure agent productivity (activity per agent)
- Scale follow-up system to all 8 agents

### Decision-Making Process

- Influenced by other agents (word-of-mouth, broker recommendations)
- Wants quick setup (no complex onboarding)
- May need broker approval if integrating with their systems
- Price-sensitive (whole team cost < $200/mo)
- Wants customer support (for troubleshooting)

### Success with Nrtur

Patricia would be successful if:
- She can set up SMS/email sequence in <20 minutes
- Sequences are simple (no branching logic needed)
- She can see team activity dashboard (each agent's messages sent, leads followed up)
- Bulk import works (load 500 past leads from spreadsheet)
- Monthly cost is < $200/mo for all 8 agents

---

## User Roles & Responsibilities

### Within a Solopreneur Team (1 person)

**User Role: Owner/Operator**
- Creates contacts and deals
- Sends SMS and emails
- Views reports
- Manages billing
- Uses automations

### Within a Small Team (3-5 people)

**User Role 1: Sales Rep**
- Creates/edits own contacts and deals
- Sends SMS/emails
- Sees own activity
- Uses automations (created by admin)
- Cannot see other reps' data (unless manager)

**User Role 2: Manager**
- Sees all contacts and deals
- Views team activity dashboard
- Creates templates and automations for team
- Cannot edit billing
- Can remove/add users

**User Role 3: Admin**
- Full access (all features)
- Manages team, billing, integrations
- Creates company-wide templates
- Sets policies (data retention, etc.)

---

## User Persona Segmentation

### By Business Type

| Type | Examples | Size | Budget | Pain Point | Key Need |
|------|----------|------|--------|-----------|----------|
| **Service** | Tax, consulting, coaching, design | 1-5 | $20-50/mo | Time on admin | Automation |
| **Sales** | Real estate, insurance, SaaS | 1-10 | $50-200/mo | Lead follow-up | SMS sequences |
| **Agency** | Digital marketing, creative, PR | 2-10 | $100-300/mo | Tool consolidation | Multi-user, SMS native |
| **E-Commerce** | Online store, e-courses | 1-5 | $30-100/mo | Customer communication | SMS + email native |
| **Non-Profit** | Fundraising, outreach | 2-10 | $20-100/mo | Volunteer coordination | Simple, low-cost |

### By Buyer Persona

| Persona | Title | Focus | Key Concern | Buying Signal |
|---------|-------|-------|-------------|---------------|
| **Solopreneur** | Owner/Founder | Simplicity, cost, time-saving | "$99/mo is too much" | Wants free trial |
| **Manager** | Sales Manager, Operations | Team productivity, reporting | "Need to see team activity" | Wants team reporting |
| **Technical** | Ops person, automation person | Integrations, customization | "Does it work with Zapier?" | Wants API docs |
| **Finance** | CFO, Finance team | ROI, cost control | "What's the actual cost?" | Wants transparent billing |

---

## User Journey Map

### Solopreneur Journey (Typical)

**Day 1: Discovery**
- Finds Nrtur on ProductHunt / Google Ads
- Reads comparison (Nrtur vs HubSpot vs Pipedrive)
- Signs up for free trial

**Day 2-3: Onboarding**
- Imports contacts from Google Sheets
- Sends first SMS (test message)
- Creates first deal

**Week 1: Validation**
- Sends SMS sequence to 5 leads
- Sees if it leads to conversions
- Tracks time saved (admin time down?)

**Week 2-4: Adoption**
- Uses more (added 20 more contacts)
- Created 3 automations
- Sees value (saves 1 hour/week)
- Decides to pay

**Month 2+: Retention**
- Uses regularly (5-10 SMS/week, 10-20 emails/week)
- Invites to share feature feedback
- May upgrade if needs more contacts

### Manager Journey (Typical)

**Week 1: Evaluation**
- Hears about Nrtur from peer
- Requests demo
- Evaluates vs Pipedrive + Close

**Week 2: Trial**
- Adds 2 salespeople to trial
- Imports lead list
- Salespeople send SMS and move deals

**Week 3: Validation**
- Sees team activity dashboard
- Measures productivity gain (fewer context switches)
- Calculates cost savings (Pipedrive + Close vs Nrtur)

**Week 4: Conversion**
- Approves team switch
- Negotiates annual plan (discount for commitment)
- Commits to using Zapier for automations

**Month 2+: Retention**
- Team fully adopted
- Using 10+ automations
- Integrated with Calendly and Stripe

---

## Areas Requiring Your Feedback

1. **Primary persona**: Is the solopreneur the right primary target? Or should it be the small team/manager?
2. **Real estate focus**: Patricia (real estate) is a strong use case. Should we create vertical-specific features (open house reminders, listing notifications) or keep horizontal?
3. **User roles**: Are the role definitions (Rep/Manager/Admin) granular enough? Or should we add more (e.g., "Read-only" for finance review)?
4. **Budget assumptions**: Are the budget ranges accurate for your market research? (Are solopreneurs really willing to pay $30-50/mo for CRM?)
5. **Churn risk**: Which persona is most at risk for churn? (I think solopreneurs churn if they don't see ROI in 30 days.)

---

**Status**: Awaiting founder feedback.

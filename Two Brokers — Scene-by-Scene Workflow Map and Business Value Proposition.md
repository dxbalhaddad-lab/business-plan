# Two Brokers: Scene-by-Scene Workflow Map and Business Value Proposition

## Scene-by-Scene Mapping

### Scene 1 — Frank walks to his car, unaware

The film opens on a man who has already lost. Frank does not know it. He had a good day. He did everything right. The camera holds on him as he walks away — and the narrator tells us the deal is already gone.

**What the client experienced:** Nothing yet. This is the frame.

**What our system would have changed:** Everything that follows. Frank's loss was not caused by a single failure — it was caused by the absence of a system that operates in the intervals between his good work. The Inspector and The Buddy exist to fill those intervals. Frank is not the villain of this story. He is the portrait of a competent broker operating without the infrastructure that would have made him unbeatable.

---

### Scene 2 — James sends the same enquiry to two brokers

James, a father of five, is buying a villa. He sends the same enquiry to Frank and Gena on the same day. The enquiry arrives through a portal — Property Finder or Bayut — as a name, a phone number, a listing reference, and a short message.

**What the client experienced:** A simple message sent into the void. Now he waits.

**Buddy workflow:**

| Step | Agent | Action | Tool |
|------|-------|--------|------|
| 1 | System | Inbound lead/message ingestion adapter receives the enquiry from the portal | Portal webhook or CRM lead capture |
| 2 | Buddy | Triage Skill classifies the enquiry — is this Tier 1 (simple: availability, viewing request) or Tier 2 (substantive: property details, pricing, transaction)? | LLM classification |
| 3 | Buddy | CRM Lead Management Skill creates the lead record in the CRM: name, phone, listing reference, enquiry message, source channel, timestamp | CRM Adapter: `createLead(payload)` |

**Inspector workflow (parallel, background):**

| Step | Agent | Action | Tool |
|------|-------|--------|------|
| 4 | Inspector | Receives lead identity data from the shared context store | Event trigger from lead creation |
| 5 | Inspector | Runs Google Search Scraper to find the lead's LinkedIn URL, news, employer | `apify/google-search-scraper` via Apify MCP |
| 6 | Inspector | Runs LinkedIn Scraper to extract profile: headline, title, company, experience, education, languages | `get-leads/linkedin-scraper` via Apify MCP |
| 7 | Inspector | Runs LinkedIn Company Details for employer context: size, industry, website, specialties | `harvestapi/linkedin-company` via Apify MCP |
| 8 | Inspector | Runs Website Content Crawler on employer About/Team page for role context | `apify/website-content-crawler` via Apify MCP |
| 9 | Inspector | LLM synthesizes all collected data into structured persona JSON with confidence scores, source URLs, buyer-type signal, and personalization hooks | LLM synthesis |

**The gap this closes:** Frank received the enquiry and added it to his queue. He would respond when he reached it. Gena's system was already working — the Buddy had created the lead, classified the inquiry, and the Inspector was building a picture of James in the background. By the time Gena picked up her phone, she already knew who James was.

---

### Scene 3 — Frank replies within 15 minutes

Frank does everything right. He replies within fifteen minutes, confirms the viewing, shows up prepared, knows the market cold, and walks James through the numbers with total confidence. There is nothing to fault in Frank's work.

**What the client experienced:** A competent broker who responded in reasonable time. Fifteen minutes is not slow by any traditional measure.

**What our system does differently:** The Buddy's Tier 1 autonomy responds in under a minute — not fifteen. The difference between 15 minutes and 50 seconds is not a marginal improvement. It is the difference between "he replied quickly" and "she was already there." The Oldroyd/MIT/InsideSales study (verified in our research) showed that qualification odds drop 21x between 5 and 30 minutes. Frank's 15-minute response is already deep in the decline curve.

**Buddy workflow — Tier 1 autonomous response:**

| Step | Agent | Action | Tool |
|------|-------|--------|------|
| 1 | Buddy | Triage Skill classifies: Tier 1 (viewing request) | LLM classification |
| 2 | Buddy | CRM Adapter checks property availability: `checkAvailability(listing_id)` | CRM Adapter |
| 3 | Buddy | CRM Adapter checks broker calendar for open slots: `getBrokerCalendar(broker_id)` | CRM Adapter |
| 4 | Buddy | Sends pre-approved Tier 1 template: "Hi James, the villa is available. Gena has openings at 2 PM or 4 PM tomorrow — which works for you?" | WhatsApp Business API |
| 5 | Buddy | CRM Adapter creates viewing appointment once James confirms: `createViewingAppointment(payload)` | CRM Adapter |
| 6 | Buddy | Logs the full interaction in CRM with timestamp and provenance | CRM Adapter: `attachDocument(lead_id, log)` |

**The gap this closes:** Frank's 15-minute reply was the beginning of his work. Gena's 50-second reply was the end of the first interval — by the time Frank started, Gena had already acknowledged, confirmed availability, proposed a time, and booked the viewing. James's first impression was already set.

---

### Scene 4 — Gena replies in 50 seconds, calendar invite follows

Gena replies in fifty seconds. A calendar invite follows immediately.

**What the client experienced:** Speed. Not just a fast text — a calendar invite. That is a different signal. A fast reply says "I'm responsive." A calendar invite says "I'm organized, I've already booked this, and I expect to see you there."

**Buddy workflow:**

| Step | Agent | Action | Tool |
|------|-------|--------|------|
| 1 | Buddy | Tier 1 autonomous reply sent (as above) — pre-approved template, under 60 seconds | WhatsApp Business API |
| 2 | Buddy | Calendar Skill creates calendar entry for the confirmed viewing slot | CRM Adapter: `createViewingAppointment()` + calendar integration |
| 3 | Buddy | Sends calendar invite to James via WhatsApp or email with property address, time, and broker contact details | WhatsApp Business API / Email API |

**Technical note:** "50 seconds" and "calendar invite follows immediately" are film narrative targets, not guaranteed system SLAs. Actual response time depends on ingestion pipeline latency (portal webhook delivery, message parsing), which the verified research indicates ranges from 6–90 seconds depending on portal. The system is designed to respond as fast as ingestion allows — the target is sub-minute, not a hard guarantee.

**The gap this closes:** Frank confirmed the viewing in his 15-minute reply. Gena confirmed it and sent a calendar invite. The calendar invite is a small thing that most brokers never send. The Buddy sends it automatically because the Calendar Skill is part of the Tier 1 autonomous flow — it is not a separate task the broker remembers to do.

---

### Scene 5 — Personalized property file lands in 3 minutes

Three minutes after Gena's reply, a personalized property file lands in James's inbox — photos, floor plan, comparable sales — before he has asked a single question.

**What the client experienced:** "She already prepared this for me." James did not request a property file. He did not ask for comparable sales. He asked one question — "Is this available?" — and received a full information pack before he could type a follow-up.

**Buddy workflow — Tier 2 draft + broker approval:**

| Step | Agent | Action | Tool |
|------|-------|--------|------|
| 1 | Buddy | Property Card / CMA Skill assembles property data from CRM: photos, listing details, price | CRM Adapter: `getListing(listing_id)` / `generateOrFetchPropertyCard(listing_id)` |
| 2 | Buddy | Drafts property file with: listing photos, floor plan (if available in CRM), comparable sales (from Bayut Transaction Insights or DLD data, if integrated), area highlights | LLM drafting + CRM/portal data |
| 3 | Buddy | Inspector persona JSON is now available — Buddy uses buyer-type signal and personalization hooks to tailor the file (e.g., "villa buyer, relocation signal — include school proximity and community guide") | Persona JSON from shared context store |
| 4 | Buddy | Draft pushed to broker dashboard for approval | Approval Queue |
| 5 | Broker | Gena reviews the draft on her dashboard, edits if needed, approves | Dashboard UI — approval gate |
| 6 | Buddy | Approved property file sent to James via WhatsApp or email | WhatsApp Business API / Email API |

**What requires integration vs. what works now:**
- Photos and listing details: available from CRM — works now
- Floor plan: requires the CRM listing to have a floor plan field — integration-dependent
- Comparable sales: requires Bayut Transaction Insights or DLD sold data access — integration-dependent
- Personalization (school proximity, community guide): assembled from persona data + available area data — works now if area data is accessible

**The gap this closes:** Frank brought prepared materials to the viewing. Gena sent the materials before the viewing. The difference is not the quality of the materials — it is the timing. Frank's materials arrived when James arrived. Gena's materials arrived when James was still deciding whether to take the viewing seriously. The property file is not a response to a question; it is an anticipation of the next question.

---

### Scene 6 — Branded cost breakdown at the viewing

At the viewing, Gena has a branded cost breakdown ready to hand over.

**What the client experienced:** Professionalism. A branded document, not a handwritten note or a phone calculator. The branded cost breakdown signals that Gena operates at a different standard — she has tools, she has templates, she has a brand.

**Buddy workflow:**

| Step | Agent | Action | Tool |
|------|-------|--------|------|
| 1 | Buddy | Drafts cost breakdown using property data: purchase price, 4% DLD transfer fee, agency commission (2% buyer-paid for resale), trustee fees, DEWA setup, service charge estimates | LLM drafting + CRM data + verified fee structure from Property Finder Partner Hub |
| 2 | Buddy | Formats as branded document using brokerage branding template | Document generation with branding config |
| 3 | Buddy | Draft pushed to broker dashboard for approval | Approval Queue |
| 4 | Broker | Gena reviews, edits if needed, approves | Dashboard UI — approval gate |
| 5 | Buddy | Delivers branded document to Gena for handover at viewing, or sends to James via WhatsApp | Dashboard / WhatsApp Business API |

**The gap this closes:** Frank walked James through the numbers verbally, with confidence. Gena handed James a branded document. The information is the same; the delivery is different. The branded cost breakdown turns a conversation into an artifact — something James can take home, show his wife, and compare. The Buddy drafts it; Gena approves it; the client receives it. The broker owns the content; the system owns the preparation.

---

### Scene 7 — Wife raises concerns about moving and fit-out; Gena has vendor list ready

When James's wife raises concerns about moving and fit-out, Gena does not improvise an answer — she already has a curated list of vendors on her phone, ready to send.

**What the client experienced:** "She thought about things I hadn't even mentioned." This is the moment that wins the deal. Not the speed, not the property file, not the cost breakdown — the vendor list. Because the vendor list says something the other moments do not: "I was thinking about your life, not just your transaction."

**Buddy workflow:**

| Step | Agent | Action | Tool |
|------|-------|--------|------|
| 1 | Buddy | Proactively prepares a "villa move-in support pack" for any villa purchase — fit-out vendors, maintenance contacts, DEWA setup steps, community guide, school proximity data | LLM drafting + available vendor/community data |
| 2 | Buddy | If James's enquiry or WhatsApp conversation explicitly mentions family concerns, fit-out, or relocation, Buddy captures this as a first-party stated requirement and tailors the pack accordingly | First-party conversation data capture |
| 3 | Buddy | If Inspector persona shows relocation signal (career move, employer relocation, new to Dubai), Buddy broadens the pack to include area orientation, school lists, and community guides | Persona JSON buyer-type signal |
| 4 | Buddy | Draft pushed to broker dashboard for approval | Approval Queue |
| 5 | Broker | Gena reviews, approves | Dashboard UI — approval gate |
| 6 | Buddy | Vendor list delivered to Gena's phone or sent to James via WhatsApp | WhatsApp Business API / Dashboard |

**Critical boundary — how the system handles the "wife" element:**

The system does not infer family status, marital status, or household composition from public data. The Inspector's compliance guardrails explicitly exclude these as sensitive attributes. The system achieves the same outcome through a different path:

- James is buying a villa. That is a first-party fact. Villa buyers have fit-out needs. Buddy prepares a move-in support pack for villa purchases — not because it inferred a family, but because villa purchases involve fit-out.
- If James says in a WhatsApp message "my wife is concerned about the fit-out," Buddy captures that as a stated requirement and tailors the vendor list. This is the broker's own conversation data, volunteered by the client — not an inference.
- If the Inspector's persona shows a relocation signal (career move, employer relocation), Buddy broadens the pack to include area orientation and school proximity. This is based on professional context, not family-status inference.

The difference between Gena's human intuition and the system's approach: Gena guessed. The system prepares for the transaction type and captures what the client volunteers. Both arrive at the same moment — a vendor list ready before it is asked for — but the system's path is auditable, compliant, and scalable across every broker in the agency.

**The gap this closes:** Frank had nothing for the wife's concerns. He could have improvised — suggested a contractor, offered to send contacts later. But improvisation reads as improvisation. Gena did not improvise. She had a list. The difference between "let me find some vendors for you" and "here are the vendors" is the difference between a broker who reacts and a broker who has already been there.

---

### Scene 8 — James chooses Gena

James chooses Gena. He tells his friend Mike: "Frank had done the right job. There was nothing wrong with it. I just felt more taken care of with Gena."

**What the client experienced:** A feeling. Not a feature list. Not a price comparison. A feeling of being taken care of — built from the accumulation of small moments where Gena was already ahead.

**What the system delivered:** The system did not make Gena a better negotiator. It did not give her a better property or a better price. It made sure that at every interval between the big moments — between the enquiry and the viewing, between the viewing and the follow-up, between the question and the answer — Gena had something ready. The feeling James describes is not generated by any single feature. It is generated by the absence of gaps.

**System logging:**

| Step | Agent | Action | Tool |
|------|-------|--------|------|
| 1 | Buddy | Logs the full interaction history in CRM: every message sent, every draft approved, every viewing booked, persona attached, vendor list delivered, cost breakdown sent | CRM Adapter: `attachDocument(lead_id, full_interaction_log)` |
| 2 | Buddy | Updates lead status to "client engaged / viewing completed" | CRM Adapter: `updateLeadStatus(lead_id, status)` |
| 3 | Buddy | Sets follow-up reminder on broker calendar for post-viewing check-in | Calendar Skill |

**The gap this closes:** Frank will never know why he lost. There is no post-mortem he can run. He did nothing wrong. He lost because the intervals between his good work were empty — and he had no system to fill them. Gena's intervals were full because the Buddy handled them, and the Inspector gave her the context to personalize them.

---

### Scene 9 — Frank walks to his car, unaware

The final image: Frank, walking to his car after the viewing, completely unaware that he has lost the deal. No red flag. No mistake he can point to and fix. He simply never saw the moments that mattered, because from where he stood, they did not look like moments at all.

**What the client experienced:** Nothing. The client is already with Gena.

**What the system represents:** Frank is every broker who operates without the platform. He is competent, hardworking, and blind to the intervals. The system does not exist to make Frank a bad broker — it exists to make sure no broker on our platform ever walks to their car unaware. Every interval is covered. Every gap is filled. Every moment that a client might remember has already been handled by the Buddy, informed by the Inspector, and approved by the broker.

---

## Business-Plan Value Proposition

### The one-sentence pitch

We do not sell better brokers. We sell the moments between what brokers already do well — and we fill those moments with an agent system that anticipates, prepares, and delivers before the client asks.

### The problem

Every brokerage promises to take care of clients. The promise is identical across the market. Clients have stopped hearing it — not because it is dishonest, but because the experience behind the promise is the same everywhere: a broker who does the big things well but leaves the small things to chance.

The small things are what clients remember. The speed of the first reply. Whether the information arrived before it was requested. Whether someone had thought about the client's life beyond the transaction. These are not features of the property. They are features of the broker's operational system — and most brokers do not have one.

A broker without a system relies on memory, habit, and available time. They reply when they can. They prepare when they remember. They anticipate when they have the mental bandwidth. On a good day, they are Frank — competent, thorough, and unaware of the gaps. On a bad day, they are worse.

### The solution

Our platform deploys two AI agents alongside every broker:

**The Inspector** builds a digital picture of each new lead — role, employer, sector, career stage, buyer-type signals — from public web data, in the background, before the broker picks up the phone. It uses verified data-extraction tools (Apify Actors via the Apify MCP server) to enrich every lead with professional context and personalization hooks. The broker walks into every conversation already knowing who they are talking to.

**The Buddy** is the broker's Executive Assistant. It reads incoming WhatsApp and email inquiries, manages the broker's calendar, maintains the CRM, coordinates listing onboarding, and drafts personalized communications and property packs. For simple requests — availability checks, viewing confirmations — it acts autonomously using pre-approved templates. For everything substantive — property details, pricing, transaction information — it drafts and waits for broker approval. Nothing substantive reaches a client without the broker's explicit sign-off.

### The value — mapped to the film's moments

| Film moment | What brokers without the platform do | What brokers with the platform do | Value created |
|---|---|---|---|
| First reply | Respond in 10–30 minutes when they reach the lead in their queue | Respond in under a minute — Buddy sends a pre-approved acknowledgement using first-party portal data | The client's first impression is "she's already here" — not "he'll get back to me" |
| Calendar invite | Confirm the viewing verbally or in a follow-up message | Buddy books the viewing and sends a calendar invite automatically | The client receives organization, not just responsiveness |
| Property file | Bring materials to the viewing | Buddy assembles and sends a property file — photos, floor plan, comparable sales — before the viewing, using CRM data and Inspector persona | The client receives information before asking — the broker arrives having already delivered value |
| Cost breakdown | Walk through numbers verbally | Buddy drafts a branded cost breakdown; broker approves; client receives a take-home document | The client receives an artifact, not a conversation — something to show a spouse, compare, and revisit |
| Vendor / move-in pack | Improvise when the client raises concerns | Buddy proactively prepares a move-in support pack for the transaction type (villa purchase, relocation signal); broker approves; client receives it before raising the concern | The client feels anticipated — "she thought about things I hadn't mentioned" |
| Follow-up after viewing | Rely on the broker to remember to follow up | Buddy logs the full interaction, sets a follow-up reminder, and (Phase 2) the voice agent calls the client post-viewing | The client is never forgotten after the big moment passes |

### The competitive moat

The moat is not the property. It is not the price. It is not the broker's competence. The moat is the intervals — the moments between the big milestones that most brokerages do not think to measure and cannot systematically fill.

A competitor can match our response time by hiring more staff. They can match our property files by building their own templates. They can match our cost breakdowns by designing their own documents. But they cannot match the system's ability to operate in every interval, for every lead, simultaneously, without the broker's active attention — because that requires the integrated combination of The Inspector (background intelligence) and The Buddy (operational anticipation), connected to the CRM, the calendar, and the communication channels, running continuously.

Brokers without this system are Frank. They are good at their jobs. They lose deals they do not know they lost, for reasons they will never see. Our platform makes sure no broker on our side ever walks to their car unaware.

### The business model implication

This is not "AI replacing brokers." This is "AI operationalizing care at scale." The broker still owns the relationship, the advice, the negotiation, and the approval. The system owns the intervals — the preparation, the anticipation, the speed, and the consistency that turn a good broker into a broker the client feels taken care of by.

The platform does not change what the broker does. It changes what arrives before the broker does it. And that difference — the difference between arriving prepared and arriving empty — is the difference between Gena and Frank.

### The closing line

"Sometimes the difference isn't the property. And it isn't whether the broker can do the job. It's what the client experiences between one step and the next."

That line is not just the film's message. It is the platform's value proposition. We are the system that fills the space between one step and the next — and we fill it for every broker, for every lead, at every moment, without fail.

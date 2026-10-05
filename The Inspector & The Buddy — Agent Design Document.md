# The Inspector & The Buddy: Agent Design Document

## Preamble: Scope and Design Principles

This document defines two AI agents — The Inspector and The Buddy — that together form the operational intelligence layer for a Dubai real estate brokerage. The Inspector builds lead personas from public web data. The Buddy acts as an Executive Assistant to the human broker, absorbing administrative work across communications, calendar, CRM, and listings.

**Design principles:**

1. **Graduated autonomy.** The Buddy operates on three tiers: some scheduling and availability actions are autonomous (if policy-enabled), all substantive content requires broker approval, and unresponsive approvals escalate.
2. **Human-in-the-loop for substance.** Nothing substantive, advisory, transactional, pricing-related, or legal is ever sent to a client without explicit broker approval. Only pre-approved, low-risk scheduling and availability messages may be sent autonomously, if enabled by product policy.
3. **Vendor-neutral CRM.** The CRM layer is an abstraction (a CRM Adapter interface) that can be implemented against PropSpace, Ruby CRM, or any system that exposes the required capabilities.
4. **Grounded tools only.** Every named tool, Actor, and integration in this document is verified against the consolidated research. Items not fully verified by research are labeled as integration-dependent or as target value hypotheses.

---

# PART A — End-to-End Workflow

## A.1 Business Narrative

### The core idea

A real estate broker's highest-value work is building relationships, closing deals, and being present with clients. Everything else — typing property details, chasing documents, confirming viewing times, updating the CRM, preparing information packs — is necessary but steals time from revenue-generating activity. The Buddy and The Inspector exist to absorb that non-revenue work, phase by phase, so the broker focuses on what matters.

The Inspector is the research analyst. When a new lead arrives, it quietly builds a digital picture of that person — their role, employer, sector, career stage, and any public signals that help the broker personalize the conversation. It works in the background using verified data-extraction tools, and it never sends anything to a client.

The Buddy is the broker's Executive Assistant. It reads incoming WhatsApp and email inquiries, triages them, manages the broker's calendar, updates the CRM, coordinates listing onboarding, and drafts personalized communications. For simple requests — "Can I view the property tomorrow?" — the Buddy can confirm and lock the viewing directly using the broker's calendar and CRM availability. For anything substantive — property details, transaction information, pricing, legal questions — the Buddy prepares the material and waits for the broker to approve before anything reaches the client.

If the broker does not respond to a pending approval within the required time window, the system escalates to the admin team. The broker remains accountable, but the system ensures nothing falls through the cracks.

### Workflow 1: New lead inquiry (WhatsApp or email)

A lead sends a WhatsApp message asking about a property listed on Property Finder or Bayut. The message arrives in the system.

The Buddy reads the inquiry. If the lead is asking a simple question — "Is this property still available?" or "Can I schedule a viewing tomorrow?" — the Buddy checks the CRM for listing status and the broker's calendar for open slots. If the property is available and the calendar has space, the Buddy sends a pre-approved confirmation message and books the viewing. The broker is notified but does not need to intervene. This alone can save the broker the back-and-forth that typically consumes the first 10–15 minutes of a new lead interaction.

In parallel, The Inspector begins its background work. It takes the lead's name and any employer or sector clues from the inquiry, searches Google for a LinkedIn profile, runs the LinkedIn Scraper to extract role, employer, and career history, pulls company details, and optionally checks Instagram for public lifestyle signals. Within a few minutes, it delivers a structured persona to the broker's dashboard — the lead's professional context, inferred buyer type, and personalization hooks, each with a confidence score and a source URL.

If the lead's question is more substantive — "What are the payment plan options?" or "Is this a good investment area?" — the Buddy drafts a response using available property data and the Inspector's persona, then queues it for broker approval. The broker reviews the enriched persona and the drafted message on their dashboard, edits if needed, and approves. The message is sent, and the full interaction is logged in the CRM.

### Workflow 2: Client requests more information about a property or project

An existing client asks for detailed information about a specific property or off-plan project. The broker would traditionally need to sit down, gather brochures, payment plans, and area data, compose a response, and send it.

The Buddy takes this on. It assembles the requested information from the CRM, portal data, and any available property cards. It drafts a structured information pack — property details, payment plan, area highlights, and relevant market data — and presents it to the broker for review. The broker approves or edits, and the Buddy sends the approved pack via WhatsApp or email. This can save the broker over 60% of the time this task would normally take.

### Workflow 3: New listing onboarding

A broker secures a new listing from a property owner. Traditionally, this means coordinating with a photographer for the photoshoot, collecting the images and property documents, drafting the listing description, updating the CRM, and creating the portal listing — a multi-step process that can stretch across days.

The Buddy streamlines this. It schedules the photoshoot on the broker's calendar, coordinates with the photographer, receives the images and property documents, drafts the listing description from the provided data, populates the CRM record, and prepares the listing for portal publication. The broker reviews the drafted listing — description, photos, price, permit number — and approves it to go live. The Buddy handles the heavy lifting; the broker makes the final call.

### Phase 2 workflows (future scope)

- **AI Voice Agent:** Inbound and outbound follow-up calls — confirming appointments, checking in with dormant leads, reconfirming viewings. The voice agent coordinates with the Buddy's calendar and CRM to know who to call, why, and what to say.
- **Social Media Management:** The Buddy manages social media account activities — responding to comments and messages, scheduling posts from the content calendar, and drafting engagement responses. All posts and replies follow the approval and plan system.

---

## A.2 Technical Sequence Breakdown

### Workflow 1: New Lead Inquiry

| Step | Actor | Tool / System | Input | Output | Human Gate |
|------|-------|---------------|-------|--------|------------|
| 1 | System | Inbound lead/message ingestion adapter | Incoming WhatsApp or email message | Parsed message object (sender name, phone/email, body, listing ref, source channel, timestamp) | — |
| 2 | Buddy | Triage Skill (LLM) | Parsed message object | Classification: Tier 1 (simple/availability/viewing) or Tier 2 (substantive/detail/transaction/legal) | — |
| 3a (Tier 1) | Buddy | CRM Adapter: `checkAvailability(listing_id)` + `getBrokerCalendar(broker_id)` | Listing reference, broker ID | Availability status, open calendar slots | — |
| 3b (Tier 1) | Buddy | CRM Adapter: `createViewingAppointment(payload)` | Listing ID, client name, phone, selected time slot | Viewing appointment created in CRM; calendar blocked | — |
| 3c (Tier 1) | Buddy | WhatsApp Business API / Email API | Pre-approved template + appointment details | Confirmation message sent to lead | **No per-message approval** (policy-enabled; all actions logged for broker review) |
| 3d (Tier 1) | Buddy | CRM Adapter: `createLead(payload)` + `updateLeadStatus(lead_id, status)` | Lead data from message | Lead record created/updated in CRM | — |
| 4a (Tier 2) | Buddy | Drafting Skill (LLM) + CRM Adapter: `getListing(listing_id)` | Parsed message, listing data, persona JSON (if available) | Draft response with supporting data | — |
| 4b (Tier 2) | Buddy | Approval Queue | Draft response | Draft pushed to broker dashboard; push notification sent | **Broker approval required** |
| 4c (Tier 2) | System | Timer / Escalation | Draft timestamp | If no approval within configurable window (default: 30 min), escalate to admin team via notification | **Escalation gate** |
| 4d (Tier 2) | Broker | Dashboard UI | Draft response | Approved / edited / rejected response | **Approval gate** |
| 4e | Buddy | WhatsApp Business API / Email API | Approved message | Message sent to lead | — |
| 5 (parallel) | Inspector | Apify MCP: `apify/google-search-scraper` | Lead name + employer/sector clues | SERP results: LinkedIn URL, news, company page | — |
| 6 (parallel) | Inspector | Apify MCP: `get-leads/linkedin-scraper` (Search Profiles mode) | LinkedIn URL or name | LinkedIn profile data: headline, title, company, experience, education, skills, email (if found) | — |
| 7 (parallel) | Inspector | Apify MCP: `harvestapi/linkedin-company` | Company name/URL | Company details: size, industry, website, specialties, funding data | — |
| 8 (parallel) | Inspector | Apify MCP: `apify/website-content-crawler` | Employer website URL | About/Team page content for role context | — |
| 9 (optional, parallel) | Inspector | Apify MCP: `apify/instagram-scraper` | Instagram handle (if public and discovered) | Profile bio, follower count, recent posts (lifestyle/area signals) | — |
| 10 | Inspector | LLM synthesis | All collected data | Structured persona JSON (see Inspector outputs) with confidence scores and source URLs | — |
| 11 | Inspector | Shared context store | Persona JSON | Written to shared store; Buddy notified via event trigger | — |
| 12 | Buddy | CRM Adapter: `updateLead(lead_id, { persona: personaJSON })` | Persona JSON | Lead record enriched in CRM | — |
| 13 | Buddy | Dashboard UI | Persona JSON + any pending Tier 2 drafts | Enriched persona displayed on broker dashboard for review | **Broker reviews persona** (no blocking gate — persona informs future interactions) |
| 14 | Buddy | CRM Adapter: `attachDocument(lead_id, interaction_log)` | Full interaction log (messages sent, persona, approvals, timestamps) | Interaction logged in CRM with provenance | — |

### Workflow 2: Client Requests Property/Project Information

| Step | Actor | Tool / System | Input | Output | Human Gate |
|------|-------|---------------|-------|--------|------------|
| 1 | System | WhatsApp / Email inbound | Client message requesting property/project details | Parsed request object | — |
| 2 | Buddy | Triage Skill (LLM) | Parsed request | Classification: type of info requested (payment plan, area data, project status, comparison) | — |
| 3 | Buddy | CRM Adapter: `getListing(listing_id)` / `generateOrFetchPropertyCard(listing_id)` / `generateOrFetchCMA(criteria)` | Listing reference, project ID, or CMA criteria | Property card, CMA data, or listing details (integration-dependent) | — |
| 4 | Buddy | Drafting Skill (LLM) | Property data + persona JSON + client context | Structured information pack (property details, payment plan, area highlights, relevant market data) | — |
| 5 | Buddy | Approval Queue | Information pack draft | Draft pushed to broker dashboard; notification sent | **Broker approval required** |
| 6 | Broker | Dashboard UI | Information pack draft | Approved / edited / rejected | **Approval gate** |
| 7 | Buddy | WhatsApp Business API / Email API | Approved information pack | Sent to client | — |
| 8 | Buddy | CRM Adapter: `attachDocument(lead_id, interaction_log)` | Full interaction log | Logged in CRM | — |

### Workflow 3: New Listing Onboarding

| Step | Actor | Tool / System | Input | Output | Human Gate |
|------|-------|---------------|-------|--------|------------|
| 1 | Broker | Dashboard UI | New listing instruction (owner details, property address, asking price, Form A / marketing contract) | Listing onboarding task created | — |
| 2 | Buddy | Calendar Skill + CRM Adapter: `getBrokerCalendar(broker_id)` | Broker availability, photographer availability | Photoshoot scheduled on broker calendar; photographer notified | — |
| 3 | Buddy | Communication (WhatsApp/Email) | Photoshoot confirmation template | Photographer sent confirmation; broker notified | — |
| 4 | Photographer | External | Photoshoot completed | Images received by Buddy (via shared folder or upload) | — |
| 5 | Buddy | CRM Adapter: `updateListing(listing_id, { images, documents })` | Listing ID, images, property documents (Form A, Trakheesi permit) | Listing record updated with images and docs | — |
| 6 | Buddy | Drafting Skill (LLM) | Property data, images, area data | Draft listing description, title, and portal-ready content | — |
| 7 | Buddy | CRM Adapter: `updateListing(listing_id, { description, title, price, permit_number })` | Drafted listing content | Listing populated in CRM | — |
| 8 | Buddy | Approval Queue | Full draft listing (description, images, price, permit) | Pushed to broker dashboard for review | **Broker approval required** |
| 9 | Broker | Dashboard UI | Draft listing | Approved / edited / rejected | **Approval gate** |
| 10 | Buddy | CRM Adapter: portal syndication (if supported) | Approved listing | Listing published to portals (Property Finder, Bayut, Dubizzle) | — |

---

# PART B — Agent Profiles

## Agent 1: The Inspector

### Business Layer

**Role:** The Inspector is the research analyst that builds a digital picture of a new lead before the broker picks up the phone.

**Problem it solves:** When a new lead arrives, the broker has a name, a phone number, and a property interest. Everything else — the lead's profession, employer, seniority, sector, career trajectory, and whether they are an investor or an end-user — must be discovered manually by searching LinkedIn, Google, and Instagram. This takes 10–40 minutes per lead, if the broker does it at all. Most brokers skip it, which means their first conversation is generic and uninformed.

**Time and value saved:** The Inspector compresses this research into a background process that completes in 2–5 minutes. The broker receives a structured persona with professional context, employer details, inferred buyer type, and personalization hooks — each with a confidence score and a source URL. The broker walks into the first call already knowing the lead's background.

**Competitive value:** Brokers without this capability either send generic property lists (low conversion) or spend billable time on manual research (fewer leads handled per day). The Inspector enables every broker in the agency to operate as if they had a dedicated research assistant. The two-pass design — instant acknowledgement first, enrichment in the background — ensures the 5-minute response window is never compromised.

### Technical Layer

**Core responsibilities:**
- Receive lead identity data (name, phone, email, listing reference, enquiry message) from the shared context store
- Execute a sequential enrichment pipeline using verified Apify Actors via the Apify MCP server
- Synthesize collected data into a structured persona JSON with per-field confidence scores and source URLs
- Write the persona JSON to the shared context store and trigger The Buddy via event notification

**Explicit out-of-scope boundaries:**
- Does NOT send any client-facing communication
- Does NOT access private social media accounts or bypass platform permissions
- Does NOT infer sensitive attributes (nationality, religion, ethnicity, health, political views, family/marital status, wealth estimation) from public data, even if technically possible
- Does NOT perform reverse-phone matching (looking up a person from a phone number) — this is unreliable and raises data-protection concerns
- Does NOT interact with the CRM, calendar, or communication channels
- Does NOT make property recommendations or buyer-type classifications independently of the broker

**Required tools and MCP integrations:**

| Tool | Actor ID | Purpose | Pricing (verified) |
|------|----------|---------|---------------------|
| Google Search Results Scraper | `apify/google-search-scraper` | Discovery: find lead's LinkedIn URL, news, employer website | From $1.80 / 1,000 SERP pages ([Apify Store](https://apify.com/apify/google-search-scraper)) |
| All-in-One LinkedIn Scraper | `get-leads/linkedin-scraper` | Profile enrichment: name, headline, title, company, experience, education, skills, email (cookieless; optional li_at cookie for deeper fields) | From $3.40 / 1,000 profiles ([Apify Store](https://apify.com/get-leads/linkedin-scraper)) |
| LinkedIn Company Details Scraper | `harvestapi/linkedin-company` | Employer context: company size, industry, website, specialties, funding data | From $3.00 / 1,000 companies ([Apify Store](https://apify.com/harvestapi/linkedin-company)) |
| Website Content Crawler | `apify/website-content-crawler` | Read employer About/Team page for role context | ~$0.2–$5 / 1,000 pages ([Apify Store](https://apify.com/apify/website-content-crawler)) |
| Instagram Scraper (optional) | `apify/instagram-scraper` | Public lifestyle/area signals (only if account is public and relevant) | From $1.50 / 1,000 results ([Apify Store](https://apify.com/apify/instagram-scraper)) |

**MCP server configuration:**
- Hosted endpoint: `https://mcp.apify.com` ([Apify MCP docs](https://docs.apify.com/integrations/mcp))
- Allowlist: restrict to the five Actors above using the `tools` query parameter
- Discovery calls (`search-actors`, `fetch-actor-details`) are free
- Rate limit: 30 requests/second/user — implement retry logic for HTTP 429
- Set `maxTotalChargeUsd` on each Actor run to prevent unexpected charges
- Estimated cost per lead enrichment: $0.01–$0.05 (illustrative estimate, not an official Apify price)

**Data inputs:**

| Field | Source | Format | Required |
|-------|--------|--------|----------|
| Lead name | Portal lead / CRM | String (first + last) | Yes |
| Phone | Portal lead | String (E.164) | Yes |
| Email | Portal lead | String (email) | No |
| Listing reference | Portal lead | String | Yes |
| Enquiry message | Portal lead | String | Yes |
| Source channel | Portal lead | Enum (Property Finder / Bayut / Dubizzle / WhatsApp / Email) | Yes |
| Employer/sector clues | Enquiry message parsing | String (free text) | No |

**Data outputs — Persona JSON schema:**

```json
{
  "lead_id": "string",
  "enrichment_timestamp": "ISO 8601",
  "identity_confidence": {
    "name": "string",
    "likely_linkedin_url": "string | null",
    "confidence_score": "high | medium | low"
  },
  "professional_context": {
    "current_title": "string | null",
    "employer": "string | null",
    "sector": "string | null",
    "seniority": "string | null",
    "career_trajectory": "string | null"
  },
  "employer_context": {
    "company_name": "string | null",
    "company_size": "string | null",
    "industry": "string | null",
    "website": "string | null",
    "locations": "array | null",
    "recent_news": "string | null"
  },
  "buyer_type_signal": {
    "classification": "investor | end_user | tenant | relocation | unknown",
    "evidence": "string",
    "confidence": "high | medium | low"
  },
  "property_intent": {
    "listing_viewed": "string",
    "area": "string | null",
    "budget_if_stated": "string | null",
    "beds": "string | null",
    "timeline": "string | null"
  },
  "personalization_hooks": {
    "hook_1": "string (e.g., 'Relocating from London for fintech role — Downtown/Business Bay commute relevance')",
    "hook_2": "string | null",
    "hook_3": "string | null"
  },
  "source_provenance": [
    {
      "field": "string",
      "source_url": "string",
      "retrieval_timestamp": "ISO 8601",
      "confidence": "high | medium | low"
    }
  ],
  "overall_confidence": "high | medium | low",
  "category": "A (high footprint) | B (low footprint)",
  "notes": "string (e.g., 'Sparse public footprint; rely on first-party data')"
}
```

**Excluded fields (do-not-use unless explicitly volunteered and directly relevant):** religion, ethnicity, nationality inference, health, political views, family/marital status, wealth estimation.

**Fallback behavior — Category B (low-footprint) leads:**
- If Google Search returns no LinkedIn profile and no news/press coverage, the Inspector skips LinkedIn and Instagram enrichment
- Returns a persona JSON with `overall_confidence: "low"`, `category: "B"`, and a note: "Sparse public footprint; rely on first-party data"
- The lead's first-party portal data (name, phone, listing interest) becomes the primary persona source
- The Inspector does NOT attempt reverse-phone matching or data inference from sparse signals

**Handoff protocol (Inspector → Buddy):**
1. Inspector writes persona JSON to shared context store (keyed by `lead_id`)
2. Inspector emits an event: `persona.enriched.{lead_id}` with the persona JSON as payload
3. Buddy's event listener receives the event and triggers downstream actions (CRM update, dashboard notification, draft personalization if a Tier 2 response is pending)
4. If the Inspector fails or times out (after configurable max latency, default: 5 minutes), it emits `persona.enrichment_failed.{lead_id}` and Buddy proceeds with first-party data only

**Success metrics / KPIs:**

| KPI | Definition | Target |
|-----|------------|--------|
| Enrichment success rate | % of leads where at least one enrichment source returns data | > 70% for Category A leads |
| Average enrichment latency | Time from lead arrival to persona JSON delivery | < 5 minutes |
| Persona field coverage rate | % of persona fields populated with non-null values | > 60% for Category A; > 15% for Category B |
| Source URL provenance rate | % of populated fields with a source URL and timestamp | 100% |
| Category B detection accuracy | % of low-footprint leads correctly identified and flagged | > 80% |

**Example scenario — "Sarah Chen" lead:**

**Fictional example for design illustration.**

1. Lead arrives from Property Finder: name "Sarah Chen", phone +971-50-XXX-XXXX, listing reference "PF-12345", enquiry "Interested in 2BR in Business Bay, budget 2.5M AED"
2. Inspector receives lead data via shared context store
3. Inspector runs `apify/google-search-scraper` with query "Sarah Chen Dubai LinkedIn" — finds LinkedIn URL `linkedin.com/in/sarah-chen-fintech`
4. Inspector runs `get-leads/linkedin-scraper` in Search Profiles mode — extracts: headline "Senior Product Manager at Checkout.com", current title "Senior Product Manager", employer "Checkout.com", experience 8 years, education "MBA, INSEAD", languages "English, Mandarin"
5. Inspector runs `harvestapi/linkedin-company` for "Checkout.com" — extracts: company size 1,001–5,000, industry "Financial Services / Payments", website "checkout.com", locations "Dubai, London, Singapore"
6. Inspector runs `apify/website-content-crawler` on `checkout.com/about` — extracts company description for context
7. Instagram check: no public Instagram account found — skipped
8. Inspector LLM synthesizes persona JSON:
   - `buyer_type_signal`: "relocation" (evidence: MBA from INSEAD, fintech sector, 2BR Business Bay budget 2.5M — consistent with executive relocation)
   - `personalization_hooks`: ["MBA from INSEAD — may value proximity to DIFC/Business Bay for fintech network", "Senior PM at Checkout.com — Downtown/Business Bay commute relevance", "8 years experience — likely mid-career, budget aligns with senior professional profile"]
   - `overall_confidence`: "high"
   - `category`: "A"
9. Inspector writes persona JSON to shared context store, emits `persona.enriched.sarah-chen-12345`
10. Buddy receives event, updates CRM lead record with persona, displays enriched profile on broker dashboard

---

## Agent 2: The Buddy

### Business Layer

**Role:** The Buddy is the broker's Executive Assistant — it manages the calendar, triages communications, maintains the CRM, coordinates listing onboarding, and drafts personalized outreach, absorbing the administrative load so the broker focuses on clients and deals.

**Problem it solves:** A Dubai real estate broker's day is dominated by administrative tasks — responding to WhatsApp messages, confirming viewing times, updating the CRM, chasing photographers, preparing property information packs, and drafting listing descriptions. These tasks are necessary but they consume the hours that should go to client relationships, negotiations, and closing deals. The Buddy takes these tasks on, phase by phase, with a simple rule: simple scheduling and availability actions are handled autonomously; everything substantive is drafted and presented for broker approval.

**Time and value saved (target value hypotheses — not independently measured):**

| Use case | Traditional broker task | With The Buddy | Target time saved |
|----------|------------------------|-----------------|-------------------|
| Lead asks about viewing or availability | Broker engages immediately, checks calendar, responds manually | Buddy reads inquiry, checks CRM + calendar, confirms and locks viewing autonomously | ~40% |
| Client requests property/project information | Broker gathers data, prepares response, sends | Buddy assembles information pack, broker reviews and approves, Buddy sends | ~60% |
| New listing onboarding | Broker coordinates photographer, collects images, drafts description, updates CRM | Buddy schedules photoshoot, receives images, drafts listing, broker approves to go live | ~60% |

**Competitive value:** Brokers without an EA either handle admin themselves (slower response, fewer leads managed) or hire a human assistant (cost, training, inconsistency). The Buddy provides consistent, always-on administrative support that scales with the broker's lead volume. The graduated autonomy model means the broker is never a bottleneck for simple confirmations, but always in control of substantive communication.

### Technical Layer

**Core responsibilities:**
- Triage inbound WhatsApp and email messages and classify them by required action tier
- Manage the broker's calendar (add, remove, remind) for viewings, meetings, and follow-ups
- Maintain CRM records (create/update leads, update listings, attach documents, log interactions)
- Draft personalized communications (WhatsApp messages, email, property shortlists) using persona data from The Inspector
- Coordinate listing onboarding (schedule photoshoots, receive images, draft descriptions, prepare for portal publication)
- Escalate to admin team when broker approvals are not completed within the required time window

**Explicit out-of-scope boundaries:**
- Does NOT send any substantive, advisory, transactional, pricing, or legal content without explicit broker approval
- Does NOT access the internet or run Apify Actors (that is The Inspector's role)
- Does NOT make pricing, commission, or negotiation decisions
- Does NOT access private social media accounts or use purchased data lists
- Does NOT replace government-system steps (Trakheesi permit issuance, Ejari registration, DLD transfer — these remain broker/government actions)
- Does NOT infer sensitive attributes about leads

**Graduated autonomy model:**

| Tier | Trigger | Example inquiry | Buddy action | Approval required | Risk flag |
|------|---------|-----------------|--------------|-------------------|-----------|
| Tier 1: Bounded admin | Simple scheduling/availability | "Can I view tomorrow?" / "Is this still available?" | Checks CRM availability + broker calendar; proposes/confirms available slots using pre-approved template; creates viewing in CRM | No per-message approval (policy-enabled; all actions logged for broker review) | **Must be approved in product policy before production. Only pre-approved template messages are used. Broker can disable Tier 1 at any time.** |
| Tier 2: Broker-approved content | Substantive property/transaction/legal/pricing | "What are the payment plan options?" / "Is this a good investment?" | Drafts response using CRM data, property cards, persona JSON; queues for broker approval | **Broker approval required** — nothing sent until approved | — |
| Tier 3: Escalation | Broker does not approve Tier 2 draft within time window | — | Notifies broker via push notification; if still no response within extended window, escalates to admin team | Escalation only; no auto-send | Broker remains accountable for completing the task |

**Risk flag — autonomy exception:** The original design constraint stated that nothing client-facing is ever sent without broker approval. The user's revised requirement introduces a bounded exception for Tier 1 scheduling and availability actions. This exception must be formally approved in product policy before production. Only pre-approved, low-risk scheduling and availability messages may be sent autonomously. All autonomous actions are logged with full provenance for broker review. The broker can disable Tier 1 autonomy at any time, reverting all communication to Tier 2 (full approval required). Tier 1 autonomy is disabled by default until the brokerage approves the template library, allowed trigger phrases, calendar rules, and audit policy.

**Required tools and integrations:**

| Tool | Purpose | Type |
|------|---------|------|
| WhatsApp Business API | Send/receive WhatsApp messages (primary client channel) | External API |
| Email API (IMAP/SMTP or service) | Send/receive email (secondary client channel) | External API |
| CRM Adapter (vendor-neutral) | Interface to CRM for leads, listings, calendar, documents | Abstraction layer |
| LLM (drafting and triage) | Message classification, response drafting, listing description generation | AI model |
| Broker Dashboard UI | Approval queue, persona review, draft editing, calendar view | Internal UI |

**CRM Adapter interface (vendor-neutral):**

The CRM Adapter is an abstraction layer that The Buddy communicates with. It defines a set of required capabilities that any underlying CRM (PropSpace, Ruby CRM, or other) must implement:

| Method | Purpose | Notes |
|--------|---------|-------|
| `getLead(lead_id)` | Retrieve lead record | — |
| `createLead(payload)` | Create new lead | Triggered on new portal lead arrival |
| `updateLead(lead_id, fields)` | Update lead (status, persona, notes) | Used when Inspector delivers persona |
| `getListing(listing_id)` | Retrieve listing details | Used for availability checks and info packs |
| `updateListing(listing_id, fields)` | Update listing (images, description, status) | Used during listing onboarding |
| `checkAvailability(listing_id)` | Check if property is available/active | Used for Tier 1 autonomous replies |
| `getBrokerCalendar(broker_id)` | Retrieve broker's calendar/schedule | Used for viewing scheduling |
| `createViewingAppointment(payload)` | Book a viewing in CRM + calendar | Used for Tier 1 autonomous confirmations |
| `attachDocument(record_id, file)` | Attach document to lead/listing | Used for interaction logs, images, permits |
| `generateOrFetchPropertyCard(listing_id)` | Retrieve or generate property card | Integration-dependent — if CRM does not support this, Buddy assembles from available listing fields |
| `generateOrFetchCMA(criteria)` | Retrieve or generate CMA | Integration-dependent — if CRM does not support this, Buddy prepares a draft request for broker action |

**Note on property card and CMA capabilities:** These are user-defined requirements. The verified research confirms that PropSpace offers auto-matching of buyer requirements to listings and stores payment plans, service charges, and escrow numbers per unit ([PropSpace 2026 Guide](https://www.propspace.com/blog/real-estate-crm-dubai-complete-guide)), but does not confirm a specific "property card generation" or "CMA generation" feature across all CRMs. Bayut's Profolio offers Market Insights (top communities, ROI areas) and Transaction Insights (DLD sold data) which could serve as CMA data sources ([Bayut Profolio Features](https://www.bayut.com/agentportal/best-features-profolio-offers/)). If the underlying CRM does not support these natively, The Buddy can only prepare a draft request or assemble available data from listing fields — it cannot invent missing CRM capability.

**Modular Skills:**

The Buddy's capabilities are organized as modular "Skills" — discrete, configurable modules that can be enabled or disabled per broker or agency:

| Skill | Description | Phase |
|-------|-------------|-------|
| Calendar Skill | Add/remove/remind viewings, meetings, follow-ups on broker calendar | Phase 1 |
| WhatsApp Triage Skill | Read, classify, and respond to inbound WhatsApp messages per autonomy tiers | Phase 1 |
| Email Triage Skill | Read, classify, and respond to inbound emails per autonomy tiers | Phase 1 |
| CRM Lead Management Skill | Create/update leads, log interactions, attach persona data | Phase 1 |
| Listing Admin Skill | Coordinate photoshoots, receive images, draft descriptions, prepare for portal publication | Phase 1 |
| Property Card / CMA Skill | Assemble property cards and CMA data from CRM/portal data (integration-dependent) | Phase 1 |
| Approval & Escalation Skill | Manage approval queue, notify broker, escalate to admin team | Phase 1 |
| Voice Agent Skill | AI voice for inbound/outbound follow-up calls | Phase 2 |
| Social Media Skill | Manage social media accounts, respond to comments, schedule posts per approval/plan | Phase 2 |

**Data inputs:**

| Input | Source | Format |
|-------|--------|--------|
| Inbound WhatsApp messages | WhatsApp Business API | Message object (sender, body, timestamp, media) |
| Inbound emails | IMAP / email API | Email object (sender, subject, body, attachments) |
| Portal leads | Portal webhook (Property Finder, Bayut, Dubizzle) | Lead object (name, phone, email, listing ref, enquiry, source) |
| Persona JSON | The Inspector (via shared context store) | Persona JSON (see Inspector outputs) |
| Active inventory | CRM Adapter: `getListing()` | Listing objects |
| Broker calendar | CRM Adapter: `getBrokerCalendar()` | Calendar/schedule object |
| Broker templates | Pre-approved message templates (Tier 1) | Template objects |

**Data outputs:**

| Output | Format | Destination |
|--------|--------|-------------|
| Tier 1 autonomous replies | Pre-approved template message + appointment details | WhatsApp / Email API → client |
| Tier 2 draft responses | Draft message + supporting data | Broker Dashboard (approval queue) |
| Property shortlists | Structured shortlist (3–7 options, 7 fields each, WhatsApp CTA — AllRealEstate format) | Broker Dashboard (approval queue) |
| Listing drafts | Title, description, images, price, permit number | Broker Dashboard (approval queue) → CRM → portal syndication |
| CRM updates | Lead records, listing records, interaction logs, calendar entries | CRM Adapter |
| Escalation notifications | Push notification + admin alert | Broker (push) + Admin team (notification) |

**Human-in-the-loop checkpoints:**

| Checkpoint | Trigger | Action | Fallback if no response |
|-----------|---------|--------|-------------------------|
| Tier 2 draft approval | Buddy drafts substantive response | Broker reviews/edits/approves on dashboard | Push notification after configurable delay (default: 10 min); if no response within extended window (default: 30 min), escalate to admin team |
| Listing draft approval | Buddy completes listing onboarding draft | Broker reviews/approves on dashboard | Same escalation pattern |
| Persona review | Inspector delivers persona JSON | Broker reviews on dashboard (non-blocking — persona informs future interactions) | No fallback needed; persona is stored in CRM regardless |
| Tier 1 action log review | Buddy sends autonomous Tier 1 reply | Broker can review all autonomous actions in dashboard log | No fallback; actions are logged for audit |

**Escalation chain:**

1. Draft enters approval queue → broker notified via push notification
2. If no response after 10 minutes → second push notification (escalated priority)
3. If no response after 30 minutes → admin team notified via dashboard alert and/or email
4. The broker remains accountable for completing the task. The system does not auto-send Tier 2 content under any circumstances.
5. All escalation timestamps and actions are logged in the CRM for audit purposes.

**Handoff protocol (Inspector → Buddy):**

1. Inspector writes persona JSON to shared context store (keyed by `lead_id`)
2. Inspector emits event `persona.enriched.{lead_id}`
3. Buddy's event listener receives event
4. Buddy executes: `CRM Adapter.updateLead(lead_id, { persona: personaJSON })`
5. Buddy pushes enriched persona to broker dashboard for review
6. If a Tier 2 draft is pending for this lead, Buddy re-drafts using the now-available persona data (personalization hooks, buyer type signal, professional context)
7. If no Tier 2 draft is pending, Buddy stores the persona and uses it for future interactions with this lead

**Success metrics / KPIs:**

| KPI | Definition | Target |
|-----|------------|--------|
| Tier 1 autonomous resolution rate | % of inbound inquiries resolved at Tier 1 without broker intervention | > 30% |
| Broker approval rate | % of Tier 2 drafts approved without edits | > 70% |
| Average time-to-approval | Time from draft queued to broker approval | < 15 minutes |
| Escalation rate | % of Tier 2 drafts that escalate to admin team | < 10% |
| CRM data completeness | % of leads with complete CRM records (interaction log, persona attached, status updated) | > 90% |
| Listing onboarding time | Time from listing instruction to portal-ready draft | < 24 hours |
| Broker satisfaction | Broker-reported satisfaction with Buddy assistance (survey) | > 4/5 |

**Example scenario — "Sarah Chen" lead (continuation):**

1. Lead arrives: Sarah Chen sends WhatsApp message "Hi, I saw the 2BR in Business Bay on Property Finder. Is it still available? Can I view tomorrow?"

2. Buddy Triage Skill classifies: Tier 1 (availability check + viewing request)

3. Buddy calls `CRM Adapter.checkAvailability(listing_id)` → property is available

4. Buddy calls `CRM Adapter.getBrokerCalendar(broker_id)` → broker has 2 PM and 4 PM open tomorrow

5. Buddy sends pre-approved Tier 1 template: "Hi Sarah, yes the 2BR in Business Bay is still available. The broker has openings at 2 PM or 4 PM tomorrow — which works for you?"

6. Sarah responds "2 PM works." Buddy calls `CRM Adapter.createViewingAppointment({ listing_id, client_name: "Sarah Chen", phone, time: "tomorrow 2 PM" })`. Buddy sends confirmation: "Great, you're booked for 2 PM tomorrow at [Property Address]. See you there."

7. Broker receives notification: "Buddy handled a viewing booking for Sarah Chen (2BR Business Bay, tomorrow 2 PM)."

8. Meanwhile, Inspector has delivered Sarah's persona: Senior PM at Checkout.com, MBA from INSEAD, relocation buyer, high confidence.

9. Before the viewing, the broker reviews the persona on their dashboard. They decide to prepare a tailored property shortlist including Business Bay options near DIFC and a Downtown option for commute comparison.

10. Broker asks Buddy to draft a shortlist. Buddy uses the AllRealEstate format (3–7 options, 7 fields each, WhatsApp CTA) and the persona's personalization hooks ("proximity to DIFC for fintech network"). Buddy drafts the shortlist and queues it for broker approval.

11. Broker reviews the shortlist on their dashboard, edits one option, approves.

12. Buddy sends the approved shortlist via WhatsApp to Sarah.

13. Buddy logs the full interaction in the CRM: messages sent, persona attached, viewing booked, shortlist sent, approval timestamps.

---

# PART C — Gaps & Open Decisions

## 1. CRM vendor selection and adapter implementation

**Status:** User confirmed vendor-neutral design. The CRM Adapter interface is defined above, but the actual implementation against a specific CRM (PropSpace, Ruby CRM, or other) is an open decision. PropSpace is the best-documented CRM in the verified research (80+ portal syndication, RERA Forms A/B/F/Form I generation, Lead Pool/Lead Rotation, optional Trakheesi permit validation, pricing AED 220/user/month). Ruby CRM offers the most portal integrations (4 portals: Bayut, Property Finder, Dubizzle, Skyloov) and native WhatsApp Business integration. The adapter implementation should prioritize the CRM the brokerage already uses or plans to adopt.

**Decision needed:** Which CRM(s) should the adapter be implemented against first?

## 2. Tier 1 autonomy boundary definition

**Status:** The user defined Tier 1 as "simple" inquiries (viewing requests, availability checks) that the Buddy can handle autonomously. The exact boundary of what constitutes "simple" needs precise definition. Proposed Tier 1 scope:
- Property availability confirmation (yes/no based on CRM status)
- Viewing scheduling (propose/confirm time slots from broker calendar)
- Acknowledgement of receipt ("Thank you, [Broker Name] will respond shortly")
- Reminder messages to clients about upcoming viewings

Proposed Tier 2 scope (everything else):
- Any question about property details, features, or specifications
- Any question about pricing, payment plans, or commission
- Any question about transaction process, legal, or regulatory matters
- Any question about investment advice or area recommendations
- Any personalized communication using persona data

**Decision needed:** Confirm the Tier 1 / Tier 2 boundary above, or adjust.

## 3. Tier 1 template library

**Status:** Tier 1 autonomous actions must use pre-approved message templates only. A library of templates needs to be created and approved by the brokerage before Tier 1 autonomy can be enabled. Templates should cover: availability confirmation, viewing proposal, viewing confirmation, viewing reminder, and receipt acknowledgement.

**Decision needed:** Who creates and approves the initial template library? Brokerage management? Individual brokers? Should templates be customizable per broker or standardized agency-wide?

## 4. Escalation rules and admin team definition

**Status:** The user specified that if the broker does not approve a Tier 2 draft within the time limit, it escalates to the admin team. The exact escalation rules (time windows, notification channels, admin team composition) need definition. Proposed defaults:
- 10 minutes → second push notification to broker
- 30 minutes → admin team notified via dashboard alert + email
- Admin team can reassign the task or contact the broker directly

**Decision needed:** Confirm escalation time windows and define who constitutes the "admin team" (office manager, team lead, operations manager?).

## 5. Property card and CMA "Skills" grounding

**Status:** The user specified that the Buddy should have "Skills" to extract property cards and CMA data from portals and CRM. The verified research confirms that PropSpace stores payment plans, service charges, and escrow numbers per unit and offers auto-matching of buyer requirements to listings, but does not confirm a specific "property card generation" or "CMA generation" feature. Bayut's Profolio offers Market Insights (top communities, ROI areas) and Transaction Insights (DLD sold data), which could serve as CMA data sources. The exact implementation of these Skills depends on which CRM and portal APIs are available.

**Decision needed:** Should the Property Card / CMA Skill be designed to pull data from Bayut Market Insights / Transaction Insights, from the CRM's own data, or from both? What constitutes a "property card" in the brokerage's workflow?

## 6. Shortlist format

**Status:** The verified research confirms the AllRealEstate shortlist format (3–7 options, 8-point brief, 7 comparison fields per option, WhatsApp CTA, < 3 minutes read) ([AllRealEstate Shortlist Guide](https://allrealestate.ae/guides/client-property-shortlists-for-brokers)). This is the most concrete public template available. The design uses this format for The Buddy's shortlist output.

**Decision needed:** Use the AllRealEstate format as-is, or design a brokerage-branded variant? If the latter, what branding elements (logo, colors, agent profile) should be included?

## 7. Apify cost budget per lead

**Status:** The consolidated research estimates $0.01–$0.05 per lead for Inspector enrichment, but this is an illustrative estimate, not an official Apify price. Actual costs depend on the Actors used, result counts, and Apify plan tier. The MCP server's `search-actors` and `fetch-actor-details` calls are free. `maxTotalChargeUsd` should be set on each Actor run to prevent unexpected charges. Apify's free plan includes $5/month in credits.

**Decision needed:** What is the acceptable cost-per-lead budget for enrichment? Should enrichment be skipped for low-value leads (e.g., rental inquiries below a threshold)?

## 8. Phase 2 scope and timeline

**Status:** The user identified Phase 2 capabilities: AI Voice Agent for inbound/outbound follow-up, and Social Media Management. These are out of scope for the current design but should be architecturally accommodated (the modular Skills system allows Phase 2 Skills to be added without redesigning the core agent).

**Decision needed:** When should Phase 2 be scoped in detail? Should the current architecture include placeholder interfaces for voice and social media Skills?

## 9. WhatsApp Business API and email infrastructure

**Status:** The design assumes WhatsApp Business API access (for sending/receiving messages) and an email infrastructure (IMAP/SMTP or email service). The verified research confirms that WhatsApp is the dominant communication channel in Dubai real estate, and that WhatsApp Business integration is offered by some CRMs (Ruby CRM's WAHA integration). The specific WhatsApp Business API provider and email infrastructure are open decisions.

**Decision needed:** Which WhatsApp Business messaging provider should be used? The CRM-integrated WhatsApp option (e.g., Ruby CRM's WAHA integration) may be used if the selected CRM supports it. Which email infrastructure?

## 10. Compliance and legal review

**Status:** The verified research explicitly states that Apify access does not override platform terms of service, data-protection rules, telemarketing rules, or messaging-channel rules. LinkedIn enrichment, WhatsApp outreach, data retention, opt-out/deletion, and cross-border processing all require qualified legal review. The design includes compliance guardrails (public/consented data only, provenance tracking, opt-out/deletion mechanism, sensitive-data avoidance), but these are design principles, not legal opinions.

**Decision needed:** Engage a qualified legal review before production use, covering: LinkedIn ToS compliance, UAE data protection law (PDPL), WhatsApp Business messaging rules, data retention policies, and cross-border data processing (Apify is hosted outside the UAE).

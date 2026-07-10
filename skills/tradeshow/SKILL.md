# tradeshow

<!-- GUILDOS-SKILL name="tradeshow" -->

<!-- GUILDOS-DESC -->
End-to-end tradeshow planning from booth selection through post-show follow-up. Actions cover creating the Google Calendar planning events for a show, booking the booth in the exhibitor portal, planning travel/shipping logistics, marketing and materials, on-site prep, closing Calendly availability for the travel window, and the post-show lead + debrief cycle.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/tradeshow/SKILL.md -->
```markdown
---
name: tradeshow
description: >-
  End-to-end tradeshow planning from booth selection through post-show
  follow-up. Actions cover creating the Google Calendar planning events for a
  show, booking the booth in the exhibitor portal, planning travel/shipping
  logistics, marketing and materials, on-site prep, closing Calendly
  availability for the travel window, and the post-show lead + debrief cycle.
---

# tradeshow

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/tradeshow/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto tradeshow <action>` (or read the source directly).

## Actions

- **bookBooth** — Select booth on floorplan, submit application within 1-hour window, and process initial payment
- **closeCalendlyAvailability** — Close Calendly availability for every day from the inbound flight through the outbound flight; verify zero bookable slots via the calendly weapon
- **planLogistics** — Book flights (<$100 above cheapest), hotel/Airbnb (<$250/day via AACR CMR), and arrange crate shipping
- **planMarketing** — Notify marketing team, request sponsorship package, launch pre/post-show email campaigns
- **planMaterials** — Audit inventory then order printing, giveaways, lead retrieval, electric
- **planShow** — Create all Google Calendar planning events for a given show using its slug and known key dates
- **postShow** — Export leads, send 24h follow-up, update CRM, run debrief. Return shipping needs no reminder.
- **prepOnsite** — Clear Calendly regular slots, open show meeting slots, pack checklist, brief booth staff
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/tradeshow/index.js -->
```js
export const definition = {
  id: "tradeshow",
  name: "Tradeshow",
  description: "End-to-end tradeshow planning from booth selection through post-show follow-up",
  toc: {
    planShow: {
      description: "Create all Google Calendar planning events for a given show using its slug and known key dates",
    },
    bookBooth: {
      description: "Select booth on floorplan, submit application within 1-hour window, and process initial payment",
    },
    planLogistics: {
      description: "Book flights (<$100 above cheapest), hotel/Airbnb (<$250/day via AACR CMR), and arrange crate shipping",
    },
    planMarketing: {
      description: "Notify marketing team, request sponsorship package, launch pre/post-show email campaigns",
    },
    planMaterials: {
      description: "Audit inventory then order printing, giveaways, lead retrieval, electric",
    },
    prepOnsite: {
      description: "Clear Calendly regular slots, open show meeting slots, pack checklist, brief booth staff",
    },
    closeCalendlyAvailability: {
      description: "Close Calendly availability for every day from the inbound flight through the outbound flight; verify zero bookable slots via the calendly weapon",
    },
    postShow: {
      description: "Export leads, send 24h follow-up, update CRM, run debrief. Return shipping needs no reminder.",
    },
  },
};

export const skillBook = {
  ...definition,
  actions: {
    planShow: {
      ...definition.toc.planShow,
      howTo: `
Create all Google Calendar planning events for a tradeshow from booth-booked through post-show.

INPUTS:
- slug: calendar tag used in every event title, e.g. AACR2027
- showStart / showEnd: first and last exhibit hall day
- venueCity / venueState: used to determine fly-in date (see TRAVEL RULES below)
- depositDeadline: 50% deposit due date (if known)
- finalPaymentDeadline: final payment due date (if known)

CALENDAR TARGET: "Boster Marketing Projects" calendar.
Attendees on all events: <REDACTED-EMAIL>, <REDACTED-EMAIL>
Reminders: all-day events → 1-day-before email; timed events → 1-day-before + 30-min-before.

ONLY CREATE ACTION-REQUIRED EVENTS. Never create passive reminders (e.g. cancel/downsize
deadline, housing deadline, rendering deadline). Only create events that require someone to
actively do something on that day.

TRAVEL DATE RULES (locked):
- Setup day = last move-in day = day BEFORE showStart. Do NOT mark earlier move-in days.
- Fly-in date:
  - West Coast venue (CA, OR, WA, NV, AZ): fly in SAME day as setup day.
    Rationale: can land ~noon and complete setup in ~2 hours.
  - All other locations: fly in 1 day BEFORE setup day.
- Hotel dates: check-in on fly-in day, check-out day after showEnd.
- The "Book Hotel + Flights" event description MUST show the calculated fly-in date and
  hotel stay dates so the booker has exact targets.

STANDARD EVENT SET (create these, in date order):
1. Notify Marketing + Request Sponsorship Package — +1 day after booth booked
2. 50% Deposit Due — from depositDeadline (if known; skip if unknown)
3. Book Hotel + Flights — ~90 days after booth booked; include fly-in date + hotel dates in desc
4. Final Booth Payment Due — from finalPaymentDeadline (if known; skip if unknown)
5. Order Badges + Lead Retrieval + Electric — ~11 weeks before showStart
6. Order Printing (Booth/Banners/Flyers/Giveaways) — ~9 weeks before showStart
7. Ship Booth Materials — ~3 weeks before setup day
8. Pre-Show Prep: Clear Calendar + Launch Email Campaign — ~2 weeks before setup day
   (this event's owner runs the CHAINED ACTION below: tradeshow.closeCalendlyAvailability)
9. Prepare Post-Show Follow-Up Emails — 1 WEEK BEFORE showStart
   (Give marketing team time to write, review, and stage emails for release right after show)
10. Fly In — fly-in date per travel rules above (create even if same as setup day)
11. Booth Setup Day — setup day (day before showStart)
12. Show Days — showStart through showEnd (multi-day all-day)

DO NOT CREATE:
- Cancel/downsize deadline (passive)
- Housing deadline (passive)
- Island booth rendering deadline (passive/conditional)
- Return shipping reminder (handled on-site; no calendar reminder needed)
- Multi-day move-in block (only setup day matters)

CHAINED ACTION (run as part of this workflow, after the event set is created):
Chain tradeshow.closeCalendlyAvailability for this show. Pass slug + the inbound
flight date and outbound flight date computed from the TRAVEL DATE RULES above
(fly-in date → day after showEnd, inclusive). It closes Calendly for every travel
day and verifies zero bookable slots via the calendly weapon. This is the
mechanized owner of "clear regular Calendly availability" — prepOnsite references
it rather than restating the mechanism.

VERIFY: Screenshot https://calendar.google.com/calendar/u/0/r/search?q=[slug]
      `.trim(),
    },

    bookBooth: {
      ...definition.toc.bookBooth,
      howTo: `
Go to the exhibitor portal link at appointment time (from booth selection email, format:
https://[SHOW].exh.mapyourshow.com/boothsales/?renewkey=[CODE]).

STEPS:
1. Log in with booking code from email.
2. Floorplan: blue = available, gray = taken. Prefer spots near poster sessions, coffee breaks,
   or wellness lounge (premium traffic). Corner adds $200/corner.
3. Submit application within 1 hour — system times out and releases your slot.
4. Payment: choose "bill me" (check/wire) OR pay online with credit card.
5. Confirmation email with booth number + invoice arrives within minutes.

IF LATE: Same link works anytime after your slot — first-come-first-served from that point.
IF WANT AACR TO ASSIGN: Email <REDACTED-EMAIL> before your slot, ask for prior-year location.

AACR 2027 PRICING:
- 10x10 inline: $4,650
- Corner surcharge: +$200/corner
- Premium (near posters/coffee/wellness): $4,850
- Island: $48.50/sq ft
- Non-profit: $1,200 first booth (select booth in non-profit section)
- End cap booths: NOT permitted

FINANCIAL DEADLINES (AACR 2027):
- Cancel/downsize penalty: 25% after Jun 12, 2026
- 50% deposit: Jul 16, 2026
- Final payment: Sep 15, 2026
- Failure to pay deposit by deadline = forfeit of booth space
      `.trim(),
    },

    planLogistics: {
      ...definition.toc.planLogistics,
      howTo: `
Book in this order (each unlocks or informs the next):

HOTEL (book as soon as 50% deposit is paid):
- Use AACR official housing vendor CMR (Convention Management Resources) — only companies
  with confirmed booth + deposit can access CMR housing.
- Budget: under $250/day total for the party.
- Deadline: watch exhibitor bulletin, typically ~5 weeks before show (AACR 2026 pattern: March 6).
- Book AACR official hotels for free shuttle access to convention center.
- Dates: arrive 1 day before first move-in; check out day after last show day.

FLIGHTS (book ASAP after hotel confirmed):
- Rule: <$100 above cheapest available fare. Multi-stop acceptable for significant savings.
- Southwest preferred: points + no change fees. Ensure flights connected to Southwest account.
- Arrive: morning of day before move-in starts (flexibility for setup).
- Depart: AFTER last show day ends — booth must be staffed until official close, no exceptions.

SHIPPING — OUTBOUND (ship ~3 weeks before move-in):
- Carrier priority: ODFL (866) 637-7333 for best price > FedEx > UPS.
- Ship to WAREHOUSE address (from Freeman service kit in Map Your Show Exhibitor Resource Center).
- Drayage charge moves crate from warehouse to booth.
- Required docs: Shepard Outbound Material Handling Authorization form + shipping labels.
  Submit via Shepard portal BEFORE shipping — saves time on-site. CLICK HERE link in exhibitor kit.
- Bill of Lading MUST include: Show Name, Booth #, Exhibitor Name, Contractor name + contact info.

SHIPPING — RETURN:
- Drayage moves crate from booth back to DOCK; carrier picks up from DOCK.
- Complete return shipping form before or during show.
- Timeline: carrier pick-up Apr 6 midnight – Apr 7 noon (AACR 2027 pattern).
      `.trim(),
    },

    planMarketing: {
      ...definition.toc.planMarketing,
      howTo: `
1. NOTIFY MARKETING TEAM (immediately after booth confirmed, +1 day):
   - Tell marketing lead (Tracy or current) all shows for the year so she can:
     - Plan announcement emails
     - Make ad banners on website
   - Provide: show name, dates, booth number, city, show URL.

2. SPONSORSHIP / ADVERTISING INQUIRY (within 1 week of booking):
   - Email show organizer (<REDACTED-EMAIL> for AACR) requesting sponsorship and advertising
     media package.
   - AACR options: mobile charging lounges, wellness lounge, coffee break sponsorship,
     exhibitor spotlight theater, water station, TrafficMax email program, digital advertising
     via Walchli Tauber Group (WTG), out-of-home via TriStar Event Media.
   - WTG and TriStar are the ONLY official AACR advertising vendors — do not use others.

3. EXHIBITOR TOOLKIT:
   - Download from Map Your Show Exhibitor Resource Center.
   - Use official assets for all social/email promotion — required per AACR guidelines.

4. PRE-SHOW EMAIL CAMPAIGN (~3 weeks before show):
   - Use SciLeads or AACR-provided attendee list.
   - Send 1: 3 weeks out — "Meet us at booth [#]" + product highlight + show special if any.
   - Send 2: 1 week out — reminder + "book a meeting" CTA with Calendly link.

5. POST-SHOW FOLLOW-UP (within 48h of show close):
   - Export leads from lead retrieval device/app.
   - Segment by product interest (noted during show).
   - Send personalized "Great meeting you" email with relevant follow-up content.
   - Add to CRM pipeline; schedule calls with hot leads within 1 week.
      `.trim(),
    },

    planMaterials: {
      ...definition.toc.planMaterials,
      howTo: `
START: ~9–10 weeks before show. Allow 2–3 weeks design + production + shipping.

STEP 1 — INVENTORY AUDIT (before ordering anything):
- Count and inspect: booth backdrop, stand-up banners, flyers, giveaways, ebooks, business cards.
- Only order what's depleted or needs refresh.

PRINT MATERIALS (budget ~$200/mo, coordinate with designer):
- Booth backdrop/graphic
- Stand-up banners
- Product/service flyers (product + services focus)
- Pathway maps (if custom layout)
- Giveaways: standard items 4–6 weeks lead time; custom/branded 6–8 weeks.

SERVICES TO ORDER via Map Your Show Exhibitor Resource Center:
- Exhibitor registration badges — order early (unlocks CMR housing access)
- Lead retrieval device — order from official show provider
- Electric hook — order via Freeman
- Furniture rental: carpet, tables, chairs — cheaper to buy if attending 3+ shows/year
- Porter service — REQUIRED if distributing giveaways or food at booth (order via convention center)
- Internet upgrade via Smart City — only if streaming video (free wifi insufficient for streaming)
- Booth cleaning — exhibitor's responsibility; order via Freeman

ISLAND BOOTH RENDERING (if island/400+ sq ft):
- Submit booth design to AACR for approval ~7 weeks before show (AACR 2026 deadline: Feb 13).
- Cannot set up without written approval.
- Upload via Map Your Show Exhibitor Resource Center.
- Contact <REDACTED-EMAIL> with questions.
      `.trim(),
    },

    prepOnsite: {
      ...definition.toc.prepOnsite,
      howTo: `
2 WEEKS BEFORE SHOW:
- Remove regular Calendly availability for all travel + show days — run
  tradeshow.closeCalendlyAvailability (it owns the mechanism + verification).
- Create "Meet at [Show] — Booth [#]" Calendly slots matching exhibit hall hours.
- Confirm hotel: log in to Airbnb app or retrieve hotel confirmation email.
- Confirm flights: Southwest boarding pass ready or auto-check-in set up.
- Buy/pack checklist: table cover, chairs, carpet tiles (if not renting), power strip,
  hand sanitizer, business cards, printed flyers, giveaways.

1 WEEK BEFORE:
- Send "Know Before You Go" briefing to all booth staff:
  show hours, badge pickup location, booth number, Freeman desk location,
  parking/shuttle info, no-outside-food rule.
- Confirm all Freeman orders placed (electric, cleaning, furniture if renting).
- Confirm lead retrieval device pickup process (location + timing).
- Second pre-show email send to attendees.

ON ARRIVAL:
- Badge pickup: registration desk (bring at all times — exhibitor badge is green).
- EAC contractors: wristbands at EAC Desk at contractor entrance (between loading docks).
- Freeman Service Desk: behind Hall B in Bayside Lobby (AACR pattern) — AV, shipping,
  electricity, carpet, cleaning.
- Booth must be show-ready by end of last move-in day. Late move-in requires prior approval.
- No alcohol or outside food in exhibit hall.
- Packing NOT permitted until official show close on last day — strictly enforced.
      `.trim(),
    },

    closeCalendlyAvailability: {
      ...definition.toc.closeCalendlyAvailability,
      howTo: `
Close Calendly availability for the entire tradeshow travel window so no prospect
can book a regular call while booth staff are travelling or on the show floor.
Standalone-callable (a leaf agent can run JUST this for one show) and chained into
\`planShow\` as the pre-show prep step.

INPUTS:
- slug: show tag, e.g. PEGQ (used only for the override note + verification log)
- inboundFlightDate / outboundFlightDate: YYYY-MM-DD. The closure window is EVERY
  calendar day from the inbound flight day THROUGH the outbound flight day,
  inclusive. If only show dates are known, derive the window from planShow's
  TRAVEL DATE RULES (fly-in date → day after showEnd) — never guess; read the
  show's calendar plan to confirm the real flight dates.

WINDOW: build the inclusive YYYY-MM-DD list once. Every day in it gets closed AND
verified — partial closure is a fail.

WRITE PATH (close the days) — try in this order, stop at the first that works:
1. Calendly MCP (PREFERRED): if \`mcp__calendly__*\` tools are present, use the
   availability-update tools to mark each day in the window unavailable (a date
   override set to Unavailable on the user's default availability schedule).
   Note: the Calendly MCP (https://mcp.calendly.com) is OAuth 2.1 DCR — it needs a
   one-time \`/mcp\` authorize before its tools appear. If the tools are NOT
   present this session, do NOT block — fall straight to step 2.
2. CIC FALLBACK (ALWAYS WORKS LOCALLY): drive the logged-in calendly.com via
   Claude-in-Chrome. Availability → the user's default schedule → "Add date
   override" → select each day in the window → set to Unavailable → Apply.
   Repeat per day (or use a multi-date override if the UI offers it). The public
   Calendly REST API is READ-ONLY for availability — there is NO API write path;
   never claim the API closed the days.

VERIFY (MANDATORY — API + visual; a close is NOT done until both pass):
- API proof: \`calendly\` weapon → \`readAvailabilityClosure({ eventTypeUri, dates })\`
  where dates = the full window list and eventTypeUri is the user's primary
  bookable event type (\`searchEventTypes\` → pick the active 30-min/intro type).
  Require \`allClosed === true\` (every day reports slotCount 0). If any day still
  has slots, the override didn't take — re-apply that day and re-verify.
  (event_type_available_times only accepts a future, <=7-day range per call; the
  weapon already queries day-by-day, so windows >7 days are fine.)
- Visual proof: CIC screenshot of the Availability page showing each window day
  marked Unavailable. Save it as the deliverable artifact.

OUTPUT: { slug, window:[...dates], writePath:"mcp"|"cic", allClosed:true,
perDay:[{date,slotCount,closed}], screenshotPath }. Report WHERE the proof lives
(screenshot path + the readAvailabilityClosure result) — never self-attest.

REOPEN: closure is fully reversible — delete the date overrides (MCP or CIC) to
restore normal availability after the show.
      `.trim(),
    },

    postShow: {
      ...definition.toc.postShow,
      howTo: `
DAY SHOW ENDS:
- Do NOT pack until official close time (AACR: noon on last day — no exceptions).
- Arrange return drayage: crate from booth → DOCK → carrier pickup.
- Return lead retrieval device to show provider desk.
- Complete Shepard return shipping form if not done pre-show.

WITHIN 24H:
- Export all leads from lead retrieval device or app.
- Segment by interest level and product discussed.
- Send initial "Great meeting you at [Show]" email to all contacts.

WITHIN 1 WEEK:
- Add all leads to CRM.
- Send secondary email with product/service details matching their interest.
- Schedule calls/demos with hot leads.

DEBRIEF (update Tradeshow Progress deck in Google Slides):
- Actions taken
- Material count (leftovers = over-ordered; zero = under-ordered)
- Lead volume vs prior year
- Booth condition on return
- Sponsorship/engagement worth repeating
- Booth staff feedback
- Next actions for the follow-up cycle
      `.trim(),
    },
  },
};
```
<!-- /GUILDOS-FILE -->

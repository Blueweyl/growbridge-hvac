# GrowBridge HVAC — Software Blueprint

## Project Overview

GrowBridge HVAC is dispatch, membership-plan, and billing software built for independent HVAC service companies running 3–15 technicians. It replaces the whiteboard-and-spreadsheet stack these shops run on today, without forcing them into the enterprise complexity (or enterprise price) of ServiceTitan.

## Core Problem Being Solved

1. **Dispatch chaos** — jobs assigned via whiteboard/spreadsheet, techs calling the office for addresses and job history, no real-time visibility into who's where.
2. **PM (preventive maintenance) contract tracking** — these contracts are worth $5K–$15K/year each; renewal and scheduling is manual today, and a missed visit risks the whole contract.
3. **Billing lag** — 15–25 days between job completion and invoice sent because paperwork sits on a truck seat before the office can process it.
4. **Software cost-creep** — incumbents (ServiceTitan, Housecall Pro, Jobber) price and scale for 30–50 tech operations; a 5-person shop pays for complexity and add-ons it doesn't need.

## Target Users

- **Owner/Admin** — oversees the whole operation, cares about gross profit % and close rate, not feature lists.
- **Dispatcher/Office Staff** — runs the day-to-day board; often the same person doing billing and answering phones.
- **Field Technician** — needs job details, price book, and truck stock on a phone; will not tolerate a slow or confusing app.
- **Customer** (Phase 3) — receives appointment reminders and can view membership/service history.

## Software Architecture

### System Components

1. **Web Dashboard** — dispatch board, customer/property records, membership plans, invoicing, reporting. Used by owner and office staff.
2. **Technician Mobile View** — responsive PWA (not a native app) covering today's jobs, job detail, price book, photo/signature capture, and marking work complete.
3. **Backend API** — job/customer/billing logic, auth, multi-tenant data isolation (one GrowBridge instance serves many HVAC companies, each fully isolated).
4. **Database** — relational store for customers, properties, equipment, jobs, membership plans, invoices, and parts inventory.
5. **Integrations** — QuickBooks (accounting sync), Twilio (SMS reminders/on-my-way texts), Stripe (invoice payment + financed-job status), Google Calendar (optional two-way sync for dispatch).

### Tech Stack Recommendation

- **Frontend:** Next.js + TypeScript + Tailwind CSS — one codebase serves both the dispatch dashboard and the technician mobile view as a responsive PWA, avoiding a separate native app build.
- **Backend:** Next.js API routes / Supabase Edge Functions — keeps the stack in one repo for a small build team.
- **Database:** Supabase (Postgres) — built-in row-level security is exactly what multi-tenant, per-company data isolation needs; Realtime subscriptions power the live dispatch board without a custom websocket layer.
- **Auth:** Supabase Auth with role-based claims (admin, dispatcher, technician).
- **File storage:** Supabase Storage for job photos and signed work orders.
- **Hosting:** Vercel (frontend/API) + Supabase (database/auth/storage).
- **SMS:** Twilio for appointment reminders and "technician on the way" texts.
- **Payments:** Stripe for invoice payment collection and financed-job status webhooks.
- **Accounting:** QuickBooks Online API for two-way invoice/customer sync.

## Database Schema

**organizations**
- id (pk), name, subscription_tier, created_at

**users**
- id (pk), organization_id (fk), role (admin | dispatcher | technician), name, phone, email, created_at

**customers**
- id (pk), organization_id (fk), name, phone, email, billing_address, created_at

**properties**
- id (pk), customer_id (fk), address, property_type (residential | commercial), notes

**equipment**
- id (pk), property_id (fk), system_type, refrigerant_type, install_date, warranty_expiration — refrigerant_type supports the Phase 3 A2L compliance log

**jobs**
- id (pk), organization_id (fk), property_id (fk), assigned_technician_id (fk → users), job_type (service | install | pm_visit), status (scheduled | en_route | in_progress | completed | invoiced), scheduled_at, completed_at, notes, created_at

**job_line_items**
- id (pk), job_id (fk), price_book_item_id (fk), description, quantity, unit_price

**price_book_items**
- id (pk), organization_id (fk), name, flat_rate_price, category

**membership_plans**
- id (pk), organization_id (fk), name, price, billing_frequency (monthly | annual), visits_included_per_year

**membership_subscriptions**
- id (pk), customer_id (fk), plan_id (fk), status (active | past_due | cancelled), next_renewal_date, next_pm_visit_due

**invoices**
- id (pk), job_id (fk), customer_id (fk), amount, status (draft | sent | paid | overdue), sent_at, paid_at

**financed_jobs**
- id (pk), job_id (fk), lender_name, status (applied | approved | funded | denied), amount_financed, updated_at

**truck_inventory**
- id (pk), technician_id (fk → users), part_name, quantity_on_hand, reorder_threshold

**notifications_log**
- id (pk), customer_id (fk), job_id (fk), channel (sms | email), type (reminder | on_my_way | review_request), sent_at

### Relationships
- One organization → many users, customers, jobs (full row-level tenant isolation)
- One customer → many properties → many equipment records
- One property → many jobs
- One job → many line items, one invoice, optionally one financed_job record
- One customer → optionally one membership_subscription

## Feature Breakdown

### MVP (Phase 1) — Must-Have for Launch
**Estimated build time: 90–120 hours**

1. **Auth & roles** — admin, dispatcher, technician logins with row-level tenant isolation.
2. **Customer & property management** — customer records, multiple properties per customer, equipment/refrigerant tracking per property.
3. **Dispatch board** — drag-and-drop job assignment, real-time status updates (scheduled → en route → in progress → completed), today's-jobs view.
4. **Technician mobile view** — job detail, flat-rate price book lookup, photo capture, customer signature, mark job complete.
5. **Invoicing from completed jobs** — auto-generate a draft invoice from job line items the moment a job is marked complete, closing the 15–25 day billing-lag gap.
6. **Membership plan tracking** — plan catalog, subscription status per customer, upcoming PM-visit-due list so renewals stop being manual.

### Phase 2 — Enhanced Functionality
**Estimated build time: 40–60 hours**

1. **QuickBooks sync** — push invoices and customers to QuickBooks Online automatically.
2. **Financed-job tracker** — applied → approved → funded status attached to the job, visible on the dispatch board.
3. **SMS automation via Twilio** — appointment reminders, "technician on the way," and post-job review requests.
4. **Truck stock tracking** — per-technician parts inventory with reorder thresholds, so a job doesn't stall because the van doesn't have the part.

### Phase 3 — Nice-to-Haves
**Estimated build time: 25–35 hours**

1. **A2L/refrigerant compliance log** — reportable view of every installed system's refrigerant type against current state rules.
2. **Technician performance dashboard** — close rate, comeback (callback) rate, and tenure by technician, so an owner can see a flight risk or a training gap before it costs a job.
3. **Customer portal** — view service history, membership status, and request a new appointment.
4. **Route optimization** — suggest the most efficient order for a technician's scheduled jobs.

## Step-by-Step Implementation Plan

### Week 1 — Foundation
- "Set up a Next.js + TypeScript + Tailwind project, configure Supabase for database, auth, and storage."
- "Create Supabase tables for organizations, users, customers, properties, equipment with row-level security scoped to organization_id."
- "Implement Supabase Auth with admin/dispatcher/technician roles and role-based route middleware."

### Week 2 — Customers, Properties, Price Book
- "Build customer list/detail pages with property and equipment sub-records."
- "Build price book management (CRUD for flat-rate line items)."
- "Build membership plan catalog and subscription assignment to customers."

### Week 3 — Dispatch Board
- "Build the dispatch board with drag-and-drop technician assignment and Supabase Realtime status updates."
- "Build the technician mobile view: today's jobs, job detail, price book selection, photo/signature capture, mark-complete action."

### Week 4 — Billing & Membership Automation
- "Auto-generate a draft invoice from job line items when a job is marked completed."
- "Build the PM-visit-due list surfaced on the dashboard, driven by membership_subscriptions.next_pm_visit_due."
- "Add invoice send/paid status tracking."

### Week 5 — Phase 2 Integrations
- "Integrate QuickBooks Online API for invoice and customer sync."
- "Add the financed_jobs table and status UI on the job detail view."
- "Integrate Twilio for appointment reminder and on-my-way SMS."

### Week 6 — Polish & Handoff
- "Add truck inventory tracking per technician with reorder threshold alerts."
- "QA pass: multi-tenant isolation, mobile responsiveness, invoice accuracy."
- "Prepare admin and technician training walkthroughs."

## Deployment Strategy

- **Hosting:** Vercel (Next.js) + Supabase (Postgres, Auth, Storage, Realtime).
- **Steps:** push to GitHub → connect Vercel → set environment variables (Supabase keys, Twilio, Stripe, QuickBooks OAuth) → deploy → point custom domain per client if white-labeled.
- **Estimated monthly cost:** ~$50–$100 (Vercel Pro + Supabase Pro) per tenant at small scale; Twilio/Stripe fees are usage-based.

## Testing & QA Checklist

- [ ] Row-level security confirmed: one organization cannot see another's data
- [ ] Dispatch board real-time updates verified across two simultaneous sessions
- [ ] Mobile view tested on actual phone hardware, not just browser resize
- [ ] Invoice line items match job line items exactly on auto-generation
- [ ] Membership renewal/PM-due list produces correct dates across plan frequencies
- [ ] QuickBooks sync does not create duplicate customers/invoices on retry

## Pricing Recommendation

Per the niche-opportunity-finder analysis for this project (see project memory), this is scoped as a **custom build sold to one HVAC company at a time**, not an upfront multi-tenant SaaS launch — though the multi-tenant schema above means a second client can be onboarded without a schema rewrite.

- **MVP only:** $15,000–$20,000
- **MVP + Phase 2:** $22,000–$28,000
- **Full build (all phases):** $28,000–$35,000

**Justification:** one missed PM contract ($5K–$15K/year) or 2–3 disorganized jobs a month already exceeds the MVP price; closing the billing-lag gap alone improves cash flow within the first invoicing cycle.

## Next Steps

1. Validate the MVP feature list against 2–3 real HVAC shop-owner conversations before locking scope.
2. Confirm which phase the first client is buying (MVP vs. MVP+Phase 2) and price accordingly.
3. Use this file as the active CLAUDE.md for the build — reference it directly when directing Claude Code through each week's tasks.
4. Re-visit the multi-tenant design once the first client is live to decide whether to repackage as a flat-tier SaaS product (see the broader HVAC market research) or continue as a per-client custom build.

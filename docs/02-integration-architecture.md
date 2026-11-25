# Task 2 — Integration Architecture

Channels integrate through backend APIs and events. **No public browser code calls Dataverse directly.** The CRM API remains the stable system-of-record boundary.

## High-level diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│ CUSTOMER CHANNELS              INTERNAL              FINANCING          │
│ Magento / public site          Admin Backoffice      Financing Portal   │
│ Auth0 (customers)              Entra ID (staff)      Auth0 + partners   │
└──────────────┬─────────────────────────┬────────────────────┬───────────┘
               │                         │                    │
               ▼                         ▼                    ▼
        ┌──────────────┐          ┌──────────────┐    ┌──────────────────┐
        │ Profile      │          │              │    │ Lender adapters  │
        │ Service      │─────────►│   CRM API    │◄───│ Mashreq/Beehive  │
        └──────────────┘          │  (façade)    │    └──────────────────┘
               ▲                  └──────┬───────┘
               │                         │
        Magento catalog                  │ Dataverse Web API / SDK
        → Catalog adapter ───────────────┤
                                         ▼
                                  ┌─────────────┐
                                  │  Dataverse  │
                                  │ Account,    │
                                  │ Contact,    │
                                  │ KYB, Lead,  │
                                  │ catalog copy│
                                  └──────┬──────┘
                                         │ after commit
                                         ▼
                                  ┌─────────────┐     ┌──────────────────┐
                                  │ Azure       │────►│ Consumers:       │
                                  │ Service Bus │     │ portal, alerts,  │
                                  │ (AMQP)      │     │ reporting        │
                                  └─────────────┘     └──────────────────┘
```

## Five integration flows

| Flow | Path | Purpose |
|------|------|---------|
| **1 Registration** | Magento/UI → Auth0 → Profile Service → CRM API → Dataverse | Create/update Contact, Account, membership, onboarding, KYB |
| **2 Catalog sync** | Magento catalog → Catalog adapter → CRM API → Dataverse | Initial + incremental product/category upsert; Magento stays master |
| **3 Staff review** | Admin Backoffice → Entra ID → CRM API → Dataverse (+ Document Service refs) | KYB approve / reject / request-info with audit |
| **4 Financing** | Public/dashboard → Financing Portal ↔ CRM API; Portal ↔ lenders | Resolve company/KYB; store CRM summary; lenders own credit decision |
| **5 Events** | CRM API → Service Bus → consumers | `CompanyRegistered`, `KYBApproved`, `CatalogProductUpdated`, etc. |

## Interaction matrix

| From → To | Protocol / pattern | Data exchanged | Mode |
|-----------|--------------------|----------------|------|
| Browser / Magento → Profile Service | HTTPS REST/JSON; Auth0 OIDC session | Registration, identity subject, company profile, role intent | **Synchronous** command |
| Profile Service → CRM API | HTTPS REST/JSON; OAuth 2.0 | Contact, Account, Membership, Onboarding, KYB metadata | **Synchronous** command |
| CRM API → Dataverse | Dataverse Web API (OData v4); Entra app identity | Upsert rows + relationships; alternate keys | **Synchronous** internal |
| Magento → Catalog adapter → CRM API | Magento REST / webhook / export; bulk pagination | Product, SKU, category, price/listing, store scope, external IDs | **Bulk + incremental** |
| CRM API → Azure Service Bus | AMQP 1.0; topic/subscription; JSON envelope | Domain events (`eventId`, `aggregateId`, `correlationId`, …) | **Asynchronous** event |
| Backoffice → CRM API | HTTPS REST; Entra ID OAuth 2.0 | KYB decision, reason, reviewer, audit context | **Sync** command (+ event after commit) |
| Financing Portal → CRM API | HTTPS REST; service + user/company context | Company lookup, KYB status, financing reference upsert | Sync query/command |
| Financing Portal ↔ lender | HTTPS REST; OAuth/mTLS/signed webhooks | Application refs, decision, offer, disbursement status | Sync submit + **async** callback |
| Services → Docs / OTP / Notify | HTTPS REST and/or Service Bus | File refs, OTP, templates, delivery result | Sync request + async delivery |

## Example logical API contracts

| Operation | Caller → receiver | Purpose |
|-----------|-------------------|---------|
| `POST /api/v1/registrations` | Magento → Profile Service | Start customer/company onboarding |
| `POST /api/v1/companies` | Profile Service → CRM API | Upsert Account + membership |
| `POST /api/v1/onboarding/{id}/submit` | Profile Service → CRM API | Submit onboarding / KYB metadata |
| `POST /api/v1/public/financing-enquiries` | Public form → CRM API | Create Lead after validation + consent |
| `GET /api/v1/companies/{id}/summary` | Financing Portal → CRM API | Company + KYB + authorized roles |
| `POST /api/v1/financing/references` | Financing Portal → CRM API | CRM-visible financing summary |
| `POST /api/v1/catalog/products:upsert` | Catalog adapter → CRM API | Magento product upsert by external ID |
| `POST /api/v1/kyb-cases/{id}/decision` | Backoffice → CRM API | Approve / reject / request-info |
| `POST /api/v1/partners/{partner}/callbacks` | Lender → Financing integration | Decision/offer/disbursement callback |

### Example financing enquiry payload

```json
{
  "fullName": "Ahmed Ali",
  "companyName": "ABC Trading LLC",
  "email": "ahmed@example.com",
  "requestedAmount": 100000,
  "currency": "AED",
  "purpose": "Inventory purchase",
  "consent": true,
  "source": "HAL_PUBLIC_WEB"
}
```

## Event delivery rules

- Assume **at-least-once** delivery.
- Consumers must be **idempotent** (`eventId` / aggregate version).
- Use Outbox (same transaction as business change) → dispatcher → Service Bus.
- **DLQ + replay** are mandatory for operational recovery.
- Do not publish documents or unnecessary PII in events — publish secure references.

## Identity & authorization (summary)

| Actor | Auth | Authorization |
|-------|------|---------------|
| Public visitor | CAPTCHA / rate-limit; contact verification after submit | Minimal enquiry only |
| Customer | Auth0 OIDC | CRM API resolves Auth0 subject → Contact + Membership + role |
| Staff | Entra ID + MFA / CA | Backoffice roles (Reviewer, Approver, Support, Auditor) |
| Integrations | Client credentials / managed identity | Least-privilege Dataverse app user — not System Administrator |
| Lenders | Partner OAuth / mTLS / signed webhooks | Dedicated connector; replay protection |

See the PDF for illustrated flows and Magento EAV → Dataverse mapping detail.

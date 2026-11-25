# Task 1 — Dataverse Entity Design

Use standard Dataverse tables where they fit the CRM domain, and custom tables (`hal_` prefix as example) for HAL-specific lifecycle and junction data.

## Conceptual model

```
Account (Company) ──1:N── Company Membership ──N:1── Contact (Person)
       │
       ├──1:N── Onboarding Application ──1:N── KYB Case
       │                                      ├──1:N── KYB Check
       │                                      ├──1:N── Document Reference
       │                                      └──1:N── Review History
       │
       └──1:N── Financing Summary ──N:1── Financing Partner

Lead (public finance enquiry) ──0..1──► Account / Contact (after qualification)

Product ◄──N:M──► Category   (via Product-Category junction; Magento catalog copy)
```

## Core tables

| Table | Key columns (proposed) | Purpose / ownership |
|-------|------------------------|---------------------|
| **Account** (standard) | `accountid`, name, accountnumber, `hal_magentoCompanyId`, `hal_tradeLicenseNumber`, `hal_businessStatus`, `hal_kybStatus` | Company master. CRM/Dataverse owned. |
| **Contact** (standard) | `contactid`, name, email, mobile, `hal_auth0Subject` | Person master. Store Auth0 subject — not credentials. |
| **hal_CompanyMembership** | account + contact lookups, role, isPrimary, status, authorityLevel | Person ↔ company junction. Roles: Buyer, Seller, Owner, Company Admin, Finance Applicant. |
| **hal_OnboardingApplication** | account, submittedOn, status, applicationNumber, sourceChannel | Onboarding attempt / status. |
| **hal_KYBCase** | account, status, riskTier, reviewedBy, decisionOn, expiryOn | Verification outcome; re-verification / expiry. |
| **hal_KYBCheck** | kybCase, checkType, result, providerReference, checkedOn | Individual checks (licence, shareholders, etc.). |
| **hal_DocumentReference** | account, kybCase, documentType, secureFileId, status | Pointer to Document Service — not raw files in Dataverse. |
| **hal_ReviewHistory** | kybCase, action, reason, actor, actionOn | Append-only business audit of approve / reject / request-info. |
| **Lead** (standard) | subject, contact fields, `hal_requestedAmount`, currency, purpose, consent, leadsource | Public financing enquiry only after submission. |
| **hal_FinancingSummary** | externalApplicationId, account, partner, amount, status, portalUrl | CRM-visible reference only. Portal owns full application. |
| **hal_FinancingPartner** | partnerCode, name, status, integrationProfileReference | Partner directory — no credentials in Dataverse. |
| **Product** / catalog tables | Magento product ID (alternate key), SKU, name, status | Synchronized catalog **copy**. Magento remains master. |
| **hal_CatalogCategory** + **hal_ProductCategory** | Magento category IDs, parent, junction | Category tree + N:M product-category mapping. |

## Relationship rules

| Relationship | Cardinality | Why |
|--------------|-------------|-----|
| Account → Membership | 1:N | Many users per company; membership carries role/authority. |
| Contact → Membership | 1:N | One person may belong to multiple companies (do not rely only on `parentcustomerid`). |
| Account → Onboarding → KYB | 1:N / 1:N | Keep each submission and review cycle. |
| Product ↔ Category | N:M | Preserve Magento multi-category assignments. |
| Account → Financing Summary | 1:N | CRM visibility only; portal owns detail. |
| Lead → Account/Contact | 0..1 after qualification | Anonymous enquiry can exist before verified company. |

## Alternate keys (upsert / de-dupe)

- Account: `hal_externalCompanyId` (when supplied)
- Contact: `hal_auth0Subject`
- Product: `hal_magentoProductId` and/or SKU
- Category: `hal_magentoCategoryId`

## Ownership boundary

| Domain | System of record |
|--------|------------------|
| Company, contact, membership, KYB, CRM lead | **Dataverse** (via CRM API) |
| Products, cart, checkout, orders | **Magento** |
| Full financing application | **Financing Portal** |
| Credit decision / facility | **Lending partner** |

Do **not** make Dataverse master for Magento orders or lender credit decisions. Store references, status, and CRM-relevant summaries only.

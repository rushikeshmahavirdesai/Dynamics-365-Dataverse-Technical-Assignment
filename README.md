# Dynamics 365 & Dataverse — Technical Assignment

**Candidate submission:** HAL UAE B2B marketplace & financing platform  
**Role:** Principal / Lead Developer (Dynamics 365 & Dataverse)

## Design in one sentence

Keep the existing HAL customer channels and Azure microservices working; introduce **Dataverse** as the CRM system of record behind the existing **CRM API**, while **Magento** remains the commerce and product-catalog master and lending partners retain credit-decision ownership.

## What’s in this repo

| Path | Contents |
|------|----------|
| [docs/HAL_D365_Dataverse_Architecture.pdf](docs/HAL_D365_Dataverse_Architecture.pdf) | Full 12-page proposed target-state architecture |
| [docs/01-entity-design.md](docs/01-entity-design.md) | Dataverse entity model (Task 1) |
| [docs/02-integration-architecture.md](docs/02-integration-architecture.md) | Integration diagram, protocols, sync vs events (Task 2) |
| [docs/03-assumptions.md](docs/03-assumptions.md) | Explicit assumptions and trade-offs |

## Task coverage

1. **Entity design** — Dataverse data model for companies, contacts, membership, KYB, catalog copy, and financing visibility.  
2. **Integration architecture** — How Magento, Profile Service, CRM API, Dataverse, Admin Backoffice, Financing Portal, Auth0/Entra ID, and Service Bus communicate (protocols, data, sync vs async).

## Design principles

- **Preserve channels** — Magento, Profile Service, and Financing Portal do not call Dataverse directly.
- **Clear ownership** — CRM in Dataverse; catalog/orders in Magento; full loan workflow in Financing Portal; credit decision with the lender.
- **Capture intent** — Create a financing Lead only when a visitor submits an enquiry or a business deliberately starts an application.

## How to review

1. Start with this README for the summary and principles.  
2. Read `docs/01-entity-design.md` and `docs/02-integration-architecture.md` for the two required deliverables.  
3. Open the PDF for the full illustrated design (journeys, ERD-style model, API/event examples, security, migration).

Prepared as a proposed architecture from the interview brief. Exact production endpoints and partner contracts are illustrative and should be validated with system owners.

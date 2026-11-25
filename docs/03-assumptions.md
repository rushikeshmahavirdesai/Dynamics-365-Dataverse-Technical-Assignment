# Assumptions, Trade-offs & Reliability

## Explicit assumptions

1. The **CRM API remains the stable integration façade**; Dataverse becomes the backing CRM store after a phased migration.
2. **Profile Service** owns registration/onboarding **orchestration**, not the authoritative CRM database.
3. **Magento** remains commerce and catalog master. Adobe Commerce B2B Company APIs are used only if that module is enabled; otherwise HAL uses storefront projection / custom APIs.
4. Public financing form creates a **Lead only after submission**, contact/anti-spam validation, and consent — not on page view, login, or cart.
5. **Financing Portal** owns the full application; Dataverse stores CRM-relevant lead/reference/status. **Lenders** own underwriting and credit outcomes.
6. Exact Magento, Auth0, lender, and existing CRM API contracts were not in the brief — endpoint names and payloads in this submission are **proposed examples** to validate with system owners.
7. “Seller Workspace” / multi-vendor seller UX depends on the installed Magento marketplace module — confirm before assuming navigation.

## Architecture trade-offs

| Choice | Benefit | Cost |
|--------|---------|------|
| Keep CRM API as façade | Avoids rewriting every channel | Extra service hop; API ownership required |
| Events for downstream work | Decouples consumers | Idempotency, replay, monitoring, DLQ ops |
| Keep orders in Magento | Avoids duplicating commerce transactions | Finance use cases may need a consented order summary |
| Keep full loan in Financing Portal | Preserves lender-specific workflow | Dataverse has less detail; depends on status sync |

## Reliability & migration (design highlights)

| Concern | Approach |
|---------|----------|
| Idempotency | Idempotency keys on commands; alternate keys on external IDs; consumers store processed `eventId` |
| Reliable events | Outbox in same Dataverse transaction; dispatcher publishes to Service Bus; no distributed TX across both |
| Retries / DLQ | Exponential backoff; dead-letter poison messages; controlled replay; alerts on age/count |
| Consistency | Reconcile by external IDs + modified timestamps across masters |
| Initial migration | Profile CRM data → dedupe → map IDs → load Accounts/Contacts/Memberships/KYB history → reconcile counts |
| Catalog scale | Bulk sync with pagination; incremental events; respect API throttling; preserve store scope where needed |
| Observability | Correlation ID end-to-end; logs/traces/metrics for API latency, sync lag, event failures, KYB backlog |
| Availability | Keep storefront up if CRM API is briefly down: queue/retry registration where safe; never falsely show KYB approval |

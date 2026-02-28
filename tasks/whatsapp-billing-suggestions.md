# WhatsApp/Twilio Billing Suggestions for hay-backoffice

> These are suggestions for future billing implementation in hay-backoffice. No action needed now.

## Context

- WhatsApp integration in hay-core uses Twilio as the provider
- Two deployment models exist:
  - **Self-hosted**: Users bring their own Twilio credentials (`.env`), billing is between them and Twilio
  - **Managed (Hay Cloud)**: Hay provides Twilio as a service, needs usage tracking + billing

## Billing Considerations for Managed Version

### 1. Usage Tracking

- Track per-organization message counts (inbound + outbound)
- Track template vs freeform messages separately (different Meta costs)
- Track media messages (may have different pricing)
- Store daily/monthly aggregates for billing cycles

### 2. Cost Structure

- **Twilio per-message fee**: ~$0.005/msg (both directions)
- **Meta template fees**: ~$0.0034/msg for utility, varies for marketing
- **Freeform replies** (within 24h window): Free from Meta's side
- Consider markup strategy: pass-through + flat fee vs. per-message markup

### 3. Suggested hay-backoffice Features

- **Usage dashboard**: Show organizations their WhatsApp message usage
- **Billing automation**: Monthly invoice generation based on usage
- **Rate limiting**: Per-org message limits based on plan tier
- **Usage alerts**: Notify orgs approaching limits
- **Cost allocation**: Track Hay's Twilio costs per organization

### 4. Database Suggestions

- `whatsapp_usage` table: org_id, date, inbound_count, outbound_count, template_count, freeform_count
- Aggregate via cron job (similar to existing hay-backoffice automations)

### 5. Integration Points

- hay-core WhatsApp plugin should emit usage events or expose a usage API
- hay-backoffice polls or receives webhooks for usage data
- Billing tied to organization plan in Twenty CRM or a dedicated billing service

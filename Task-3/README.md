# Task 3 - Integration Design

## 1. End-to-End Integration Architecture

The consultation form is submitted from the landing page and processed by a backend service. The backend acts as the single integration point for all downstream systems instead of allowing the frontend to communicate with multiple external services directly. This keeps the client application simple, improves security by protecting API credentials, and provides a single place to handle validation, logging, retries, and error handling.

```text
Patient
    │
    ▼
Landing Page
(HTML / CSS / JavaScript)
    │
    ▼
Backend API
(Form Validation & Business Logic)
    │
    ├──────────────► HubSpot CRM
    │                 • Create or update contact
    │                 • Set lead properties
    │
    ├──────────────► Karix WhatsApp API
    │                 • Send confirmation message
    │
    └──────────────► Google Ads Conversion
                      • Record successful conversion

---

## 2. Technology Decisions

### Selected Approach

A backend service using the **HubSpot CRM API** for contact management, the **Karix WhatsApp Business API** for messaging, and the **Google Ads Conversion API** for conversion tracking.

### Why this approach?

The landing page already uses a custom form, so sending submissions through a backend service provides more control than embedding a HubSpot form or relying on third-party automation platforms.

Using direct API integrations allows the backend to validate incoming data, apply business rules, handle retries, and manage failures in one place.

For contact management, the backend should first search HubSpot using the submitted phone number. If a matching contact exists, that record should be updated. Otherwise, a new contact should be created. This avoids relying on HubSpot's default email-based deduplication, which is not suitable for a form that only collects a name and phone number.

This approach keeps the integration logic within the application's control and makes future changes easier without modifying the frontend implementation.

---

## 3. Failure Handling

### Biggest Failure Point

The biggest risk in this workflow is a partial failure after the consultation request has been accepted by the backend.

For example, the contact may be successfully created in HubSpot, but the WhatsApp API request could fail due to a temporary network issue or a third-party service outage. In this situation, the CRM contains the patient record while the confirmation message is never delivered, leaving the workflow in an inconsistent state.

### Handling Strategy

To reduce the impact of partial failures, each integration should be executed independently with proper error handling and logging.

If a downstream service fails:

- Record the failure in application logs.
- Retry the failed API request using a limited retry policy.
- Prevent duplicate contact creation by using the existing HubSpot contact if a retry occurs.
- Notify the operations team if the retries continue to fail so that manual follow-up is possible.

This approach improves reliability without affecting the patient's original form submission.

---

## 4. WhatsApp Delivery SLA

The confirmation message should be sent immediately after the contact has been successfully processed in HubSpot. Under normal conditions, this allows the patient to receive the confirmation within the required two-minute window.

To improve reliability, the backend should monitor the status of each WhatsApp API request and log any failed or delayed deliveries.

If message delivery fails due to a temporary issue, the backend should retry the request a limited number of times before marking it as failed. Repeated failures should be logged and flagged for manual follow-up so that affected patients can still be contacted.

Monitoring delivery status and retrying transient failures helps maintain the expected delivery time while providing visibility into issues that require operational attention.
# Task 1 - GTM Event Tracking Schema

## Scope

This document defines the proposed event tracking strategy for OrthoNow's website based on the requirements provided in the assignment. The objective is to capture meaningful user interactions, establish a consistent event tracking structure, and provide reliable data for Google Tag Manager (GTM), Google Analytics 4 (GA4), and Google Ads.

The schema is designed with the assumption that user interactions are captured by the frontend application using `window.dataLayer.push()`, while GTM is responsible for consuming those events and forwarding them to the required marketing and analytics platforms.

---

## 1. Design Principles

The following principles were used while designing the event schema:

- Track meaningful user interactions instead of every click.
- Follow a consistent snake_case naming convention for all events and parameters.
- Keep event names descriptive and reusable across analytics platforms.
- Avoid sending personally identifiable information (PII) through analytics events.
- Use standardized parameters wherever possible to simplify reporting and audience creation.
- Fire events only after a user action has been successfully completed.
- Keep frontend event generation and GTM tag configuration as separate responsibilities.

---

## 2. GTM Event Tracking Schema

The events below cover the primary user interactions mentioned in the assignment. Each event has a clear purpose, a consistent naming convention, and enough context to make the data useful in GA4 without collecting unnecessary information.

| Event Name | Trigger Type | Key Parameters | Purpose | GA4 Report / Audience |
|------------|-------------|---------------|---------|-----------------------|
| booking_started | Custom Event (`dataLayer`) | clinic_location, specialty, booking_source | User begins the consultation booking flow. | Funnel Exploration |
| booking_step_completed | Custom Event (`dataLayer`) | step_number, step_name, clinic_location | Measure progress through each booking step. | Funnel Exploration |
| booking_completed | Custom Event (`dataLayer`) | clinic_location, specialty, booking_source | Successful consultation booking. | Conversions |
| call_now_clicked | Click Trigger | page_location, clinic_location, button_position | Measure intent to contact a clinic by phone. | Events |
| whatsapp_chat_opened | Click Trigger | page_location, clinic_location, device_type | Measure engagement with the WhatsApp support channel. | Events / Audience |
| patient_guide_requested | Form Submission | guide_name, page_location, lead_source | Track successful patient guide requests. | Lead Generation |
| patient_guide_downloaded | Link Click | guide_name, file_type, page_location | Measure successful PDF downloads after form submission. | Events |
| clinic_page_viewed | Page View | clinic_location, city, page_path | Compare engagement across clinic locations. | Pages and Screens |
| blog_article_viewed | Page View | article_title, article_category, author | Measure traffic to educational content. | Pages and Screens |
| blog_scroll_depth | Scroll Trigger (75%) | article_title, scroll_percentage, article_category | Identify readers who meaningfully engage with blog content. | Engagement |


---

## 3. Booking Funnel Tracking

The appointment booking flow consists of three user-facing steps. Instead of tracking only the final booking confirmation, each step is tracked independently. This makes it possible to identify where users leave the booking flow and helps both the product and marketing teams understand where improvements are needed.

```text
Booking Started
      │
      ▼
Step 1
Select Clinic Location + Specialty
      │
      ▼
Step 2
Enter Name + Phone + Preferred Date
      │
      ▼
Step 3
Confirm Booking
      │
      ▼
Booking Completed
```

| Booking Step | Event Fired | Trigger | Purpose |
|--------------|------------|---------|---------|
| Booking Started | `booking_started` | User opens the booking flow | Measures the total number of users entering the funnel. |
| Step 1 Completed | `booking_step_completed` | User successfully completes Step 1 | Identifies users who selected a clinic and specialty. |
| Step 2 Completed | `booking_step_completed` | User successfully completes Step 2 | Measures users who submitted their personal details. |
| Step 3 Completed | `booking_completed` | User successfully confirms the booking | Represents a successful consultation booking. |

### Funnel Reporting in GA4

The above events can be used to build a Funnel Exploration in GA4 using the following sequence:

1. `booking_started`
2. `booking_step_completed` (Step 1)
3. `booking_step_completed` (Step 2)
4. `booking_completed`

Comparing the number of users at each stage immediately highlights where users abandon the booking flow. This makes it easier to identify friction points and measure the impact of future UX or marketing changes.


---

## 4. Booking Form `dataLayer` Implementation

Each booking step pushes a structured event into the `dataLayer` only after the current step has been successfully completed. This ensures that GTM receives accurate funnel data and prevents users from being counted if they abandon a step midway.

### Step 1 - Clinic & Specialty Selected

Fired immediately after the user completes the first booking step.

```javascript
window.dataLayer.push({
    event: "booking_step_completed",
    step_number: 1,
    step_name: "clinic_specialty_selected",
    clinic_location: "Indiranagar",
    specialty: "Orthopaedics"
});
```

### Step 2 - Patient Details Submitted

Fired after the user successfully submits their personal details.

```javascript
window.dataLayer.push({
    event: "booking_step_completed",
    step_number: 2,
    step_name: "patient_details_submitted",
    clinic_location: "Indiranagar",
    specialty: "Orthopaedics"
});
```

### Step 3 - Booking Confirmed

Fired only after the appointment has been successfully confirmed.

```javascript
window.dataLayer.push({
    event: "booking_completed",
    step_number: 3,
    step_name: "booking_confirmed",
    clinic_location: "Indiranagar",
    specialty: "Orthopaedics"
});
```
Implementation Note: These events are generated by the frontend application. GTM listens for these custom events but does not create them automatically.\


---

## 5. Google Ads Conversion Selection

### Selected Conversion

`booking_completed`

### Why this event?

The primary goal of the campaign is to generate consultation bookings, not just user engagement. While actions such as clicking the call button, opening WhatsApp, or downloading the patient guide indicate interest, they do not guarantee that a consultation request has been completed.

Using `booking_completed` as the primary Google Ads conversion allows campaign optimisation to focus on users who successfully complete the booking journey. This produces cleaner conversion data and aligns advertising performance with the business objective of generating qualified patient enquiries.

Other tracked events continue to provide valuable behavioural insights in GA4, but they should be treated as supporting events rather than primary advertising conversions.


---

## 6. Frontend vs GTM Responsibilities

The tracking flow for the booking journey is shared between the frontend application and Google Tag Manager. The frontend is responsible for identifying when a meaningful user interaction has occurred, while GTM is responsible for consuming those events and forwarding them to the required analytics and marketing platforms.

```text
User Action
     │
     ▼
Frontend Application
(window.dataLayer.push)
     │
     ▼
Google Tag Manager
(Triggers & Tags)
     │
     ├────────────► Google Analytics 4
     │
     └────────────► Google Ads
```

| Frontend Application | Google Tag Manager |
|----------------------|--------------------|
| Detects meaningful user interactions. | Listens for custom events pushed into the `dataLayer`. |
| Pushes structured event data using `window.dataLayer.push()`. | Matches events using configured triggers. |
| Defines the event name and parameters. | Reads the event data and fires the appropriate tags. |
| Ensures events are fired only after successful user actions. | Sends the event to GA4, Google Ads, or other connected platforms. |

### Developer Note

Google Tag Manager does not automatically understand application state or multi-step forms. It can only respond to events that are made available through the `dataLayer`. For this reason, the frontend application is responsible for deciding **when** an event should be pushed and **what** information should be included. GTM then uses those events to trigger analytics and marketing tags without requiring changes to the application code.
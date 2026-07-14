# Booking session max reservation

> Source: [Confluence — Booking session max reservation](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/980221982/Booking+session+max+reservation)

Related docs:

- [ENG-964919303](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/964919303)
- [Driver probation check in Ticket booking session](driver-probation-check-in-ticket-booking-session.md)
- [Booking model configuration](driver-match-intelligent-booking-session/booking-model-configuration.md)

Booking session max reservation is based on `booking_session_template.reservation`.

Real booking session max reservation is the **min** of:

- Booking session max reservation
- Global booking limit
- Global booking limit (new driver)
- Booking session wave limit
- Driver probation

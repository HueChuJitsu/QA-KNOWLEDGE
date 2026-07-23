# Driver probation check in Ticket booking session

> Source: [Confluence — Driver probation check in Ticket booking session](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/973275397/Driver+probation+check+in+Ticket+booking+session)

Driver probation check is applied for each corresponding Booking session.

## 1. How to create a Driver probation?

Go to [dispatch.staging.gojitsu.com/driver-probations](https://dispatch.staging.gojitsu.com/driver-probations).

## 2. Related types of Driver probation

- **Delayed:** not allow the Driver to book within `{booking_start_time + delayed_seconds}`.
- **Blackout:** not allow the Driver to book a ticket.
- **Limit Reservation:** max tickets a driver can hold (including booked and assigned tickets) in a booking session.
- **Reduced route:** `Booking session limit − Reduced route` = max tickets a driver can hold (including booked and assigned tickets) in a booking session.
  - Booking session limit of a driver = Booking session limit, or = `booking_session.attributes.max_reservations` (related to boost) (higher priority), and will be increased by `booking_session.attributes.max_reservations_boost`. After getting the final result here, the probation check starts (if any).

**Note:**

- If a driver's max reservation is limited by a Driver probation, the driver cannot book more tickets even though the driver already completed tickets of that Booking session.
- The system compares Limit Reservation and (Booking session limit − Reduced route) if the Driver has both types of Probation, and takes the smaller value as the Driver max reservation.

## 3. Probation time period

- **Timezone:** the timezone of the browser used to create that driver probation.
- The system compares the time the Driver clicks the **[Book ticket]** button with the Probation time period to know if the Probation is still valid.

## 4. Driver probation cache logic in Driver app

**Pre-condition:**

- Create probation for a new Driver (new = that Driver does not have any driver probation before; can create many Driver probations for 1 driver).
- Booking session template has a record in `item_metadata`:

```json
{
  "_id": ObjectId("65d6c317cbb489222ae6f04a"),
  "id": "c1d42f62-b52b-4b69-99ed-a82714167d41",
  "scope": "APP_FEATURES",
  "owner": "BST_a4afe276-7a28-45ab-a7b1-ae6e7518698d",
  "values": {
    "check_probation": {
      "type": "java.lang.Boolean",
      "value": "true"
    }
  }
}
```

Upon the Driver clicking the **[Book ticket]** button on the Driver app, the Driver app gets _all_ probation info of the Driver, then saves it in the Driver-app cache for 30 mins. Within 30 mins, the Driver probation info on the Driver app does not change even if the Dispatcher creates a new Probation or edits a Probation for that Driver. After 30 mins, the cache is cleared; when the Driver clicks **[Book ticket]** again, a new cache for the driver is created. Each driver has a different cache end time, based on the time they click the **[Book ticket]** button (= cache start time).

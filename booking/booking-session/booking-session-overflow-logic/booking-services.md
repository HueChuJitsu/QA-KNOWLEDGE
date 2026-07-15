# Compare booking services

> Source: [Confluence — Compare booking services](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/877297677/Compare+booking+services)

Related documents:

- [Booking request flow diagram (draw.io)](https://app.diagrams.net/#G1xPImLX35ow9zJBv-ZHqtllMRDshnNvyH)
- [Comparison spreadsheet](https://docs.google.com/spreadsheets/d/1AnrQRIY07fC2CPzw2MQrdjutPtksxhaHg-4B4iKI4pQ/edit#gid=0)

## Booking service v0

- Used Redis Ticket set info to book/unbook a ticket.
- Creates a real ticket.

## Booking service v1 (only change in Redis)

- Used Redis Semaphore and Ticket map info to book/unbook a ticket instead of Ticket set.
- Redis Ticket map and Semaphore are reset TTL whenever refreshing the BS, adding/deleting a ticket to a ticket book, or booking/unbooking a ticket (whether unbooked automatically due to lack of eta, or manually).
- When a Driver request to book a ticket comes and the semaphore does not exist, only 1 request can pass the semaphore check to continue booking the ticket.
- If the Redis Ticket map expires before the Redis Semaphore and there are still tickets available for the ticket book, the error message "No ticket available…" comes out, as that error is based on the Ticket map → the Ticket map should have TTL >= Semaphore expiration TTL.
- Creates a real ticket.
- Set up to create a booking session v1 in the expected day and region: [Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/booking_service/new_booking_service_regions/).
  - Booking session v1 will have a corresponding record in `item_metadata` after creation.

## Booking service v2 (change in both Redis and Database)

- Creates a virtual ticket → gets info from the Ticket book and disables the Ticket note feature at this time.
- A real ticket is created when booking / assigning a ticket.
- When unbooking / unassigning, the real ticket is deleted.
- Book/unbook and assign/unassign/reassign are based on Redis Ticket left.
- Set up to create a booking session v2 in the expected day and region: [Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/apps/booking_service/new_booking_service_regions_v2/).

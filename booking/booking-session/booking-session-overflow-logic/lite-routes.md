# Lite routes

> Source: [Confluence — Lite routes](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/844660838/Lite+routes)

_This docs is to provide information about Lite routes._

Data savings:

- [Looker Studio report](https://lookerstudio.google.com/reporting/4a52ad77-2f14-47fc-be65-777a56578589/page/wPulD)

_Lite routes means route that contains shipment whose dropoff location is around Warehouse. These routes will normally have_ `actual tour cost` < `min of Cost formula used in regions`, so if we still use `Cost formula with NO Min value` we will need to pay Driver more — so we enhance to use formula with no min for these routes to save cost.

- New ticket book with `attributes.is_lite = true` needs to be added to the booking template to be able to create a booking session with a ticket book whose `is_lite = true`.
- Currently system considers routes in ticket book whose `is_lite = true` to be Lite routes (`is_lite` param that is **not** in `attributes` section) — check in collection `ticket_book`.
- Assignments which satisfy conditions to be added in a ticket book and have `total time (pickup time + travel time + service time) < x mins` (set up in [Consul](https://consul.gojitsu.sdm.network/ui/dc1/kv/prod/traffic/attributes/max_delivery_time_lite_route_by_region/edit)) will be added automatically to the respective ticket book with `is_lite = true` by worker `RefreshBookingSessionWorker` or by clicking the **[Refresh assignment]** icon (1). Those routes will not be added to other zones although matching conditions to be added. (Condition to be added in a Ticket book: refer to point 4 in [Advance ticket booking doc](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/845119492).)
  - As conditions to add assignment to Lite zone or normal zone may be the same, just differing in total-time condition, and system adds assignment to each ticket book based on order of ticket book in booking template → need to set up the Lite zone above the normal zone with same conditions in the Booking template.
- New column in table `assignments`: `is_lite`. This column is updated to `true` when a Route is added to a ticket book whose `is_lite = true`.
  - Currently this column is used to support the system to generate correct sort label for shipment. Shipment in lite route will get sort label template from `warehouse_config.lite_route_label_version`. (No longer in use.)
  - The column is intended to validate route to update tour cost when pickup succeeds (price displayed in Accept route screen). (To be enhanced later.)
- Set up to recalculate traffic when pickup route successfully: `collection item_metadata, owner: RG_{region}`.
- Assignment event when updating tour cost when there is a change to number of shipments in Route or when Pickup succeeded: Finance formula = Lite route formula ([Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1/kv/staging/finance/attributes/finance_model/edit)).

**Updated:**

- Already updated to use `assignments.is_lite = true` to validate route to update tour cost when pickup succeeds.
- Currently system considers routes in ticket book whose `assignments.is_lite = true`.
- `Assignment.tour_cost` is reset to the Lite route tour cost when splitting a route, removing a shipment from a route, or when refreshing the BS (get from `pricing_assignments.tour_cost` with formula set up on [Consul](https://consul-staging.gojitsu.sdm.network/ui/dc1-staging/kv/staging/finance/attributes/finance_model/edit)) (?? recheck when have time).
- `Assignment.tour_cost` is **not** reset to the Lite route tour cost when the assignment is added to the BS manually.
- Already updated Lite route logic for Regular booking session.

### `update_lite_label`

If `true`, when routes are added to the BS by worker `RefreshBookingSessionWorker` or by clicking the **[Refresh assignment]** icon, their prefix is updated per the mapping in `warehouse_config.label_lite_route_prefixes`.

### `update_layout`

This covers the case where a sort session is created before Routes are added to the Booking session → Lite routes would not be added to specific slots for them.

- If `true`, Inbound layout is regenerated whenever the booking session is refreshed (by worker `RefreshBookingSessionWorker` or by clicking the **[Refresh assignment]** icon) if the current sort session with the same solution changes fewer than 4 times (not including action `create_layout`).
  - If the Inbound sort session already changed more than 4 times and an error case happens → the in-charge person will need to contact Tech support in reality.
- If `false`, the sort session is not regenerated whenever the booking session is refreshed.

### Delivery time threshold for Lite route config (relates to (1))

Other than the Consul set up, we can set up the delivery time threshold for a Lite route in `item_metadata` for each region (`owner = RG_{region}`) or each warehouse (`owner = WH_{warehouse}`), or in `Prefix management` (no UI yet).

Priority level of each set up, highest to lowest: `Prefix management` → `Warehouse` → `Region` → `Consul`.

### Number of lite routes config

We can set `number_of_lite_routes` on `booking_session_template.attributes` or `zone.attributes`.

The total of lite routes would not be larger than `booking_session_template.number_of_lite_routes` + 1, and lite routes of each zone would not be larger than `number_of_lite_routes` of each zone.

- If `number_of_lite_routes` of the booking session is not set, it means no limit.
- If `number_of_lite_routes` of the booking session is set (e.g. 27) and `number_of_lite_routes` of a zone is set (e.g. zone 2: 5), the maximum lite routes of other zones would be 22 + 1.

**Note:**

- If a Route is added manually to a Lite zone, no Lite zone config is validated for those routes.
- If a route is once added to a Lite zone, it will forever be a Lite route (no update of tour cost / label / `is_lite` when removing the Lite route from the Lite zone).

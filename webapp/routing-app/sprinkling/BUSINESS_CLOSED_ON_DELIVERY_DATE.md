# BUSINESS_CLOSED_ON_DELIVERY_DATE — One-Off Sprinkle Failure Reason

> **Ticket:** `ALT-1883` · **Env:** `staging` · **Region ví dụ:** `JFK / client 980` · **Core lib:** `sortation-bizlogic`

**Labels:** `routing` `sortation` `sprinkling` `business-hour` `ALT-1883`

---

## 1. Tổng quan

`BUSINESS_CLOSED_ON_DELIVERY_DATE` là một reason mới trong enum `OneOffFailure`, được thêm bởi **ALT-1883**. Reason này được sinh ra khi shipment đi vào `oneOffSprinkleV2` → `checkDate`, và `BusinessHourManager.getDeliveryEndTime()` trả về **empty** — tức là business đóng cửa vào đúng ngày giao hàng dự kiến (`target_delivery_date`).

Khác với case bị chặn ở guard trước khi vào sprinkle (in ZPL "Business closed do not sort"), reason này chỉ xuất hiện khi shipment đã **thực sự vào** `doOneOffSprinkleV2` và bị từ chối ở bước `checkDate`. Golden rule: nếu không có document trong `sprinkling_traces` thì lib chưa từng chạy — check này không áp dụng.

---

## 2. Nguồn dữ liệu

Business-open/closed được quyết định từ 2 nguồn, nằm ở 2 store khác nhau — luôn phải join tay khi debug:

| Trường | Store | Bảng/Collection | Ghi chú |
| --- | --- | --- | --- |
| `is_business` | Postgres | `shipment_extra` | Set bởi worker `ShipmentBusinessHourHandler` khi xử lý DAS event (residential=false), chỉ khi business-hour được **enable** cho region/client. Set tay `address_type = COMMERCIAL_BUILDING` **không** có tác dụng. |
| `business_hours[<dayOfWeek>].status` | Mongo | `deliverable_address` (qua `customer_profile.deliverable_address_id`) | So sánh với `OPEN_STATUS = "Open"`. |

Nếu `shipment_extra.is_business = false` → luôn coi là **open**, bỏ qua toàn bộ check business hours.

**Gotcha:** hai nguồn (`is_business` ở Postgres và `business_hours` ở Mongo) có thể lệch nhau (mismatch) → gây behavior sai. Khi setup data test, phải đảm bảo cả hai khớp.

---

## 3. Engine xử lý

- Lib: `BusinessHourManager` trong `routing-bizlogic`.
- Hàm chính: `getDeliveryEndTime()`, `canDeliverShipmentOnDate()`.
- `getDeliveryEndTime()` empty ⇒ business đóng cửa vào delivery date ⇒ `checkDate` trả `BUSINESS_CLOSED_ON_DELIVERY_DATE`.

Thứ tự check trong `checkDate` (chỉ tới đây sau khi đã qua hết các gate A–D):

1. **TOO_EARLY** — check trước tiên: `start.plusDays(days).isAfter(now) && inboundReceivedTs != null`
2. **BUSINESS_CLOSED_ON_DELIVERY_DATE** — `getDeliveryEndTime()` empty
3. OK → tiếp tục sang nearest-neighbor → overload check

Vì TOO_EARLY được check trước, nếu shipment vừa too-early vừa business-closed thì kết quả trả về là `TOO_EARLY`, không phải reason này.

---

## 4. Guard khác nhau theo luồng scan (điểm quan trọng nhất)

Có một guard business-hour đứng **trước** khi vào `oneOffSprinkleV2`, và guard này khác nhau tùy đường scan — quyết định reason `BUSINESS_CLOSED_ON_DELIVERY_DATE` có tái hiện được hay không.

| | Small parcel (`warehouse-api`) | Large (`inbound-api`) |
| --- | --- | --- |
| Guard điều kiện | `if (assignmentId == null && isClosed)` | `if (!isOpened)` — **bỏ qua assignmentId** |
| Redelivery (`assignmentId != null`) + business closed | **Bypass guard** → vào sprinkle → `checkDate` → **`BUSINESS_CLOSED_ON_DELIVERY_DATE`** | Vẫn bị chặn (in ZPL business) |
| No-route (`assignmentId == null`) + business closed | Chặn ("Business closed do not sort") | Chặn |

> **Hệ quả:** reason `BUSINESS_CLOSED_ON_DELIVERY_DATE` gần như **chỉ tái hiện được qua đường small-parcel redelivery** (`assignmentId != null`). Trên đường large, business closed luôn bị chặn ở guard trước, không bao giờ tới được one-off.

`client_service.enable_business_hour` phải bật thì business-hour check mới áp dụng (độc lập với `sprinklable`).

---

## 5. Playbook tái hiện

**Setup:**

1. Shipment ở dạng **redelivery small-parcel**: có `assignment_id`, stop `deliverShipment` = `FAILED`.
2. `shipment_extra.is_business = true`.
3. `deliverable_address.business_hours[<delivery day>].status = "Closed"`, DA linkage đúng (`customer_profile.deliverable_address_id` → đúng `da_id`).
4. `client_service.enable_business_hour = true`, `sprinklable = true`.
5. Scan vào đúng ngày mà business đóng cửa (đảm bảo không rơi vào TOO_EARLY trước).

**Không dùng:**
- Large scan (guard chặn trước, không ra được reason này).
- Shipment no-route business-closed (bị chặn với message khác, không vào sprinkle).

**Verify kết quả:** query `sprinkling_traces` theo `shipment_id`, check `failure_reason = "BUSINESS_CLOSED_ON_DELIVERY_DATE"`.

```js
db.sprinkling_traces.find({ shipment_id: <id> }).sort({ _id: -1 })
```

**Ví dụ đã dùng trong investigation:** shipment `70633002` — redelivery small, `is_business=true`, `Tue=Closed` → ra đúng `BUSINESS_CLOSED_ON_DELIVERY_DATE`.

---

## 6. Reference queries

### Postgres

```sql
-- Business flag
SELECT * FROM shipment_extra WHERE id = :id;

-- Client service (enable_business_hour)
SELECT sprinklable, enable_business_hour FROM client_service
WHERE client_id = :clientId AND region = :region AND _deleted IS NOT TRUE;

-- Upsert is_business để test
INSERT INTO shipment_extra (id, is_business, routing_earliest_dropoff_ts, routing_latest_dropoff_ts)
VALUES (:id, true, :earliest_ts, :latest_ts)
ON CONFLICT (id) DO UPDATE SET is_business = true, _updated = NOW();
```

### Mongo

```js
// Business hours của DA
db.customer_profile.find({ id: "<cp_id>" }, { deliverable_address_id: 1 })
db.deliverable_address.find({ id: "<da_id>" }, { business: 1, business_hours: 1 })

// Trace kết quả sprinkle
db.sprinkling_traces.find({ shipment_id: <id> }).sort({ _id: -1 })
```

---

## 7. Gotchas liên quan

- `is_business` **không** được set từ `address_type` — chỉ set bởi worker khi xử lý DAS event.
- Hai nguồn (`is_business` Postgres vs `business_hours` Mongo) lệch nhau → behavior sai, phải check cả hai khi debug.
- Guard business-closed khác nhau: small chỉ chặn khi `assignmentId == null`; large luôn chặn.
- Config/rule cache (RMapCache) — sau khi sửa Mongo/Postgres có thể cần đợi TTL hoặc restart service.

---

> *Tài liệu tổng hợp từ investigation staging (ALT-1883), tham chiếu `sortation-bizlogic` / `routing-bizlogic` HEAD hiện tại. Không post lên Confluence vì Atlassian connector chưa được authorize trong session này — nếu muốn publish thật, cần authorize qua `/mcp` hoặc claude.ai connector settings rồi yêu cầu lại.*

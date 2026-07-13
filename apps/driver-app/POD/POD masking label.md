# POD Package Label Masking (Global On/Off Flag)

> Documentation for a Driver App / POD Analyzer function, based on the standard `_template.md`.
> Source: [MOB-2996](https://gojitsu.atlassian.net/browse/MOB-2996) — [Pending Release] Allow on/off package masking feature in the AI detection model
> Related: [MOB-2966](https://gojitsu.atlassian.net/browse/MOB-2966) (dev ticket, Done Pending Release) | [MOB-2806](https://gojitsu.atlassian.net/browse/MOB-2806) / [MOB-2788](https://gojitsu.atlassian.net/browse/MOB-2788) (POD | Address model masking & Threshold)
> Component: POD Analyzer | Status: Code Review (pending release)

## 1. Description

Applies package label masking before the OCR detection step in the on-device AI detection model (from [MOB-2806](https://gojitsu.atlassian.net/browse/MOB-2806)):

- Android and iOS must apply package label masking **before** OCR detection. The masking process runs on the input image/frame first, removes or obscures the package label regions, and then passes the masked image/frame into OCR detection.
- The masking step is enabled consistently on both Android and iOS before any OCR text extraction or detection logic is executed.
- iOS processing order matches Android: package label masking → OCR detection → downstream model/threshold handling.
- iOS detection threshold matches Android at **0.7** — only OCR detection results with confidence greater than 0.7 are recorded in the event.
- If the model returns an empty string, the condition is marked as failed.

[MOB-2996](https://gojitsu.atlassian.net/browse/MOB-2996) puts this masking step behind a global flag so it can be enabled/disabled without an app release.

**Actors:**

- **Driver** — takes the POD photo at delivery; masking runs transparently on-device as part of photo analysis.
- **Ops / Admin** — turns package masking on or off globally via Consul remote config.

## 2. Business Flow

**Masking enabled (`enable_package_masking = true` — current default behaviour):**

1. Driver takes a POD photo on the drop-off screen.
2. On-device object detection identifies the package and its bounding box.
3. Package label masking is applied to the image/frame **before** OCR detection (same order on Android and iOS: masking → OCR detection → downstream model/threshold handling).
4. OCR detection runs on the masked image; results with `detection_confidence` (in `ocr_rec.result`) > 0.7 are recorded in the `photo_analyzed` event.

**Masking disabled (`enable_package_masking = false`):**

1. Driver takes a POD photo as normal.
2. The masking step is skipped entirely — the raw image/frame is passed directly into OCR detection.
3. All downstream behaviour (thresholds, rule analysis, enforcement, upload) is unchanged.

**Exit conditions / guarantees:**

- The flag only gates the masking step — photo capture, validation flow, and stop completion are never blocked by the flag's value.
- Config is global (per Consul `mobile_app_config`), not per-assignment or per-client.

## 3. Spec / Rules

**Configuration:**

| Config key | Location | Default | Description |
|---|---|---|---|
| `enable_package_masking` | Consul `mobile_app_config` | `true` (masking is current behaviour) | Global on/off for package masking in the AI detection model (this feature, MOB-2996) |

**Rules (carried over from MOB-2806, unchanged by the flag):**

- Package label masking is applied **before** OCR detection on both Android and iOS.
- iOS OCR/model detection threshold = **0.7**, aligned with Android — only OCR detection results with confidence > 0.7 are recorded in the event.
- If the model returns empty string → the condition is marked as **failed**

## 4. QA / Test notes

**Required config (used in MOB-2806 verification):**

1. `RG_{region}_DELIVERY_pod_warn_use_conditional_soft_block` = `warn` in Client / Delivery Setting
2. `RG_{region}_DELIVERY_pod_analyzer_enforcement_policy` = `false` in Client / Delivery Setting
3. `enable_package_masking` on Consul `mobile_app_config` — toggle per scenario
4. `delivery_pod_rules` on Consul `delivery_assignment` must be set up

**Testcases:**

- `enable_package_masking = true` → take a POD photo with a shipment label visible → label masking is applied before OCR detection; masked image visible in the `photo_analyzed` event / uploaded photos.
- `enable_package_masking = false` → take the same POD photo → no masking applied; OCR detection runs on the raw image; downstream validation still works.
- Toggle the flag on Consul → behaviour switches globally without an app release (restart app / refresh config as required).
- OCR confidence → Unit number condition status = FAIL, the `detection_confidence` is not recorded in the event.
- OCR confidence in `ocr_rec.result` > 0.7 → Unit number condition status = PASS the `detection_confidence` is recorded in the event.
- Model returns empty / null → Unit number condition status = FAIL.
- Flag off must NOT affect: photo capture, soft/hard block enforcement, photo upload, or stop completion.

**Things to watch when testing:**

- Check the `photo_analyzed` event on the stop (Dispatch route view) for extracted OCR results and thresholds. There are two models: OCR Detection and OCR Recognition — if the confidence > 0.7, `ocr_rec.result` will have a `detection_confidence` value; otherwise `ocr_rec.result` will be empty.
- Check the `pod_reviews` collection in MongoDB to verify the Condition status and Confidence
- `https://external-portal.staging.gojitsu.com/pod-management/` (staging) will show the POD AI Detection review

# Delivery Photo Capture (POD) & On-device AI/ML validation

## Table of Contents

- [Overview](#overview)
- [User Stories](#user-stories)
  - [Story 1 — Take a mandatory delivery photo](#story-1--take-a-mandatory-delivery-photo)
  - [Story 2 — Photo is optional](#story-2--photo-is-optional)
  - [Story 3 — Retake a photo (soft block)](#story-3--retake-a-photo-soft-block)
  - [Story 4 — Blocked until photo passes (hard block)](#story-4--blocked-until-photo-passes-hard-block)
  - [Story 5 — Delete a photo](#story-5--delete-a-photo)
  - [Story 6 — Upload photo instead of capturing (QA)](#story-6--upload-photo-instead-of-capturing-qa)
- [When Is a Photo Required?](#when-is-a-photo-required)
- [App Behaviour — Step-by-Step Flow](#app-behaviour--step-by-step-flow)
  - [1. Accessing Photo Capture](#1-accessing-photo-capture)
  - [2. Taking a Photo](#2-taking-a-photo)
  - [3. Tutorial Delivery Routes and Photo Capture](#3-tutorial-delivery-routes-and-photo-capture)
  - [4. Photo Validation (AI/ML)](#4-photo-validation-aiml)
  - [5. Handling Validation Failures](#5-handling-validation-failures)
  - [6. Photo Upload](#6-photo-upload)
- [Offline Mode](#offline-mode)
  - [Photo Capture Offline](#photo-capture-offline)
  - [POD Analysis (ML Validation) Offline](#pod-analysis-ml-validation-offline)
  - [Summary](#summary)
- [Configuration Reference](#configuration-reference)
  - [Feature Flags](#feature-flags)
  - [Camera & Image Quality](#camera--image-quality)
  - [Masking (Privacy Blurring)](#masking-privacy-blurring)
  - [Validation & AI Rules](#validation--ai-rules)
  - [Enforcement Policy (per assignment/stop)](#enforcement-policy-per-assignmentstop)
  - [Upload & Storage](#upload--storage)
  - [QA / Testing](#qa--testing)
- [Internationalisation (i18n)](#internationalisation-i18n)
  - [Client-Side i18n](#client-side-i18n)
  - [Server-Driven Text and Machine Translation](#server-driven-text-and-machine-translation)
  - [Machine Translation Configuration](#machine-translation-configuration)
- [POD Validation Rules](#pod-validation-rules)
- [Related Code Locations](#related-code-locations)

---

## Overview

Delivery Photo Capture allows drivers to take photos of a package at the point of delivery as a Proof of Delivery (POD). This is one of the most critical functions in the delivery flow — it creates an auditable, timestamped record that a package was delivered to the correct location and in the expected state.

Photos are attached to the stop, uploaded to cloud storage, and can be validated in real-time by an on-device AI/ML model that checks whether the photo meets quality requirements (e.g. package is visible, label is readable).

---

## User Stories

### Story 1 — Take a mandatory delivery photo

As a driver, I want to take a photo of the delivered package so that I can confirm proof of delivery to the merchant and operations team.

Given I am on the drop-off screen for a stop that requires a photo
When I tap the Photo POD button
Then the camera opens with a guided overlay frame and an on-screen instruction
And I take the photo
Then the photo is saved and uploaded in the background
And I can proceed to complete the stop

---

### Story 2 — Photo is optional

As a driver, I want to optionally take a photo even when it is not required so that I can provide additional evidence of delivery.

Given I am on the drop-off screen for a stop where a photo is not required
When I tap the Photo POD button
Then the camera opens and I can take a photo
And after taking the photo I can still complete the stop without it if I choose not to

---

### Story 3 — Retake a photo (soft block)

As a driver, I want to be warned when my photo does not meet quality requirements so that I have a chance to retake it before completing the stop.

Given I have taken a photo that fails one or more validation rules
And the soft block policy is active
When the ML model analyses the photo
Then a warning dialog appears explaining that the photo may be missing key details
And I can tap "Take another photo" to retake it
When I have exceeded the maximum guided retake attempts
Then the warning becomes dismissible and I can proceed anyway

---

### Story 4 — Blocked until photo passes (hard block)

As an operations manager, I want to prevent drivers from completing a stop when the photo does not meet requirements so that proof of delivery is always valid and auditable.

Given I have taken a photo that fails one or more validation rules
And the hard block policy is active
When the ML model analyses the photo
Then a blocking screen appears listing each required rule with pass/fail status
And I cannot complete the stop
When I retake the photo and it passes all rules
Then the block is lifted and I can complete the stop
When I am unable to take a passing photo
Then I can contact dispatch to request a manual review and unblock

---

### Story 5 — Delete a photo

As a driver, I want to delete an incorrect photo from the gallery so that only valid photos are submitted for the stop.

Given I have taken a photo for a stop that is not yet completed
And the photo has been uploaded (has a remote URL)
And photo deletion is enabled
When I tap the delete button on the photo thumbnail
Then a confirmation dialog appears
When I confirm the deletion
Then the photo is removed from the stop

---

### Story 6 — Upload photo instead of capturing (QA)

As a QA engineer, I want to upload an existing image file instead of using the camera so that I can test the ML validation pipeline with controlled, repeatable inputs.

Given the `qa_scan_picker_enabled` flag is turned on
When I tap the Photo POD button
Then a file picker opens instead of the camera
And I select an image from the device
Then the selected image is processed and validated by the ML model exactly as a captured photo would be

---

## When Is a Photo Required?

Photo requirements are determined per shipment and per delivery outcome:

| Scenario | Behaviour |
|---|---|
| Shipment has `deliveryProofPhotoRequired = true` | Photo is mandatory to complete the stop |
| Shipment has `deliveryProofPhotoRequired = false` | Photo is optional (encouraged but not blocking) |
| Delivery fails and the selected failure reason has `photoRequired = true` | Photo is mandatory for that failure reason |
| Delivery fails and the selected failure reason has `photoRequired = false` | Photo is optional |

> The photo POD button is styled with the primary (accent) colour when a photo is required, and a muted colour when optional.

All POD checks (`photoPODPass`, `signaturePODPass`, `idScanPODPass`) must pass before the driver can complete a stop.

---

## App Behaviour — Step-by-Step Flow

### 1. Accessing Photo Capture

The driver taps the Photo POD button on the delivery card:

- If no photos have been taken: shows a camera icon.
- If photos exist: shows a thumbnail of the most recent photo plus a count badge.

The button is protected by a geofence guard — it can only be activated within the valid delivery window/location.

### 2. Taking a Photo

Tapping the button opens the Photo Gallery screen, from which the driver can:

- View existing photos for the stop.
- Tap Take Photo to open the camera.
- Delete a photo (with a confirmation dialog) — subject to the conditions below.

When the camera opens:

1. The app displays the highest-priority unsatisfied validation rule as an on-screen instruction (e.g. "Make sure the corners of the package are visible").
2. A guided overlay frame assists the driver in framing the shot correctly.
3. The driver takes the photo. Blur detection can warn (or silently flag) if the image is too blurry.
4. The photo is compressed before saving (configurable quality and dimensions).

#### Deleting a photo

A delete button (red circle with a minus icon) appears on a photo thumbnail only when all of the following are true:

| Condition | Detail |
|---|---|
| `remove_pod_enabled = true` | Consul flag; default `true`. Set to `false` to globally disable photo deletion. |
| Stop is not yet complete | The stop status must not be `succeeded`, `failed`, or `discarded`. Once a stop is finalised, photos cannot be deleted. |
| Photo has a UID and a remote URL | Photos that have not yet finished uploading (no URL assigned) cannot be deleted. |

When validation rules are visible (i.e. the ML model has run), there is one additional condition: the delete button is only shown for photos that failed validation (`notPassed`). Photos that passed validation cannot be deleted from the gallery thumbnail — the driver must open the full preview screen instead.

Tapping the button shows a confirmation dialog: *"Are you sure you want to delete this photo? This action cannot be undone."*

### 3. Tutorial Delivery Routes and Photo Capture

The app provides a tutorial mode for onboarding new drivers. The driver goes through the full delivery flow including photo capture — using a dummy (practice) assignment created from scanning a QR/barcode. Photo capture in the tutorial works identically to a real delivery; the dummy shipment carries a `deliveryProofPhotoRequired` flag set by the server that determines whether the photo is mandatory.

### 4. Photo Validation (AI/ML)

After a photo is taken, the on-device ML model immediately validates it:

1. Object detection — identifies the package and its bounding box in the image.
2. OCR — reads text on the label/package.
3. Rule analysis — compares detected results against the required rules defined for the stop (sourced from the assignment).

Validation result statuses:

| Status | Meaning |
|---|---|
| `ACCEPTED` | All required rules satisfied |
| `PARTIALLY_ACCEPTED` | Some rules satisfied, some failed |
| `BLOCKED` | Required rules not satisfied — driver must retake |
| `MANUALLY_ACCEPTED` | Dispatch reviewed and unblocked manually |

Depending on the enforcement policy (see Configuration below), the driver may be warned, blocked, or allowed to proceed regardless.

> If analysis takes longer than the configured UI timeout (`pod_analysis_ui_timeout_by_second`, default 2 seconds), the loading indicator is hidden and the block is automatically bypassed for that session. Analysis continues in the background.

### 5. Handling Validation Failures

#### Soft Block (`pod_soft_block` enabled)
- Driver sees a warning that the photo may be missing key details.
- Driver can choose to retake or proceed anyway.
- A retake count is tracked (`retakeCount`).

#### Hard Block (`pod_hard_block` enabled)
- Driver is blocked from completing the stop until the photo passes validation.
- Driver can tap "Contact Dispatch" to trigger an event (`pod_blocked_dispatch_contact`) that sends the stop's photos to the backend for review by the Gemini model.
- If dispatch approves, the block is lifted (`markUnblockedByReview()`), and the stop can proceed.

#### Conditional Soft Block (`warn` policy + `pod_warn_use_conditional_soft_block = true`)

This is a hybrid mode that combines elements of soft and hard block based on how many rules the photo satisfies. It is activated when the enforcement policy is `warn` and the `pod_warn_use_conditional_soft_block` flag is enabled on the assignment.

The outcome depends on the ML analysis result:

| Analysis result | Meaning | Driver experience |
|---|---|---|
| Accepted | All required rules satisfied | No interruption — driver proceeds normally |
| Partially accepted | At least one rule satisfied, but not all | Warning dialog appears, but it is immediately dismissible — driver can choose to proceed |
| Blocked | No required rules satisfied | Hard block — driver cannot complete the stop until a passing photo is taken or dispatch unblocks |

Key distinction from standard soft block: a driver who takes a completely invalid photo (e.g. a blank wall with no package visible) is blocked and cannot bypass the check. A driver whose photo partially meets requirements (e.g. package visible but label unreadable) is warned but allowed to proceed at their discretion.

### 6. Photo Upload

Once taken, the photo is saved locally and uploaded asynchronously:

Primary strategy — Cloud First (default):
1. File is registered with the sync controller.
2. Synced to Firebase/cloud storage asynchronously.
3. Upload event (including URLs and validation results) is reported to the API.
4. If cloud upload fails → falls back to direct API upload.

Fallback strategy — API First (enabled via `upload_through_api = true`):
1. State is updated immediately with the local file path (photo visible right away).
2. File is uploaded directly to the backend API.
3. If API upload fails → falls back to cloud upload.

Both the original and masked/blurred versions of the image can be uploaded when `pod_mask_enabled` is active and `send_original_image_enabled = true`.

If both strategies fail, the driver is prompted to contact dispatch, and the error is pre-formatted in the chat.

---

## Offline Mode

Both photo capture and POD analysis are designed to work fully offline. The driver is never blocked from taking a photo or receiving validation feedback due to a lack of network connectivity.

### Photo Capture Offline

When the driver takes a photo with no network connection:

1. The photo is saved to local device storage immediately.
2. It is registered in a persistent local queue (Hive database) with all its metadata (shipment ID, stop ID, upload parameters).
3. A pre-constructed cloud storage URL is generated at registration time, so the photo can be attached to the stop in the app's state right away — without waiting for the actual upload to complete.
4. A background sync is triggered. If the device is offline, the sync attempt exits silently without error.
5. When connectivity is restored, the sync controller automatically retries and uploads all queued photos.
6. Photos remain in the queue until the server confirms a successful upload. A failed upload is never silently dropped.

The driver sees the photo thumbnail immediately after capture regardless of connectivity. There is no upload progress indicator — the upload happens silently in the background.

### POD Analysis (ML Validation) Offline

POD photo validation runs entirely on-device using a bundled ML model. It has no network dependency and produces the same validation results whether the device is online or offline.

- The ML models (object detection, OCR) are downloaded and cached locally at app startup.
- If remote model updates are unavailable (offline), the app falls back to the locally bundled model versions.
- Analysis begins immediately after the photo is taken, using only the local image file and the locally stored models.
- Validation results (rules passed/failed, enforcement action) are applied normally — soft block, hard block, or log-only behaviour all function the same offline as they do online.

The event that links the validation result to the shipment server-side is cached locally and sent once connectivity returns.

### Summary

| Capability | Offline behaviour |
|---|---|
| Take a photo | Fully available — saved locally and queued for upload |
| View taken photos in gallery | Fully available |
| ML validation (object detection, OCR) | Fully available — runs on-device with bundled models |
| Soft block / hard block enforcement | Fully available — based on local analysis result |
| Photo upload to cloud | Deferred — queued locally, uploaded automatically when connectivity returns |
| Linking photo to shipment (API event) | Deferred — event cached locally and retried when online |
| Dispatch contact / manual unblock | Not available offline — requires network to reach dispatch chat |

---

## Internationalisation (i18n)

The feature uses a two-layer approach: static client-side translations for core UI strings, and on-device machine translation for dynamic server-driven strings such as POD rule names, instructions, and warning messages.

### Client-Side i18n

Core UI strings are managed using the `slang` package. Translation files live in `assets/i18n/` and are compiled into generated Dart code at `lib/i18n/translations.g.dart`.

Supported locales: `en` (English), `es` (Spanish).

The app locale is resolved at startup by reading the device system locale. If the system locale matches a supported locale it is used directly; otherwise it falls back to English.

Strings used in the photo capture and POD validation UI:

| Key | Default English text |
|---|---|
| `dropoff.takePhotoOfTheDelivery` | "Take photo of the delivery" |
| `dropoff.requirements` | "Requirements" |
| `podValidation.guidedInstruction` | "Take a photo of the package inside the frame first. Make sure the corners of the package are visible." |
| `podValidation.allRulesSatisfied` | "All requirements satisfied" |
| `podValidation.someRulesNotSatisfied` | "Some requirements not satisfied" |
| `properPhotosRequired` | "Proper Photos Required" |
| `yourPhotoMustContainTheBelowToConfirmDelivery` | "Your photo(s) must contain the below to confirm delivery. Please retake the photo(s) to continue." |
| `yourPhotoMayBeMissingKeyDetails` | "Your photo may be missing key details like the package, door, or unit number. Add extra photos if needed." |
| `checkYourPhoto` | "Check Your Photo" |
| `retakePhoto` | "Retake Photo" |
| `contactDispatch` | "Contact Dispatch" |
| `podAcceptanceGuidelines` | "POD Acceptance Guidelines" |
| `podAcceptanceGuidelinesContent` | Full guidelines text (clarity, framing, no people, etc.) |
| `button.takePhoto` | "Take photo" |
| `button.takeAnotherPhoto` | "Take another photo" |
| `button.gotItExclamation` | "Got It!" |
| `cameraAccessRequiredForDelivery` | "Camera access is required for delivery…" |

### Server-Driven Text and Machine Translation

POD validation rule names, instructions, and the `pod_validation_warning_message` config value are all sourced from the server and are not bundled in the app. These strings are displayed using the `MachineLocalizableTextBuilder` widget, which handles on-device translation automatically.

How it works:

1. The widget receives the raw English string from the server (`untranslatedText`) and an optional client-side fallback (`fallbackI18nText`).
2. If the device locale is English, the server string is shown as-is.
3. If the device locale is non-English, the widget uses Google MLKit on-device machine translation to translate the string to the system locale.
4. While translation is loading, a shimmer placeholder is shown.
5. Once translated, an optional toggle button lets the driver switch between the translated and original English text.
6. Google Translate attribution is shown alongside translated content when configured.

Strings that go through `MachineLocalizableTextBuilder`:

| String | Source |
|---|---|
| POD rule names (e.g. "Package visible") | Server — `pod_rules[].name` on the assignment |
| POD rule instructions (e.g. "Make sure the corners are visible") | Server — `pod_rules[].instruction` on the assignment |
| Soft-block warning message | Consul — `pod_validation_warning_message` (falls back to client i18n if not set) |

### Machine Translation Configuration

| Config key | Default | Description |
|---|---|---|
| `machine_translations_enabled` | `false` | Enables on-device machine translation for server-driven strings |
| `machine_translations_download_max_attempts` | `4` | Max retry attempts when downloading the MLKit translation model |
| `machine_translations_download_delay_factor_ms` | `4000` | Delay in ms between download retry attempts |
| `machine_translations_download_timeout_seconds` | `90` | Timeout for the model download before giving up |

The ML translation model is downloaded once and cached on-device. If the download fails (e.g. offline at first launch), server strings are shown in their original English until the model becomes available.

---

## Configuration Reference

All keys are Consul remote-config values unless noted as feature flags (controlled via phased release).

### Feature Flags

Feature flags are not simple booleans. They are rolled out by percentage using the Consul key:

```
public@phase_release_percentage_by_feature
```

This key holds a JSON object mapping each flag name to a rollout percentage (0–100):

```json
{
  "pod_soft_block": 50,
  "pod_hard_block": 0,
  "enable_blur_detection": 30
}
```

How it works:

1. Each device is assigned a stable bucket (0–99), computed as `SHA-256(deviceId) % 100`.
2. A feature is enabled for a device when `bucket < percentage`.
3. Setting a percentage to `100` enables the feature for all devices; `0` disables it for all.
4. The bucket is deterministic per device — the same device always lands in the same bucket, so the feature state does not change between app launches.

| Flag Key | Description |
|---|---|
| `enable_blur_detection` | Enables real-time blur detection during capture. Warns or silently flags blurry images. |
| `pod_soft_block` | Shows a warning when validation fails but allows the driver to proceed. |
| `pod_hard_block` | Blocks the delivery until the photo passes validation or dispatch approves. |
| `pod_log_only` | Runs validation and logs results without showing anything to the driver. Used for data collection. |
| `upload_pod_to_firebase_storage` | Routes photo uploads through Firebase Storage instead of direct API. |
| `upload_to_firebase_storage` | General Firebase upload toggle (applies beyond POD). |

> `pod_soft_block`, `pod_hard_block`, and `pod_log_only` are mutually exclusive by intent. Only one should be active at a time (i.e. only one should have a non-zero percentage).

### Camera & Image Quality

| Config Key | Default | Description |
|---|---|---|
| `camera_capture_type` | `internal` | Camera to use: `internal` (device camera) or `external`. |
| `pod_photo_quality` | `70` | JPEG quality for POD photos (0–100). |
| `pod_photo_compress_width` | `1024` | Max width in pixels after compression. |
| `pod_photo_compress_height` | `768` | Max height in pixels after compression. |
| `compressed_format` | `null` | Image format override (e.g. `jpeg`, `png`). Defaults to JPEG. |
| `compressed_quality` | `60` | Secondary compression quality applied after initial resize. |
| `minimum_quality` | `veryLow` | Blur detection sensitivity. Values: `veryLow`, `low`, `medium`, `high`, `veryHigh`. |
| `silent_blur_detection` | `true` | When `true`, blur warnings are not shown to the driver (logged only). |

### Masking (Privacy Blurring)

| Config Key | Default | Description |
|---|---|---|
| `pod_mask_enabled` | `false` | Enables a privacy mask/blur overlay on parts of the image (e.g. faces, licence plates). |
| `pod_mask_color` | `#B4AAA0` | Hex colour used for the mask overlay. |
| `mask_threshold` | `0.2` | Confidence threshold for the mask detection model (0.0–1.0). |
| `send_original_image_enabled` | `true` | When `true`, both the original and masked images are uploaded. |

### Validation & AI Rules

| Config Key | Default | Description |
|---|---|---|
| `pod_analysis_ui_timeout_by_second` | `2` | Seconds before the UI stops waiting for ML analysis and auto-bypasses the block. |
| `pod_validation_warning_message` | _(i18n default)_ | Custom warning message shown on soft-block. Overrides the default localised string. |
| `pod_warn_max_guided_attempts` | `1` | Max number of guided retake attempts before the warning UI changes. |
| `max_retakes_required` | `null` (unlimited) | Max number of retakes the driver is allowed before bypass is offered. |
| `object_detection_model_version` | `1.0.0` | ML model version for package bounding-box detection. |
| `ocr_detection_model_version` | `1.0.0` | ML model version for label text detection. |
| `ocr_recognition_model_version` | `1.0.0` | ML model version for label text recognition. |
| `model_labels` | `{"0":"cardboard box","1":"poly bag"}` | JSON map of ML model class IDs to label names. |

### Enforcement Policy (per assignment/stop)

Set on the assignment or stop level (not a global Consul key):

| Policy Value | `pod_warn_use_conditional_soft_block` | Behaviour |
|---|---|---|
| `none` | n/a | Validation is skipped entirely. |
| `invisible` | n/a | Validation runs but results are never shown to the driver. |
| `warn` | `false` (default) | Warning shown; driver can always proceed after exhausting guided retake attempts. |
| `warn` | `true` | Conditional soft block: partially accepted → warning + proceed; fully blocked → hard block. |
| `strict` | n/a | Driver blocked until all rules pass or dispatch approves. |

### Upload & Storage

| Config Key | Default | Description |
|---|---|---|
| `upload_through_api` | `false` | Forces direct API upload instead of cloud-first strategy. |
| `firebase_upload_retry_timeout` | `30` (seconds) | Max time Firebase SDK retries a failed upload before falling back to API. |
| `syncFileInterval` | `30` (seconds) | How often the sync controller checks for pending file uploads. |
| `asset_expiration` | `48` (hours) | How long signed photo URLs remain valid. |
| `remove_pod_enabled` | `true` | Whether drivers are allowed to delete a photo after it has been taken. |

### QA / Testing Help

| Config Key | Default | Description |
|---|---|---|
| `qa_pod_analyze_delay_by_millisecond` | `0` | Artificial delay before ML analysis runs. Used in QA to force the UI timeout path. |
| `qa_scan_picker_enabled` | `false` | Replaces the camera with a file picker, allowing QA to upload an existing photo instead of capturing one. Useful for testing ML analysis with controlled images. |

---

## POD Validation Rules

Rules are defined server-side per assignment and delivered with the assignment payload. Each rule has:

- `name` — identifier.
- `condition` — the detection condition the image must satisfy (e.g. `package_visible`, `label_readable`).
- `priority` — higher priority rules are shown first as guided instructions.
- `instruction` — the human-readable prompt shown to the driver (e.g. "Take a photo of the package inside the frame first").

The app always shows the highest-priority unsatisfied rule as the active instruction.

---

## Related Code Locations

| Area | File |
|---|---|
| Photo capture screen | [lib/widgets/take_photo_screen.dart](../../lib/widgets/take_photo_screen.dart) |
| Capture with blur detection | [lib/widgets/capture_photo_screen.dart](../../lib/widgets/capture_photo_screen.dart) |
| Photo gallery (stop-level) | [lib/screens/delivery/gallery/photo_gallery.dart](../../lib/screens/delivery/gallery/photo_gallery.dart) |
| Photo preview + rule feedback | [lib/screens/delivery/gallery/pod_photo_preview_screen.dart](../../lib/screens/delivery/gallery/pod_photo_preview_screen.dart) |
| Photo POD button (delivery card) | [lib/screens/delivery/dropoff/widget/pod_button/photo_pod_button.dart](../../lib/screens/delivery/dropoff/widget/pod_button/photo_pod_button.dart) |
| Upload orchestration | [lib/blocs/delivery/delivery_bloc_helpers/handle_take_stop_photo.dart](../../lib/blocs/delivery/delivery_bloc_helpers/handle_take_stop_photo.dart) |
| ML validation notifier | [lib/blocs/pod_validator/pod_validator_notifier.dart](../../lib/blocs/pod_validator/pod_validator_notifier.dart) |
| Validation state | [lib/blocs/pod_validator/pod_validator_state.dart](../../lib/blocs/pod_validator/pod_validator_state.dart) |
| POD rule model | [lib/blocs/pod_validator/models/pod_rule_model.dart](../../lib/blocs/pod_validator/models/pod_rule_model.dart) |
| POD pass/fail logic | [lib/utils/extensions/decor_stop_extensions.dart](../../lib/utils/extensions/decor_stop_extensions.dart) |
| Camera config | [lib/config/src/camera_config.dart](../../lib/config/src/camera_config.dart) |
| Feature flags enum | [lib/controllers/driver_app_info.dart](../../lib/controllers/driver_app_info.dart) |
| POD analyzer guard | [lib/screens/delivery/dropoff/widget/pod_analyzer_guard.dart](../../lib/screens/delivery/dropoff/widget/pod_analyzer_guard.dart) |
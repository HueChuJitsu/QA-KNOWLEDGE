# How To: Create an IC Driver via the Driver App

> **App:** Jitsu Drive (Driver App) · **Driver type:** IC (Independent Contractor) · **Env:** Staging (notes for Beta/Prod where they differ)
> Step-by-step guide to sign up and activate an IC driver through the Driver App.
>
> **Sources (Confluence):**
> - [\[Driver app\] Registration Flow](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/717095051/Driver+app+Registration+Flow)
> - [How to sign up driver in Driver App](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/982089802/How+to+sign+up+driver+in+Driver+App)
>
> Related: [Training Document §11.5 — Sign Up & Approve a Driver](./jitsu-qa-training-document.md) · domain context in [jitsu-context.en.md](./jitsu-context.en.md)

---

## 1. Overview

IC (Independent Contractor) drivers are individual drivers who **self-register through the Driver App** (Jitsu Drive) and deliver within a zone — never called couriers. This differs from DSP drivers, which are created via the DSP Portal + Dashboard (see Training Document §11.8).

There are **two ways** to register an IC driver:
- **A. Sign up in the Driver App** (the standard self-registration flow) — covered in §3.
- **B. Register via the Admin App** (`admin.staging.gojitsu.com/drivers/create`) — covered in §6.

After registration, the driver goes through a Background Check (BC). For **QA test drivers** (Staging, Beta, Production), BC approval is shortcut via API (§4). See the **Production & Beta permissions & data-hygiene** rules below before creating anything on Prod/Beta.

## ⚠️ Production & Beta — Permissions & Data Hygiene

Read this **before** creating any account on Production or Beta.

### Account-creation permissions

- **Staging:** the QA Lead has **super-admin** and can create an admin account for you.
- **Production:** **contact the QA Lead first** before creating any account. You do **not** need to escalate via Slack `#devops`.

### Creating a driver on Prod / Beta

- A real background check **cannot be submitted** in a test context, so the driver is still approved via **Simulate FCRA approval (Postman)** — the same shortcut as Staging (see §4).

### Data hygiene — avoid junk data on Prod / Beta

- **Clients:** create **at most 4** — one per client type. Do not create more than needed (avoids junk data on Prod).
- **Warehouses:**
  - Do **NOT** create a warehouse with type **Jitsu main warehouse**.
  - Only create with type **client** or **dsp warehouse**.
  - If a warehouse already exists in that region, **reuse it** — don't create a new one.
  - If you must create one, **delete it after use**.
- **Assignments:** use only the **2 prefixes `TEST` and `QA`** for testing. Do not create other prefixes — if you need a different prefix, **ask the QA Lead first**.

### TEST region (Beta / Prod)

- The QA team has created a dedicated **TEST region** on **Beta and Production**. Prefer testing new features in the TEST region whenever real data is not strictly required.
- After creating a client, **message the QA Lead** to add the client id into the **`zipcodes_clients`** table so the client can use the TEST region.
- **Do NOT route with IC drivers in any region other than TEST.** If you must run routing in a real region, after finishing you **must clean up**: delete the **shipment(s)**, delete the **assignment(s)**, **deselect the problem**, and **delete the routing problem**.

## 2. Prerequisites

- Driver App installed (Staging build from Firebase / TestFlight).
- A phone number for verification:
  - **Staging:** any random number works (verification code is hardcoded).
  - **Beta/Prod:** any random number works — the SMS verification code is retrieved from the Mongo **`communication_logs`** collection (no real phone needed).
- An email address not already registered on the system.
- Postman + QA Master collection (for the FCRA-approval shortcut). **To use the company Postman (team workspace / collection), contact the QA Lead to have your account added.**
- DB access (MongoDB) for config/verification (see §8).

## 3. Sign Up in the Driver App

### 3.1 Phone verification

1. Tap **Sign Up to Driver**.
2. Input the phone number → tap **Create** → moves to the **Verify Mobile Number** screen.
   - Staging: input a random number.
   - Beta/Prod: input a random number.
3. Input the verification code → tap **Create** → moves to the **Registration Questions** screen.
   - **Staging:** hardcode `6868`.
   - **Beta/Prod:** read the code from the Mongo `communication_logs` collection.
   - Config (enable/disable questions): Consul key `beta/apps/driverappapi/mobile_app_config/public@disable_question`.

### 3.2 Registration questions & delivery quiz

4. **Answer all 4 registration questions** → moves to the **Delivery Quiz** screen.
   - If the driver is in an area where Jitsu is **not hiring**, they can input an **invite code** to bypass (ENG-5510).
5. **Delivery Quiz** — scoring depends on the questionnaire config in Mongo:
   `db.getCollection('questionnaire').find({name: "MOBILE-APP-DRIVER-CANDIDATE"})`
   - **≥ 8 points:** pass → moves to the **Create Account** screen.
   - **< 8 points:** fail.

### 3.3 Create account

6. **Create Account** — input:
   - **Username** — invalid if it already exists on the system.
   - **Password** — at least 6 characters, with ≥1 uppercase, 1 lowercase, 1 digit, 1 special character.
   - **Confirm Password** — must match the password.
   - **Email address** — invalid if it already exists on the system.
7. **Email verification:**
   - A valid code is sent to the registered email.
   - Tapping **Resend code** expires the previous code.
   - The code stays valid as long as the driver does not tap Resend.
   - Tap **Continue** → moves to 3 sections: Personal Information, Vehicle Information, Social Security Information.

### 3.4 Personal Information

8. Fill in:
   - **Nickname** — optional.
   - **First Name** — mandatory, ≤ 32 chars.
   - **Middle Name** — optional, ≤ 32 chars.
   - **Last Name** — mandatory, ≤ 32 chars.
   - **Date of Birth** — mandatory. Driver must be **at least 21**; DOB picker does not allow under-21.
   - **Gender** — mandatory, select option.
   - **Address Line 1** — mandatory; **Address Line 2** — optional.
   - **City** — mandatory.
   - **State** — mandatory, select option.
   - **Zipcode** — mandatory; format `xxxxx` or `xxxxx-yyyy` / `xxxxx yyyy`.
   - **Driver's License No** — mandatory.
   - **Driver's License State** — mandatory.
   - **Driver's License Issued Date** / **Expired Date** — mandatory (no validation currently — improvement pending).
   - **Driver's License Front Photo** / **Back Photo** — mandatory; capture or upload from device.

### 3.5 Vehicle Information

9. Year / Make / Model are validated against the `cars` table in Mongo:
   - **Year** — mandatory, dropdown. Selecting a year enables **Make**.
   - **Make** — mandatory, dropdown. Selecting Year + Make enables **Model**.
   - **Model** — mandatory, dropdown.
   - **Color** — mandatory, ≤ 32 chars.
   - **License Plate** — mandatory, ≤ 32 chars.
   - **License Plate State** — mandatory.
   - **Vehicle Insurance Issued Date** / **Expired Date** — mandatory.
   - **Vehicle Insurance Photo** — mandatory; capture or upload.
   - **Registration Record Issued Date** / **Expired Date** — mandatory.
   - **Registration Record Photo** — mandatory; capture or upload.

   > **Note:** If the driver doesn't finish all Vehicle Insurance / Registration Record fields, the section can still be saved and shows the correct completion percentage, but the **Continue** button to the next step is not displayed (requirement being updated).

### 3.6 Social Security Information

10. **SSN** (mandatory) must satisfy:
    1. 9 digits.
    2. Split into 3 parts by hyphen (`-`).
    3. First part: 3 digits, not `000`, `666`, or `900–999`.
    4. Second part: 2 digits, `01–99`.
    5. Third part: 4 digits, `0001–9999`.
    6. Cannot reuse an SSN that already exists on the system.

### 3.7 Background Check & contract

11. **Background Check screen** — informational text about the BC.
12. **Intents list** — tick the checkboxes to continue:
    - "I do not have a middle name" — optional.
    - "I certify that this is me… agree to Turn Technologies" — optional.
    - The remaining checkboxes are **mandatory**.
13. **Background Check Submit screen** — information cannot be edited; **signing is required**; tap **Submit** to start the BC.
14. The **AxleHire waiting screen** is shown while BC is in progress.
15. **Frozen / Unfrozen** — controlled by Mongo `driver_coverage_area` (per selected region):
    - `Hiring = false` → driver is **Frozen** for that region. Driver stays at the submit step and sees: *"We are currently not enrolling new drivers with your specifications. We will reach out when we are admitting new drivers in your area and/or vehicle class."*
    - `Hiring = true` → driver is **Unfrozen** and continues waiting until BC approves.
16. When BC is **bypassed/approved** → the **Contract** is displayed.
17. After the driver taps **I agree**, a "wait for review contract" screen shows. After ~30 seconds, a popup informs the driver that registration is complete. The driver must **log in** to use the app.

## 4. Background Check Approval (FCRA) — QA shortcut

> **💡 QA test drivers.**
> - **Staging:** you **can submit** the Background Check from the app, then approve it via the Simulate FCRA approval API below.
> - **Beta/Prod:** a real Background Check **cannot be submitted** for a test driver, so use the same Simulate FCRA approval shortcut to approve.
>
> Genuine production drivers still go through the real background-check process — this shortcut is for QA-created test accounts only.

After submitting the Background Check, approve the driver via API:

1. Get the driver's access token via the **`[Oauth] Login`** API in Postman.
2. Run **`7.[Reg] Simulate fcra approval`** using the token of the signed-up driver above.

(Optionally simulate Turn status with the `Outcome received` API; BC data is saved in the `driver_registration_record` collection in MongoDB.)

## 5. Verify the Driver Is Active

- After FCRA approval + agreeing to the contract, the driver can **log in** to the Driver App without a pending-approval block.
- The driver can reach the **booking screen** and book a route.

## 6. Alternative — Register via the Admin App

Instead of the in-app flow, an IC driver can be created from the Admin App:
`https://admin.staging.gojitsu.com/drivers/create`

## 7. Notes & Gotchas

- **Beta/Prod SMS code:** the verification code is not sent to a real phone — read it from the Mongo `communication_logs` collection.
- **Not hiring in area:** use an invite code to bypass the quiz gate (ENG-5510).
- **Frozen region:** if `driver_coverage_area.Hiring = false`, the driver is stuck at the BC submit step — set `Hiring = true` to unfreeze.
- **DOB:** under-21 is blocked by the picker.
- **License / Registration dates:** currently have no validation — easy to enter invalid dates.

## 8. Reference Data (MongoDB collections)

| Collection / query | Use |
|---|---|
| `db.getCollection("quiz").find({})` | Delivery quiz data |
| `db.getCollection('questionnaire').find({name: "MOBILE-APP-DRIVER-CANDIDATE"})` | Quiz scoring config (≥ 8 to pass) |
| `db.getCollection('event').find({action: "use-invite-code"})` | Invite-code usage events |
| `db.getCollection("invitation_codes").find({})` | Invite codes |
| `db.getCollection("invitation_code_logs").find({})` | Invite-code logs |
| `db.getCollection("driver_registration_target_sets").find({region:/ATL/})` | Registration target sets per region |
| `db.getCollection("driver_coverage_area").find({})` | Hiring / Frozen control per region |
| `cars` table | Year / Make / Model validation for Vehicle Information |
| `driver_registration_record` | BC / Turn status data |
| `communication_logs` | SMS verification code on **Beta/Prod** (look up the code sent during phone verification) |

## 9. Related References

- [\[Driver app\] Registration Flow (Confluence)](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/717095051/Driver+app+Registration+Flow) — authoritative source for this guide
- [How to sign up driver in Driver App (Confluence)](https://gojitsu.atlassian.net/wiki/spaces/ENG/pages/982089802/How+to+sign+up+driver+in+Driver+App)
- [Training Document §11.5 — Sign Up & Approve a Driver](./jitsu-qa-training-document.md)
- [Training Document §11.8 — Create DSP Account & DSP Driver](./jitsu-qa-training-document.md) (the DSP driver path, by contrast)

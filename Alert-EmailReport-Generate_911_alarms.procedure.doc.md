Let’s dive into understanding the `Generate_911_alarms` procedure in Progress 4GL (ABL), which is designed to manage emergency call alerts for a 911 system, likely within a hospitality or property management environment. Below, I’ll provide a detailed, self-contained explanation of what this procedure does, how it processes its inputs, and how it generates alerts, all based on the provided code.

---

### Overview of the Procedure
The `Generate_911_alarms` procedure is responsible for generating and managing alerts when a 911 emergency call is detected. It takes specific input parameters related to the call, gathers additional context (like property details, guest information, and room numbers), checks for duplicate alerts, and then triggers notifications through various channels—such as email, phone, SMS, or pop-ups—based on the property’s configuration. It also logs the event for tracking and ensures compatibility with legacy systems.

---

### Input Parameters
The procedure accepts the following inputs:

- **`p_company_num` (integer)**: Identifies the company or property where the call originated.
- **`p_origination_ext#` (character)**: The extension number from which the 911 call was made.
- **`p_dCallDateTime` (datetime)**: The date and time when the call occurred.
- **`pDigitsDialed` (character)**: The digits dialed (expected to be "911").
- **`p_iPbxRawSeq` (decimal)**: A sequence number linking to raw PBX (Private Branch Exchange) data for the call.
- **`p_ExtnName` (character)**: The name associated with the extension, potentially including PBX name and location separated by a `|` delimiter.

These parameters provide the core details needed to process the emergency call and generate appropriate alerts.

---

### Key Variables
The procedure defines several variables to store intermediate data, including:
- Buffers (e.g., `b-pbx_raw_data` for PBX data, `bmsg_queue` for message queuing).
- Strings for company name (`vccompanyname`), guest name (`vcguestname`), room number (`lcRoomNumber`), and email content (`vcEmailText`).
- Logical flags for alert types (`lSendEmailAlert`, `lSendPhoneAlert`, `lSendPopUpAlert`, `lSMSCreate` for SMS).
- A unique alert ID (`liAlertID`) and time zone offset (`l_iTimeDiff`).

These variables help manage the data collection and alert generation process.

---

### Step-by-Step Execution

#### 1. **Logging the Call**
The procedure begins by logging the call details to a file (e.g., `DLL-911AlarmTrigger-MM-DD-YYYY.log`), including the property ID, extension, call date/time, and sequence number. This ensures a record of every invocation.

#### 2. **Legacy Property Check**
- It checks if the property is "legacy" by querying the `Posting_Parameters` table for `p_company_num` where `Activate = yes` and `CreatePBXManagement = yes`.
- If no record is found or `CreatePBXManagement` is not `yes`, `isLegacyProp` is set to `true`. This flag influences duplicate alert handling later.

#### 3. **Duplicate Alert Check**
- It searches for an existing `Alert_Popups` record with:
  - `Alert_Type = 9` (indicating a 911 alert).
  - Matching `EventDateTime`, `Extension`, and `propertyid`.
- If no exact match is found and `p_iPbxRawSeq > 0`, it broadens the search to a 30-second window around `p_dCallDateTime`.
- If an alert exists and the property is not legacy:
  - If the existing alert’s `JazzAlertRawReference = 0` and a sequence number is provided, it logs a "SKIPPED" message and exits to avoid duplicate processing.

#### 4. **PBX Raw Data Retrieval**
- If `p_iPbxRawSeq` is valid (not null or zero), it retrieves the corresponding `pbx_raw_data` record to get the raw call record (`l_pbx_raw_record`) and sets `lcPbxRawSeq` to the sequence number as a string.

#### 5. **Alert Type and Initial Setup**
- The alert type (`liAlertType`) is set to `9` (for 911 calls).
- Guest name, room number, and guest ID are initialized as empty.

#### 6. **Time Zone Adjustment**
- It queries `ext_company_num` for a `TIME-DIFF` field to calculate the property’s time zone offset (`l_iTimeDiff`) in seconds.

#### 7. **Company Name Retrieval**
- The property’s name (`vccompanyname`) is fetched from the `company_number` table using `p_company_num`.

#### 8. **Primary Extension Determination**
- If `p_origination_ext#` is not found in `Extension_Master`, it checks `Secondary_Extension_Master` to find the primary extension (`p_Primary_ext#`) linked to the secondary extension.

#### 9. **Room and Guest Information**
- It looks for an `appartment` record tied to `p_Primary_ext#` to get the room number (`lcRoomNumber`).
- If found:
  - It retrieves the latest `guest_stay` record for that room with no `move_out_date` (indicating a current guest).
  - It then fetches the guest’s name (`vcguestname`) and ID (`lcGuestId`) from the `guest` table.
  - If no guest is found, `vcguestname` defaults to "Guest Room".
- If no apartment is found:
  - It checks `Extension_Master` for the extension’s name. If none exists, it sets `lNoExtension = true` and leaves fields blank.

#### 10. **Alert Configuration**
- Alert flags (`lSendEmailAlert`, `lSendPhoneAlert`, `lSendPopUpAlert`) are initially set to `no`.
- It checks `Alert_PropertyLevel_Config` for property-wide settings:
  - `SupportPhoneAlerting` or `SupportExtensionAlerting` sets `lSendPhoneAlert`.
  - `SupportSMSAlerting` sets `lSMSCreate`.
- It then overrides these with specific settings from `Alert_Config` for `Alert_Type = 9`.
- Finally, it forces all alert flags to `true` (this appears redundant and may be a default override).

#### 11. **Alert ID Generation**
- If any alert flag is `true`, a unique `liAlertID` is generated using the `Alert_seq` sequence.

#### 12. **Email Content Construction**
- Subject: `"A 911 Emergency Call has been placed at " + vccompanyname`.
- Condition: `"A 911 Emergency Call has been placed."`.
- If `p_ExtnName` is provided, it extracts `cPBXNAME` and `cPBXLOCATION` (split by `|`).
- Email text (`vcEmailText`) includes:
  - Property name, extension, room number, guest name, call time, digits dialed, sequence number, and raw record.

#### 13. **Backward Compatibility Message**
- `l_cAlertMsg` is a pipe-delimited string with all alert details (e.g., property, guest ID, room, etc.) for legacy systems.

#### 14. **Alert Generation**
- **Email**: If `lSendEmailAlert = true`, it calls `ConstructEmail` to prepare the email and `GenerateEmailMsgQ` to queue it.
- **Phone**: If `lSendPhoneAlert = true`, it creates an `Alert_Schedule` record with call details.
- **SMS**: If `lSMSCreate = true`, it creates another `Alert_Schedule` record with `alert_extension = "sms"`.
- **Pop-up**: 
  - If `lSendPopUpAlert = true`, it creates an `Alert_Popups` record with the alert details.
  - If `false`, it still creates an `Alert_Popups` record but marks it as acknowledged by "SYSTEM_COLLECTOR".

#### 15. **Event Subscription Check**
- If the property subscribes to `GenerateEmergencyAlertEvent` (checked via `parameter_master`), it creates a `msg_queue` record with event details (room, extension, guest, alert datetime).

#### 16. **Cleanup**
- Releases `Alert_Schedule` and `Alert_Popups` buffers.

---

### Summary
The `Generate_911_alarms` procedure processes 911 emergency calls by:
1. Logging the call and checking for duplicates.
2. Gathering property, extension, room, and guest details.
3. Configuring and triggering alerts (email, phone, SMS, pop-ups) based on property settings.
4. Logging events for subscribed properties.

It ensures robust alert delivery while avoiding redundant notifications and maintaining compatibility with legacy systems. The procedure is flexible, handling various configurations and data scenarios effectively within a property management context.

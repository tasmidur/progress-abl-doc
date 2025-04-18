Below is an explanation of the provided Progress 4GL program, which handles 911 emergency call alerts for a PBX (Private Branch Exchange) system. The program processes data from a temp-table, determines the property (company) associated with the call, converts the call time to the property's local time, checks for duplicates, and generates an alarm if necessary. It also logs various steps for debugging and auditing purposes. I’ll break down the code into its key sections (or "blocks") and explain the purpose of each.

---

## Overview of the Program
The program is designed to:
- Take input data from a temp-table (`tt911AlarmTrigger`) containing details of a 911 call (e.g., call time, extension, dialed digits, etc.).
- Identify the company/property associated with the call.
- Convert the call’s UTC time to the property’s local time.
- Check if the call is exempt or a duplicate.
- Generate a 911 alarm if all conditions are met.
- Log key steps for debugging and auditing.

It uses an input parameter (the temp-table) and outputs a logical result (`opResult`) indicating success or failure.

---

## Code Breakdown and Purpose of Each Section

### 1. Include Files and Variable Definitions
```progress
USING TimeZoneConvert.* FROM ASSEMBLY.
/*------------------------------------------------------------------------------
    Purpose:     
    Parameters:  <none>
    Notes:       
------------------------------------------------------------------------------*/
{tt911AlarmTrigger.i}
define var l_cdateformat as char no-undo.
assign l_cdateformat = session:date-format.
assign session:date-format = "mdy".

DEFINE INPUT PARAMETER TABLE FOR tt911AlarmTrigger.
DEFINE OUTPUT PARAMETER opResult    AS logical  NO-UNDO.   
```
- **Purpose**: This section sets up the environment for the program.
  - `USING TimeZoneConvert.*`: Imports a time zone conversion library.
  - `{tt911AlarmTrigger.i}`: Includes a file defining the temp-table structure for 911 call data.
  - `l_cdateformat`: Stores the current session date format so it can be restored later.
  - `session:date-format = "mdy"`: Sets the date format to "month-day-year" for consistent date handling.
  - `DEFINE INPUT PARAMETER TABLE FOR tt911AlarmTrigger`: Declares the temp-table as an input parameter.
  - `DEFINE OUTPUT PARAMETER opResult`: Declares a logical output parameter to indicate the program’s success (`yes`) or failure (`no`).

---

### 2. Variable Declarations
```progress
DEFINE var  ipBWEnterprise   AS CHARACTER NO-UNDO.
DEFINE var  ipBWGroup        AS CHARACTER NO-UNDO.
def var utcdatetime as datetime.
define var p_origination_ext#   as char.
def var l_next_seq                  as int.
def var l_output_msg                as char no-undo.
def var l_operation_successfull     as log no-undo.
def var l_send_alarm_immediately    as log no-undo initial yes.
DEF VAR l_911_code                  AS CHAR NO-UNDO.
DEF VAR l_PropOrigin                AS CHAR NO-UNDO.
DEFINE VARIABLE vccompanyname  AS CHARACTER NO-UNDO.
DEFINE VARIABLE vcguestname AS CHARACTER NO-UNDO.        
define variable lcRoomNumber as char no-undo. 
DEF VAR opiCompany_num    AS INT NO-UNDO.    
def var ldPropertyTime as datetime no-undo.
def var l_cPropertyZone as char no-undo.
def var cresult as char no-undo.
def var ccommand as char no-undo.
```
- **Purpose**: Declares variables used throughout the program.
  - Examples include:
    - `ipBWEnterprise`, `ipBWGroup`: Store enterprise and group IDs from the PBX system.
    - `p_origination_ext#`: Stores the originating extension number.
    - `opiCompany_num`: Stores the company/property number.
    - `ldPropertyTime`: Stores the call time adjusted to the property’s local time.
    - `l_cPropertyZone`: Stores the property’s time zone offset.
  - These variables hold call details, time conversion data, and operational flags.

---

### 3. Initial Assignments and Logging
```progress
assign opResult  = no  
       opiCompany_num = 0.
find first tt911AlarmTrigger no-lock no-error.
if not avail tt911AlarmTrigger then return.
ASSIGN ipBWEnterprise   = tt911AlarmTrigger.BWenterpriseId
       ipBWGroup        = tt911AlarmTrigger.BWgroupId . 
       p_origination_ext# = tt911AlarmTrigger.BWuserExtension .
output to value("..\logs\DLL-911AlarmTrigger-" + string(month(today)) + "-" + string(day(today)) + "-" + string(year(today)) 
+ ".log" ) append.
put unformatted "Entry: " string(today) " " string(time,"hh:mm:ss") ","
        opiCompany_num ","
        BWcallStartTime ","
        BWdialedDigits ","
        BWuserId  ","
        BWuserName ","
        BWuserExtension ","
        BWuserPhone ","
        BWenterpriseId ","
        BWgroupId ","
        BWgroupName ","
        BWgroupAddress ","
        BWclidNumber ","
        BWclidName "," 
        BWsrcIpAdd "," 
        session:date-format skip.
output close.
```
- **Purpose**: Initializes key variables and logs the program’s entry.
  - Sets `opResult = no` (default failure) and `opiCompany_num = 0`.
  - Checks if the temp-table has data; if not, exits.
  - Assigns values from the temp-table (e.g., enterprise ID, group ID, extension).
  - Logs call details (e.g., timestamp, user ID, dialed digits) to a file named with the current date (e.g., `DLL-911AlarmTrigger-10-15-2023.log`) for auditing.

---

### 4. Determine Company Number
```progress
if  BWgroupId  <> "ooma-emergency" and BWgroupId  <> "peerless-emergency" then
do:
{BWgetJazzCompanyNum.i}
end.
else
do:
   if BWgroupId = "peerless-emergency" then
   do:
        opiCompany_num = 0.
        for each if-Interface-ComLib-Parameter WHERE  if-Interface-ComLib-Parameter.InterfaceGroup = "PBX"
            AND if-Interface-ComLib-Parameter.company_num > 0    
            AND if-Interface-ComLib-Parameter.AttrName = "BWEnterpriseCode"   no-lock :
            if trim(entry(1,if-Interface-ComLib-Parameter.AttrValue,";")) = trim(BWenterpriseId ) then
            do:
                assign opiCompany_num = if-Interface-ComLib-Parameter.company_num .
                leave.
            end.      
        end.
        if opiCompany_num = 0 then
        do:
            opResult = false .
            message "peerless emergency error: Property for " + trim(BWenterpriseId ) + " not found in Jazz."  . 
            return.
        end.   
   end.
   find first BW-ExtensionToUserName_Map where  BW-ExtensionToUserName_Map.BwUserName =  BWuserId no-lock no-error.
   if avail BW-ExtensionToUserName_Map then 
   do:
        opiCompany_num = BW-ExtensionToUserName_Map.Company_Num.
        p_origination_ext# = BW-ExtensionToUserName_Map.Extension_Num.
   end.
   else 
   do:
        find first BW-ExtensionToLinePort_Map where BW-ExtensionToLinePort_Map.BwLinePort = BWuserId no-lock no-error.
        if avail BW-ExtensionToLinePort_Map then
        do:
            opiCompany_num = BW-ExtensionToLinePort_Map.Company_Num.
            p_origination_ext# = BW-ExtensionToLinePort_Map.Extension_Num.
        end.
   end.
   if opiCompany_num = 0 and (trim(BWenterpriseId) ne "" and trim(BWenterpriseId) ne ? ) then   
   do:
        FIND FIRST if-Interface-ComLib-Parameter WHERE  if-Interface-ComLib-Parameter.InterfaceGroup = "PBX"
            AND if-Interface-ComLib-Parameter.AttrName = "BWEnterpriseCode"  AND 
            trim(if-Interface-ComLib-Parameter.AttrValue) = TRIM(tt911AlarmTrigger.BWenterpriseId) + ";" NO-LOCK NO-ERROR.
        IF AVAILABLE if-Interface-ComLib-Parameter THEN 
        DO:
            opiCompany_num =  if-Interface-ComLib-Parameter.company_num. 
        END.
   end.
end.
if opiCompany_num = 0 then 
do:
    assign session:date-format = l_cdateformat.
    output to value("..\logs\DLL-911AlarmTrigger-" + string(month(today)) + "-" + string(day(today)) + "-" + string(year(today)) 
    + ".log" ) append.    
    put unformatted "Property not found: " string(today) " " string(time,"hh:mm:ss") ","
            opiCompany_num ","
            BWcallStartTime ","
            BWdialedDigits ","
            BWuserId  ","
            tt911AlarmTrigger.BWuserName ","
            BWuserExtension ","
            BWuserPhone ","
            BWenterpriseId ","
            BWgroupId ","
            BWgroupName ","
            BWgroupAddress ","
            BWclidNumber ","
            BWclidName "," 
            BWsrcIpAdd "," 
            session:date-format skip.
    output close.
    return.
end.
```
- **Purpose**: Identifies the company/property number (`opiCompany_num`) associated with the call.
  - For non-emergency groups (`BWgroupId <> "ooma-emergency" and <> "peerless-emergency"`), it uses an include file (`BWgetJazzCompanyNum.i`).
  - For "peerless-emergency":
    - Looks up the company number in `if-Interface-ComLib-Parameter` based on the enterprise ID.
    - Exits with an error if not found.
  - For other cases:
    - Checks mappings (`BW-ExtensionToUserName_Map` or `BW-ExtensionToLinePort_Map`) to find the company number and extension.
    - Falls back to `if-Interface-ComLib-Parameter` if mappings fail.
  - If no company number is found, logs the failure and exits.

---

### 5. Check for Exempt Numbers
```progress
find first parameter_master where parameter_master.company_num = opiCompany_num  and 
           parameter_master.Parameter_code = "EmergencyAlertExemptNumbers" no-lock no-error.
if not available parameter_master then 
find first parameter_master where parameter_master.company_num = 0  and 
           parameter_master.Parameter_code = "EmergencyAlertExemptNumbers" no-lock no-error.           
if avail parameter_master then 
do:
    if lookup(trim(BWdialedDigits),trim(parameter_master.char_Val)) > 0  THEN 
    DO:
        opResult = false .
        message "error: 911 emergency number: " + trim(BWdialedDigits) + " does not send out alert."  . 
        return.
    END.
end.
```
- **Purpose**: Checks if the dialed number (`BWdialedDigits`) is exempt from generating an alert.
  - Looks for an exemption list in `parameter_master` for the specific company or a global default (company_num = 0).
  - If the dialed number is in the list, sets `opResult = false` and exits.

---

### 6. Time Conversion to Property Local Time
```progress
ldPropertyTime = tt911AlarmTrigger.BWcallStartTime.
find first ServerTZMapping where ServerTZMapping.propertyid =  opiCompany_num no-lock no-error.
if available ServerTZMapping then 
do:
    Assign  l_cPropertyZone = ServerTZMapping.TZoffset  .        
    utcdatetime =  TimeZoneConvert.TimeZoneConversion:UTCToProperty(INPUT ldPropertyTime, INPUT int(l_cPropertyZone), INPUT "d:\sdd64\config\SddTimeZone.xml")  .
    cresult = string(utcdatetime).
    if cresult ne "" and  cresult ne  ?  then 
    do:       
       ldPropertyTime = datetime(cresult).
    end.
end. 
if not avail ServerTZMapping or cresult eq ""  or  cresult eq  ?  or ldPropertyTime eq ? then
do:
   find first ext_company_num where ext_company_num.company_num eq opiCompany_num and 
           ext_company_num.Field_name eq "TIME-DIFF" no-error.
   if  avail ext_company_num  then 
   do:
      if ldPropertyTime = ? then ldPropertyTime = tt911AlarmTrigger.BWcallStartTime.
      assign  ldPropertyTime = add-interval(ldPropertyTime,ext_company_num.int_val,"hours") .
   end.
end.
output to value("..\logs\DLL-911AlarmTrigger-" + string(month(today)) + "-" + string(day(today)) + "-" + string(year(today)) 
+ ".log" ) append.    
put unformatted "Converted time (UTC to Property): " string(today) " " string(time,"hh:mm:ss") ","
        opiCompany_num ","
        BWcallStartTime ","
        BWdialedDigits ","
        BWuserId  ","
        tt911AlarmTrigger.BWuserName ","
        BWuserExtension ","
        BWuserPhone ","
        BWenterpriseId ","
        BWgroupId ","
        BWgroupName ","
        BWgroupAddress ","
        BWclidNumber ","
        BWclidName "," 
        BWsrcIpAdd "," 
        string(ldPropertyTime) skip.
output close.
```
- **Purpose**: Converts the call start time (`BWcallStartTime`) from UTC to the property’s local time.
  - Uses `ServerTZMapping` and the `TimeZoneConvert` library if available.
  - Falls back to a time difference (`ext_company_num.int_val`) if the conversion fails.
  - Logs the converted time (`ldPropertyTime`) for auditing.

---

### 7. Determine Cloud PBX Type
```progress
def var CloudPBXType as char no-undo.
CloudPBXType = "".
find parameter_master where parameter_master.company_num = opiCompany_num  and 
 parameter_master.parameter_code = "CloudPBXType" no-lock no-error.
 if  Available parameter_master then do:
  Assign CloudPBXType =  trim(parameter_master.char_val) . 
 end.
```
- **Purpose**: Identifies the type of Cloud PBX system (e.g., "peerless", "ooma") used by the property.
  - Retrieves the value from `parameter_master` to influence later logic (e.g., duplicate checks).

---

### 8. Check for Duplicate Alerts
```progress
if CloudPBXType  = "peerless" or CloudPBXType = "ooma" then
do:
    if trim(BWsrcIpAdd) ne "" and trim(BWsrcIpAdd) ne ?  then 
    do:
        FIND FIRST Alert_Popups where Alert_Popups.Alert_Type = 9 and 
                      Alert_Popups.propertyid = opiCompany_num  and 
                      Alert_Popups.AckIPAddress = BWsrcIpAdd  no-lock no-error.
    end.
end.
if not avail Alert_Popups then 
do:
    if  BWenterpriseId <> "ooma-emergency" then
    FIND FIRST Alert_Popups where Alert_Popups.Alert_Type = 9 and 
                  Alert_Popups.EventDateTime =  ldPropertyTime and 
                  Alert_Popups.Extension =   p_origination_ext# and 
                  Alert_Popups.propertyid = opiCompany_num no-lock no-error.
end.
if  avail Alert_Popups then
do:
    output to value("..\logs\DLL-911AlarmTrigger-" + string(month(today)) + "-" + string(day(today)) + "-" + string(year(today)) 
    + ".log" ) append.
    put unformatted "Duplicate found: "string(today) " " string(time,"hh:mm:ss") ","
            opiCompany_num ","
            BWcallStartTime ","
            BWdialedDigits ","
            BWuserId  ","
            tt911AlarmTrigger.BWuserName ","
            BWuserExtension ","
            BWuserPhone ","
            BWenterpriseId ","
            BWgroupId ","
            BWgroupName ","
            BWgroupAddress ","
            BWclidNumber ","
            BWclidName "," 
            BWsrcIpAdd "," 
            string(ldPropertyTime) skip.
    output close.
    assign session:date-format = l_cdateformat.
    opResult  = yes. 
    return.
end.
```
- **Purpose**: Prevents duplicate alerts for the same 911 call.
  - For "peerless" or "ooma" PBX types, checks `Alert_Popups` by IP address (`BWsrcIpAdd`).
  - Otherwise, checks by time, extension, and property ID.
  - If a duplicate is found, logs it and exits with `opResult = yes`.

---

### 9. Generate 911 Alarm
```progress
opResult  = yes. 
define var Pbxuser as char no-undo.
Pbxuser = replace(BWclidName, "|", " "). 
run Generate_911_Alarms (input opiCompany_num , input p_origination_ext#,input ldPropertyTime, INPUT BWdialedDigits,0,Pbxuser) .
if  trim (BWsrcIpAdd ) ne "" and trim(BWsrcIpAdd) ne ?  then
do:
    FIND FIRST Alert_Popups where Alert_Popups.Alert_Type = 9 and 
                  Alert_Popups.EventDateTime =  ldPropertyTime and 
                  Alert_Popups.Extension =   p_origination_ext# and 
                  Alert_Popups.propertyid = opiCompany_num exclusive-lock no-wait no-error.
    if avail Alert_Popups and not locked(Alert_Popups) then Alert_Popups.AckIPAddress = BWsrcIpAdd.
    find current Alert_Popups no-lock no-error.
    release  Alert_Popups.
end.
```
- **Purpose**: Generates the 911 alarm and updates records.
  - Sets `opResult = yes` to indicate success.
  - Cleans the caller name (`BWclidName`) and runs the `Generate_911_Alarms` procedure with call details.
  - Updates the `Alert_Popups` record with the source IP address if available.

---

### 10. Final Return
```progress
return.
```
- **Purpose**: Ends the program execution after processing the 911 call alert.

---

## Summary
The Progress 4GL program processes 911 emergency calls from a PBX system by:
1. Setting up the environment and variables.
2. Identifying the property/company associated with the call.
3. Converting the call time to the property’s local time.
4. Checking for exemptions and duplicates.
5. Generating an alarm if all conditions are met.
6. Logging key steps for debugging and auditing.

Each section plays a critical role in ensuring that 911 alerts are handled accurately and efficiently, with appropriate error handling and logging throughout the process.
Based on your query asking for a "flow chart or pseudo code," I will provide the **pseudo-code** for a 911 Alarm Trigger Process, as it aligns with your request and allows me to present a clear, structured representation of the logic flow. Below is a detailed and self-contained pseudo-code that outlines the key steps of the process, written in a simplified, easy-to-understand format.

---

### Pseudo-Code for 911 Alarm Trigger Process

```plaintext
// Program: 911 Alarm Trigger Process
PROGRAM 911_Alarm_Trigger

// Purpose: Process emergency call data and generate a 911 alarm if conditions are met
// Input: Temp-table tt911AlarmTrigger (contains call details)
// Output: opResult (logical - true if alarm is generated, false otherwise)

// Initialize variables
SET opResult = false
SET opiCompany_num = 0

// Step 1: Check if input data exists
IF tt911AlarmTrigger has no data THEN
    RETURN "No data to process"

// Step 2: Extract key call details from temp-table
SET ipBWEnterprise = tt911AlarmTrigger.BWenterpriseId
SET ipBWGroup = tt911AlarmTrigger.BWgroupId
SET p_origination_ext# = tt911AlarmTrigger.BWuserExtension
SET BWdialedDigits = tt911AlarmTrigger.BWdialedDigits
SET BWcallStartTime = tt911AlarmTrigger.BWcallStartTime
SET BWsrcIpAdd = tt911AlarmTrigger.BWsrcIpAdd
SET BWclidName = tt911AlarmTrigger.BWclidName

// Step 3: Log the entry
LOG "Processing call" with timestamp, ipBWEnterprise, ipBWGroup, p_origination_ext#

// Step 4: Determine company number based on group ID
IF ipBWGroup NOT IN ["ooma-emergency", "peerless-emergency"] THEN
    RUN BWgetJazzCompanyNum procedure to set opiCompany_num
ELSE IF ipBWGroup = "peerless-emergency" THEN
    SET opiCompany_num = 0
    FOR each if-Interface-ComLib-Parameter where InterfaceGroup = "PBX" and AttrName = "BWEnterpriseCode"
        IF AttrValue matches ipBWEnterprise THEN
            SET opiCompany_num = company_num
            BREAK
    IF opiCompany_num = 0 THEN
        LOG "Property not found for peerless-emergency"
        RETURN
ELSE
    FIND BW-ExtensionToUserName_Map where BWuserId matches tt911AlarmTrigger.BWuserId
    IF found THEN
        SET opiCompany_num = Company_Num
        SET p_origination_ext# = Extension_Num
    ELSE
        FIND BW-ExtensionToLinePort_Map where BWuserId matches tt911AlarmTrigger.BWuserId
        IF found THEN
            SET opiCompany_num = Company_Num
            SET p_origination_ext# = Extension_Num
    IF opiCompany_num = 0 AND ipBWEnterprise is not empty THEN
        FIND if-Interface-ComLib-Parameter where AttrValue matches ipBWEnterprise
        IF found THEN
            SET opiCompany_num = company_num

// Step 5: Validate company number
IF opiCompany_num = 0 THEN
    LOG "Company/Property not found"
    RETURN

// Step 6: Check for exempt numbers
FIND parameter_master where Parameter_code = "EmergencyAlertExemptNumbers" 
    AND (company matches opiCompany_num OR is global)
IF found AND BWdialedDigits is in exempt list THEN
    LOG "Dialed number is exempt, no alert generated"
    RETURN

// Step 7: Convert call time to property local time
SET ldPropertyTime = BWcallStartTime
FIND ServerTZMapping where property matches opiCompany_num
IF found THEN
    SET l_cPropertyZone = TZoffset
    CONVERT ldPropertyTime to property time using l_cPropertyZone
ELSE
    FIND ext_company_num where key = "TIME-DIFF" and company matches opiCompany_num
    IF found THEN
        ADJUST ldPropertyTime by time difference in hours

// Step 8: Log converted time
LOG "Call time converted" with ldPropertyTime

// Step 9: Determine Cloud PBX type
FIND parameter_master where Parameter_code = "CloudPBXType" and company matches opiCompany_num
SET CloudPBXType = parameter value (default to empty if not found)

// Step 10: Check for duplicate alerts
IF CloudPBXType IN ["peerless", "ooma"] AND BWsrcIpAdd is not empty THEN
    FIND Alert_Popups where Alert_Type = 9 
        AND propertyid = opiCompany_num 
        AND AckIPAddress = BWsrcIpAdd
    IF found THEN
        LOG "Duplicate alert found by IP"
        SET opResult = true
        RETURN
IF ipBWEnterprise <> "ooma-emergency" THEN
    FIND Alert_Popups where Alert_Type = 9 
        AND EventDateTime = ldPropertyTime 
        AND Extension = p_origination_ext# 
        AND propertyid = opiCompany_num
    IF found THEN
        LOG "Duplicate alert found by time and extension"
        SET opResult = true
        RETURN

// Step 11: Generate the 911 alarm
SET opResult = true
SET Pbxuser = cleaned BWclidName (remove special characters)
RUN Generate_911_Alarms with parameters:
    - Company Number: opiCompany_num
    - Extension: p_origination_ext#
    - Event Time: ldPropertyTime
    - Dialed Number: BWdialedDigits
    - Priority: 0
    - User: Pbxuser

// Step 12: Update Alert_Popups with IP address if available
IF BWsrcIpAdd is not empty THEN
    FIND Alert_Popups where Alert_Type = 9 and matches event details
    IF found THEN
        UPDATE AckIPAddress = BWsrcIpAdd

// Step 13: End of process
RETURN
```

---

### Explanation of the Pseudo-Code

This pseudo-code represents a process for triggering a 911 alarm based on emergency call data stored in a temp-table (`tt911AlarmTrigger`). Here’s a breakdown of the key steps:

1. **Initialization**: Sets up the output variable `opResult` and a company number variable `opiCompany_num`.
2. **Data Check**: Ensures the temp-table has data to process.
3. **Data Extraction**: Pulls critical fields like enterprise ID, group ID, extension, and call details from the input.
4. **Company Number Logic**: Determines the company/property number based on group ID and mappings, with fallback searches if needed.
5. **Exemption Check**: Skips processing if the dialed number is in an exempt list.
6. **Time Conversion**: Adjusts the call start time to the property’s local time using time zone data or a time difference.
7. **Duplicate Check**: Prevents duplicate alarms by checking existing records based on IP address or event details.
8. **Alarm Generation**: If all conditions are met, generates a 911 alarm with the relevant details.
9. **Record Update**: Updates the alarm record with the source IP address if provided.

This pseudo-code provides a clear, step-by-step flow of the logic, making it easy to follow the decision-making and actions involved in processing a 911 alarm trigger. Let me know if you'd prefer a flow chart instead!


The line `DEFINE INPUT PARAMETER TABLE FOR tt911AlarmTrigger` in the provided Progress 4GL code can be confusing if you're not familiar with Progress 4GL syntax or its concepts. Let me break it down clearly and explain its purpose, context, and meaning.

---

Below is a **flow chart** representation of the 911 Alarm Trigger Process based on the provided pseudo-code. Since I cannot directly render a graphical flow chart in this text-based format, I will describe the flow chart in a structured textual format using a hierarchical outline that mirrors the decision points, actions, and flow of the program. This description can be used to visualize or create a flow chart using tools like Lucidchart, Draw.io, or Microsoft Visio. Each step corresponds to the pseudo-code, and I’ll include connectors to show the flow.

---

### Flow Chart for 911 Alarm Trigger Process

#### Start
- **Symbol**: Oval
- **Description**: Start of the 911 Alarm Trigger Process
- **Action**: Begin execution of the program
- **Connector**: Arrow to "Initialize Variables"

#### Initialize Variables
- **Symbol**: Rectangle (Process)
- **Description**: Set initial values for variables
- **Action**:
  - `SET opResult = false`
  - `SET opiCompany_num = 0`
- **Connector**: Arrow to "Check Input Data"

#### Check Input Data
- **Symbol**: Diamond (Decision)
- **Description**: Check if `tt911AlarmTrigger` has data
- **Condition**: `IF tt911AlarmTrigger has no data`
  - **Yes**:
    - **Action**: `RETURN "No data to process"`
    - **Symbol**: Oval (End)
    - **Description**: Exit program
  - **No**:
    - **Connector**: Arrow to "Extract Call Details"

#### Extract Call Details
- **Symbol**: Rectangle (Process)
- **Description**: Extract key call details from temp-table
- **Action**:
  - `SET ipBWEnterprise = tt911AlarmTrigger.BWenterpriseId`
  - `SET ipBWGroup = tt911AlarmTrigger.BWgroupId`
  - `SET p_origination_ext# = tt911AlarmTrigger.BWuserExtension`
  - `SET BWdialedDigits = tt911AlarmTrigger.BWdialedDigits`
  - `SET BWcallStartTime = tt911AlarmTrigger.BWcallStartTime`
  - `SET BWsrcIpAdd = tt911AlarmTrigger.BWsrcIpAdd`
  - `SET BWclidName = tt911AlarmTrigger.BWclidName`
- **Connector**: Arrow to "Log Entry"

#### Log Entry
- **Symbol**: Rectangle (Process)
- **Description**: Log the call processing entry
- **Action**: `LOG "Processing call" with timestamp, ipBWEnterprise, ipBWGroup, p_origination_ext#`
- **Connector**: Arrow to "Determine Company Number"

#### Determine Company Number
- **Symbol**: Diamond (Decision)
- **Description**: Check `ipBWGroup` to determine company number
- **Condition**: `IF ipBWGroup NOT IN ["ooma-emergency", "peerless-emergency"]`
  - **Yes**:
    - **Action**: `RUN BWgetJazzCompanyNum procedure to set opiCompany_num`
    - **Symbol**: Rectangle (Process)
    - **Connector**: Arrow to "Validate Company Number"
  - **No**:
    - **Connector**: Arrow to "Check Peerless Emergency"

#### Check Peerless Emergency
- **Symbol**: Diamond (Decision)
- **Description**: Check if `ipBWGroup = "peerless-emergency"`
- **Condition**: `IF ipBWGroup = "peerless-emergency"`
  - **Yes**:
    - **Action**: `SET opiCompany_num = 0`
    - **Symbol**: Rectangle (Process)
    - **Subprocess**: Iterate `if-Interface-ComLib-Parameter` where `InterfaceGroup = "PBX" and AttrName = "BWEnterpriseCode"`
      - **Condition**: `IF AttrValue matches ipBWEnterprise`
        - **Action**: `SET opiCompany_num = company_num` and `BREAK`
      - **Symbol**: Rectangle (Process)
    - **Condition**: `IF opiCompany_num = 0`
      - **Yes**:
        - **Action**: `LOG "Property not found for peerless-emergency"`
        - **Action**: `RETURN`
        - **Symbol**: Oval (End)
      - **No**:
        - **Connector**: Arrow to "Validate Company Number"
  - **No**:
    - **Connector**: Arrow to "Check User Mapping"

#### Check User Mapping
- **Symbol**: Rectangle (Process)
- **Description**: Find company number using user mapping
- **Action**: `FIND BW-ExtensionToUserName_Map where BWuserId matches tt911AlarmTrigger.BWuserId`
- **Condition**: `IF found`
  - **Yes**:
    - **Action**: `SET opiCompany_num = Company_Num`
    - **Action**: `SET p_origination_ext# = Extension_Num`
    - **Connector**: Arrow to "Validate Company Number"
  - **No**:
    - **Connector**: Arrow to "Check Line Port Mapping"

#### Check Line Port Mapping
- **Symbol**: Rectangle (Process)
- **Description**: Find company number using line port mapping
- **Action**: `FIND BW-ExtensionToLinePort_Map where BWuserId matches tt911AlarmTrigger.BWuserId`
- **Condition**: `IF found`
  - **Yes**:
    - **Action**: `SET opiCompany_num = Company_Num`
    - **Action**: `SET p_origination_ext# = Extension_Num`
    - **Connector**: Arrow to "Validate Company Number"
  - **No**:
    - **Connector**: Arrow to "Fallback Company Lookup"

#### Fallback Company Lookup
- **Symbol**: Diamond (Decision)
- **Description**: Check if fallback lookup is needed
- **Condition**: `IF opiCompany_num = 0 AND ipBWEnterprise is not empty`
  - **Yes**:
    - **Action**: `FIND if-Interface-ComLib-Parameter where AttrValue matches ipBWEnterprise`
    - **Condition**: `IF found`
      - **Yes**:
        - **Action**: `SET opiCompany_num = company_num`
      - **No**: (No action, proceed)
    - **Connector**: Arrow to "Validate Company Number"
  - **No**:
    - **Connector**: Arrow to "Validate Company Number"

#### Validate Company Number
- **Symbol**: Diamond (Decision)
- **Description**: Check company number
- **Condition**: `IF opiCompany_num = 0`
  - **Yes**:
    - **Action**: `LOG "Company/Property not found"`
    - **Action**: `RETURN`
    - **Symbol**: Oval (End)
  - **No**:
    - **Connector**: Arrow to "Check Exempt Numbers"

#### Check Exempt Numbers
- **Symbol**: Rectangle (Process)
- **Description**: Check if dialed number is exempt
- **Action**: `FIND parameter_master where Parameter_code = "EmergencyAlertExemptNumbers" AND (company matches opiCompany_num OR is global)`
- **Condition**: `IF found AND BWdialedDigits is in exempt list`
  - **Yes**:
    - **Action**: `LOG "Dialed number is exempt, no alert generated"`
    - **Action**: `RETURN`
    - **Symbol**: Oval (End)
  - **No**:
    - **Connector**: Arrow to "Convert Call Time"

#### Convert Call Time
- **Symbol**: Rectangle (Process)
- **Description**: Convert call time to property local time
- **Action**: `SET ldPropertyTime = BWcallStartTime`
- **Subprocess**: `FIND ServerTZMapping where property matches opiCompany_num`
- **Condition**: `IF found`
  - **Yes**:
    - **Action**: `SET l_cPropertyZone = TZoffset`
    - **Action**: `CONVERT ldPropertyTime to property time using l_cPropertyZone`
    - **Connector**: Arrow to "Log Converted Time"
  - **No**:
    - **Action**: `FIND ext_company_num where key = "TIME-DIFF" and company matches opiCompany_num`
    - **Condition**: `IF found`
      - **Yes**:
        - **Action**: `ADJUST ldPropertyTime by time difference in hours`
      - **No**: (No action, proceed)
    - **Connector**: Arrow to "Log Converted Time"

#### Log Converted Time
- **Symbol**: Rectangle (Process)
- **Description**: Log the converted call time
- **Action**: `LOG "Call time converted" with ldPropertyTime`
- **Connector**: Arrow to "Determine Cloud PBX Type"

#### Determine Cloud PBX Type
- **Symbol**: Rectangle (Process)
- **Description**: Determine Cloud PBX type
- **Action**: `FIND parameter_master where Parameter_code = "CloudPBXType" and company matches opiCompany_num`
- **Action**: `SET CloudPBXType = parameter value (default to empty if not found)`
- **Connector**: Arrow to "Check Duplicate Alerts (IP)"

#### Check Duplicate Alerts (IP)
- **Symbol**: Diamond (Decision)
- **Description**: Check for duplicate alerts by IP
- **Condition**: `IF CloudPBXType IN ["peerless", "ooma"] AND BWsrcIpAdd is not empty`
  - **Yes**:
    - **Action**: `FIND Alert_Popups where Alert_Type = 9 AND propertyid = opiCompany_num AND AckIPAddress = BWsrcIpAdd`
    - **Condition**: `IF found`
      - **Yes**:
        - **Action**: `LOG "Duplicate alert found by IP"`
        - **Action**: `SET opResult = true`
        - **Action**: `RETURN`
        - **Symbol**: Oval (End)
      - **No**:
        - **Connector**: Arrow to "Check Duplicate Alerts (Time/Extension)"
  - **No**:
    - **Connector**: Arrow to "Check Duplicate Alerts (Time/Extension)"

#### Check Duplicate Alerts (Time/Extension)
- **Symbol**: Diamond (Decision)
- **Description**: Check for duplicate alerts by time and extension
- **Condition**: `IF ipBWEnterprise <> "ooma-emergency"`
  - **Yes**:
    - **Action**: `FIND Alert_Popups where Alert_Type = 9 AND EventDateTime = ldPropertyTime AND Extension = p_origination_ext# AND propertyid = opiCompany_num`
    - **Condition**: `IF found`
      - **Yes**:
        - **Action**: `LOG "Duplicate alert found by time and extension"`
        - **Action**: `SET opResult = true`
        - **Action**: `RETURN`
        - **Symbol**: Oval (End)
      - **No**:
        - **Connector**: Arrow to "Generate 911 Alarm"
  - **No**:
    - **Connector**: Arrow to "Generate 911 Alarm"

#### Generate 911 Alarm
- **Symbol**: Rectangle (Process)
- **Description**: Generate the 911 alarm
- **Action**: `SET opResult = true`
- **Action**: `SET Pbxuser = cleaned BWclidName (remove special characters)`
- **Action**: `RUN Generate_911_Alarms with parameters:`
  - Company Number: `opiCompany_num`
  - Extension: `p_origination_ext#`
  - Event Time: `ldPropertyTime`
  - Dialed Number: `BWdialedDigits`
  - Priority: `0`
  - User: `Pbxuser`
- **Connector**: Arrow to "Update Alert_Popups"

#### Update Alert_Popups
- **Symbol**: Diamond (Decision)
- **Description**: Check if IP address needs updating
- **Condition**: `IF BWsrcIpAdd is not empty`
  - **Yes**:
    - **Action**: `FIND Alert_Popups where Alert_Type = 9 and matches event details`
    - **Condition**: `IF found`
      - **Yes**:
        - **Action**: `UPDATE AckIPAddress = BWsrcIpAdd`
      - **No**: (No action, proceed)
    - **Connector**: Arrow to "End"
  - **No**:
    - **Connector**: Arrow to "End"

#### End
- **Symbol**: Oval
- **Description**: End of the 911 Alarm Trigger Process
- **Action**: `RETURN`

---

### How to Visualize This Flow Chart

To create a visual flow chart, you can use a diagramming tool (e.g., Lucidchart, Draw.io, or Visio) and follow these guidelines:

1. **Shapes**:
   - **Oval**: Start and End points.
   - **Rectangle**: Process steps (e.g., `SET`, `LOG`, `RUN`).
   - **Diamond**: Decision points (e.g., `IF` conditions).
   - **Arrow**: Connectors showing the flow between steps.

2. **Structure**:
   - Start at the top with the "Start" oval.
   - Follow the linear flow for initial steps (Initialize Variables, Check Input Data).
   - Branch out at decision points (e.g., Check Input Data, Determine Company Number) with "Yes" and "No" paths.
   - Merge paths back where appropriate (e.g., after company number determination).
   - End with the "End" oval, which can be reached from multiple exit points (e.g., RETURN statements).

3. **Labels**:
   - Label each shape with the action or condition (e.g., "SET opResult = false", "IF tt911AlarmTrigger has no data").
   - Include brief descriptions for clarity (e.g., "Log call details").

4. **Example Layout**:
   ```
   [Start]
      |
      v
   [Initialize Variables]
      |
      v
   [Check Input Data] ----> [No data to process] --> [End]
      |
      v
   [Extract Call Details]
      |
      v
   [Log Entry]
      |
      v
   [Determine Company Number] --> [Yes: Run BWgetJazzCompanyNum] --> [Validate Company Number]
      |                           |
      |                           v
      |                        [Check Peerless Emergency] --> [Yes: Process peerless] --> [End if opiCompany_num = 0]
      |                           |                              |
      |                           v                              v
      |                        [Check User Mapping] ----------> [Validate Company Number]
      |                           |
      |                           v
      |                        [Check Line Port Mapping] -----> [Validate Company Number]
      |                           |
      |                           v
      |                        [Fallback Company Lookup] -----> [Validate Company Number]
      |
      v
   [Validate Company Number] --> [opiCompany_num = 0] --> [End]
      |
      v
   [Check Exempt Numbers] --> [Number is exempt] --> [End]
      |
      v
   [Convert Call Time]
      |
      v
   [Log Converted Time]
      |
      v
   [Determine Cloud PBX Type]
      |
      v
   [Check Duplicate Alerts (IP)] --> [Duplicate found] --> [End]
      |
      v
   [Check Duplicate Alerts (Time/Extension)] --> [Duplicate found] --> [End]
      |
      v
   [Generate 911 Alarm]
      |
      v
   [Update Alert_Popups]
      |
      v
   [End]
   ```

---

### Notes

- **Complexity**: The flow chart is moderately complex due to multiple decision points and branches, especially in the company number determination step. Ensure the diagram is clear by grouping related steps (e.g., user and line port mapping) and using color coding for paths.
- **Pseudo-Code Alignment**: The flow chart directly follows the pseudo-code structure, ensuring every step is represented.
- **Exit Points**: The program has multiple exit points (e.g., no data, company not found, exempt number, duplicates), each leading to the "End" oval.

If you’d like a more detailed description of specific sections, a simplified version of the flow chart, or assistance with creating the diagram in a specific tool, let me know! Alternatively, I can generate a textual representation using ASCII art for a basic visualization, though it would be less detailed.

### What Does `DEFINE INPUT PARAMETER TABLE FOR tt911AlarmTrigger` Mean?

In Progress 4GL (also known as OpenEdge ABL), this statement defines an **input parameter** for a program or procedure. Specifically, it indicates that the program expects a **temp-table** (a temporary, in-memory table) named `tt911AlarmTrigger` to be passed as input when the program is called.

Here’s a detailed explanation:

1. **`DEFINE INPUT PARAMETER`**:
   - This part of the statement declares that the program is expecting an input parameter. Input parameters are values or data structures passed to a program or procedure when it is invoked.
   - `INPUT` specifies that the parameter is read-only (the program can read the data but not modify it to send back to the caller).

2. **`TABLE FOR`**:
   - The `TABLE FOR` clause indicates that the parameter is not a simple variable (like an integer or string) but an entire **temp-table**.
   - A temp-table in Progress 4GL is a temporary, in-memory table used to store structured data (rows and columns) during program execution. It’s similar to a database table but exists only for the duration of the program or session.

3. **`tt911AlarmTrigger`**:
   - This is the name of the temp-table being passed as the input parameter.
   - The structure of `tt911AlarmTrigger` is defined elsewhere, likely in the included file `{tt911AlarmTrigger.i}`, which specifies the fields (columns) of the temp-table, such as `BWcallStartTime`, `BWdialedDigits`, `BWuserId`, etc.

4. **Purpose**:
   - This statement tells the program that it will receive a temp-table named `tt911AlarmTrigger` containing data about a 911 emergency call (e.g., call time, user details, dialed digits, etc.).
   - The program uses this temp-table to process the call data, perform validations, and generate an alarm if necessary.

---

### Context in the Code

The `tt911AlarmTrigger` temp-table is the primary input to the program. It contains details about a 911 call, such as:

- `BWcallStartTime`: When the call was made.
- `BWdialedDigits`: The number dialed (e.g., "911").
- `BWuserId`, `BWuserName`, `BWuserExtension`: Information about the caller.
- `BWenterpriseId`, `BWgroupId`: Identifiers for the enterprise and group in the PBX system.

The temp-table is likely populated by another part of the system (e.g., a PBX interface) and passed to this program for processing. The program reads the data in `tt911AlarmTrigger` to:

1. Identify the company/property associated with the call.
2. Convert the call time to the property’s local time.
3. Check for exemptions or duplicates.
4. Generate a 911 alarm if appropriate.

The `DEFINE INPUT PARAMETER TABLE FOR tt911AlarmTrigger` ensures that the program knows to expect this temp-table as input and can access its records and fields.

---

### Why Is This Needed?

In Progress 4GL, programs and procedures often need to work with structured data, such as multiple rows of call details. Temp-tables are a common way to pass such data between programs. By defining `tt911AlarmTrigger` as an input parameter:

- The program can process one or more rows of 911 call data in a structured format.
- The temp-table’s schema (fields and indexes) is predefined, ensuring consistency in how the data is accessed.
- The `INPUT` keyword ensures the program treats the temp-table as read-only, preventing accidental modifications to the caller’s data.

---

### Example to Clarify

Imagine this program is called from another part of the system. The calling program might look like this:

```progress
/* Caller program */
DEFINE TEMP-TABLE tt911AlarmTrigger NO-UNDO
    FIELD BWcallStartTime AS DATETIME
    FIELD BWdialedDigits AS CHARACTER
    FIELD BWuserId AS CHARACTER
    /* ... other fields ... */.

CREATE tt911AlarmTrigger.
ASSIGN
    tt911AlarmTrigger.BWcallStartTime = NOW
    tt911AlarmTrigger.BWdialedDigits = "911"
    tt911AlarmTrigger.BWuserId = "user123".

RUN 911AlarmTrigger.p (INPUT TABLE tt911AlarmTrigger, OUTPUT opResult).
```

In this example:
- The caller defines and populates the `tt911AlarmTrigger` temp-table.
- It passes the temp-table to the `911AlarmTrigger.p` program (the program containing your code).
- The `DEFINE INPUT PARAMETER TABLE FOR tt911AlarmTrigger` in `911AlarmTrigger.p` allows the program to receive and process this temp-table.

---

### Common Points of Confusion

1. **Where is the temp-table defined?**
   - The structure of `tt911AlarmTrigger` is defined in the included file `{tt911AlarmTrigger.i}`. This file likely contains a `DEFINE TEMP-TABLE` statement with fields like `BWcallStartTime`, `BWdialedDigits`, etc.
   - The actual data in the temp-table is provided by the calling program or system.

2. **Why use a temp-table instead of individual parameters?**
   - A temp-table allows the program to handle multiple rows of data or complex structures efficiently. For example, it could process multiple 911 calls in a single call if needed.
   - It also encapsulates related data (e.g., call time, user ID, extension) in one structure, making the code cleaner.

3. **What does `TABLE FOR` mean?**
   - It specifically indicates that the parameter is a temp-table, not a single value or a database table. This is a Progress 4GL convention for passing temp-tables as parameters.

---

### In the Context of Your Code

In your code, the `tt911AlarmTrigger` temp-table is used immediately after the parameter definition:

```progress
find first tt911AlarmTrigger no-lock no-error.
if not avail tt911AlarmTrigger then return.
```

- The `FIND FIRST` statement checks if there’s at least one row in the temp-table.
- If no data is found, the program exits (`return`), indicating there’s nothing to process.
- If data is found, the program extracts fields like `BWenterpriseId`, `BWgroupId`, and `BWuserExtension` to proceed with processing.

This shows that `tt911AlarmTrigger` is the core data source for the program, and the `DEFINE INPUT PARAMETER TABLE FOR tt911AlarmTrigger` statement is what enables the program to receive this data from the caller.

---

### Summary

- `DEFINE INPUT PARAMETER TABLE FOR tt911AlarmTrigger` declares that the program expects a temp-table named `tt911AlarmTrigger` as a read-only input parameter.
- The temp-table contains 911 call details, defined in `{tt911AlarmTrigger.i}`.
- It allows the program to process structured call data passed from another part of the system.
- Think of it as the program saying, “I need a temp-table called `tt911AlarmTrigger` to work with, and it’s coming from the caller.”

If you have further questions about this statement or need clarification on related concepts (e.g., temp-tables, include files, or how the temp-table is populated), let me know!

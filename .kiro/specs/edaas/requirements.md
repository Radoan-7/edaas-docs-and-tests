# Requirements Document

## Introduction

The Emergency Drug Allergy Alert System (EDAAS) is an educational decision-support prototype
built for a software engineering course (Group 9). It helps healthcare staff identify
drug-allergy conflicts before a medication is dispensed, ensuring patient safety through
a fail-safe alert and override workflow.

All data is fictional. The system is not for clinical use. Healthcare professionals always
make the final decision; EDAAS never replaces clinical judgment.

The system is built on React + TypeScript + Tailwind + shadcn/ui for the frontend and
Supabase (Auth, Postgres, RLS, RPC) for the backend. Role enforcement is performed
server-side via a `SECURITY DEFINER has_role()` function; the client is never trusted
for role decisions.

---

## Glossary

- **EDAAS**: Emergency Drug Allergy Alert System — the system described in this document.
- **Patient**: An individual whose allergy records and medication orders are managed in EDAAS.
- **Doctor**: A clinical staff member with authority to verify allergies and override conflict warnings.
- **Nurse**: A clinical staff member with read-only access to patient allergy information.
- **Pharmacist**: A staff member who initiates medication checks and performs dispensing.
- **Admin**: A system administrator who manages user accounts, assigns roles, and views the audit log.
- **Allergy_Record**: A data record linking a Patient to an allergen, reaction type, severity, and verification status.
- **Allergen**: The substance a patient is allergic to (e.g., drug name, ingredient, or drug class).
- **Medication**: A drug order presented for dispensing, characterised by its name, active ingredient, and drug class.
- **Conflict**: A match between a patient's Allergen and the Medication's name, active ingredient, or drug class (case-insensitive).
- **Conflict_Alert**: A full-screen modal displayed when a Conflict is detected; it is never a toast notification.
- **Override**: A doctor-authorised action that allows a flagged Medication to be dispensed despite a Conflict, accompanied by a typed reason.
- **Verification_Status**: One of `unverified`, `verified`, or `rejected`, describing whether a doctor has reviewed an Allergy_Record.
- **Audit_Log**: An immutable, append-only record of every significant action performed in EDAAS.
- **RLS**: Row-Level Security — Postgres policies that restrict data access per role.
- **RPC**: Remote Procedure Call — a Supabase database function invoked from the frontend.
- **user_roles**: A Supabase database table that stores role assignments for each user.
- **has_role()**: A `SECURITY DEFINER` Postgres function that enforces server-side role checks.
- **V-Model**: A development process where each build stage is paired with a corresponding test stage to maintain traceability.

---

## Requirements

### Requirement 1: Register Allergy

**User Story:** As a patient or staff member acting on behalf of a patient, I want to add an allergy record for a patient, so that the system can identify potential drug-allergy conflicts during medication checks.

#### Acceptance Criteria

1. WHEN a user submits an allergy registration form, THE EDAAS SHALL require a valid patient identifier, allergen name (1–100 characters), reaction type (one of: rash, swelling, anaphylaxis, nausea, or respiratory distress), and severity (mild, moderate, or severe) before saving the record.
2. WHEN a new Allergy_Record is saved, THE EDAAS SHALL assign the Verification_Status "unverified" to that record by default.
3. WHEN an allergy registration form is submitted with one or more required fields empty or invalid, THE EDAAS SHALL display a field-level validation error message indicating which fields are missing or invalid, and SHALL NOT save the record.
4. IF an Allergy_Record with the same patient identifier and allergen name already exists, THEN THE EDAAS SHALL reject the submission and display an error message indicating a duplicate allergy record exists for that patient.

---

### Requirement 2: Check Medication

**User Story:** As a doctor or pharmacist, I want to initiate an allergy check for a patient's medication, so that I can identify any drug-allergy conflict before dispensing.

#### Acceptance Criteria

1. WHEN a doctor or pharmacist submits a medication check request, THE EDAAS SHALL compare the Medication's name, active ingredient, and drug class against all of the Patient's Allergen values whose Verification_Status is "unverified" or "verified", using case-insensitive string comparison; Allergy_Records with Verification_Status "rejected" SHALL be excluded from the comparison.
2. WHEN a case-insensitive match is found between the Medication and any eligible Allergen value, THE EDAAS SHALL classify the result as a Conflict, disable the dispense button, and display an alert identifying the matched Allergen value, the Medication field (name, active ingredient, or drug class) that produced the match, and the Verification_Status of the matched Allergy_Record (so that an "unverified" match is clearly labelled as such).
3. WHEN no case-insensitive match is found between the Medication and any eligible Allergen value, THE EDAAS SHALL classify the result as "no conflict" and display a "Safe to proceed" message, then enable the dispense button.
4. WHEN a medication check is performed for a Patient with zero Allergy_Records, THE EDAAS SHALL classify the result as "no conflict", display both a "Safe to proceed" message and a "No allergies recorded for this patient" notice, and enable the dispense button without raising an error.
5. WHEN an allergy check RPC call is initiated, THE EDAAS SHALL return the conflict classification result within 5 seconds.
6. IF the allergy check RPC call does not return a result within 5 seconds, THEN THE EDAAS SHALL disable the dispense button and display an error message indicating that the check timed out.
7. IF the allergy check RPC call returns an error response, THEN THE EDAAS SHALL disable the dispense button and display an error message indicating that the check failed.
8. IF the network is unavailable when an allergy check is initiated, THEN THE EDAAS SHALL disable the dispense button and display an error message indicating that the network is unavailable.
9. IF a user who does not hold the "doctor" or "pharmacist" role attempts to initiate a medication check, THEN THE EDAAS SHALL reject the request with an authorisation error.

---

### Requirement 3: Display Alert

**User Story:** As a pharmacist or doctor, I want to see an unmistakable conflict alert when a drug-allergy conflict is detected, so that no conflicting medication is dispensed inadvertently.

#### Acceptance Criteria

1. WHEN a Conflict is detected, THE EDAAS SHALL display a Conflict_Alert as a full-screen modal dialog.
2. WHEN a Conflict_Alert is displayed, THE EDAAS SHALL NOT use a toast notification to communicate the conflict.
3. WHEN a Conflict_Alert is displayed, THE EDAAS SHALL NOT show the text "Safe to proceed" anywhere on the screen.
4. WHEN a Conflict_Alert is displayed, THE EDAAS SHALL include the patient name, allergen name, medication name, conflict reason (name match, ingredient match, or class match), allergy severity (mild, moderate, or severe), and Verification_Status (unverified, verified, or rejected).
5. WHILE a Conflict_Alert is displayed, THE EDAAS SHALL keep the dispense button disabled until a doctor performs a valid Override for that specific check.
6. WHEN a pharmacist cancels the Conflict_Alert dialog, THE EDAAS SHALL close the dialog but SHALL keep the dispense button disabled; cancelling the dialog does NOT enable dispensing.
7. WHEN a valid Override is recorded for the specific check while a Conflict_Alert is displayed, THE EDAAS SHALL record the doctor's name, UTC timestamp, and typed reason, then dismiss the alert and enable the dispense button for that check.
8. IF the allergy check fails, times out (after 5 seconds), or the network is unavailable, THEN THE EDAAS SHALL display a full-screen blocking dialog indicating the specific reason for the failure, keep the dispense button disabled, and require an explicit user action to dismiss the dialog.

---

### Requirement 4: Override Warning

**User Story:** As a doctor, I want to override a conflict warning with a documented reason, so that I can authorise the dispensing of a flagged medication when clinically justified.

#### Acceptance Criteria

1. IF a user who does not hold the "doctor" role attempts to perform an Override, THEN THE EDAAS SHALL reject the action and display an authorisation error message.
2. IF no active, unresolved allergy conflict exists for the prescription, THEN THE EDAAS SHALL reject any Override attempt and display an error indicating no conflict is present.
3. WHEN a doctor submits an Override, THE EDAAS SHALL require a typed reason between 10 and 500 characters (inclusive).
4. IF a doctor submits an Override with a reason fewer than 10 characters or more than 500 characters, THEN THE EDAAS SHALL display a validation error and SHALL NOT save the Override.
5. WHEN a valid Override is saved, THE EDAAS SHALL record the doctor's user ID, full name, UTC timestamp, and typed reason alongside the Override.
6. WHEN a valid Override is saved, THE EDAAS SHALL enable the dispense action for the pharmacist specifically for the medication check to which the Override is attached; the enablement SHALL NOT apply to any other check.
7. THE EDAAS SHALL NOT expose an override action in the UI to admin, nurse, pharmacist, or patient roles.

---

### Requirement 5: Manage Allergy Database

**User Story:** As an admin, I want to add, edit, and delete medication records and correct allergy records, so that the medication database and patient allergy data are accurate and up to date.

#### Acceptance Criteria

1. WHEN an admin submits a new medication record, THE EDAAS SHALL require a medication name (1–100 characters), active ingredient (1–100 characters), and drug class (1–100 characters) before saving.
2. WHEN an admin edits a medication record, THE EDAAS SHALL update the stored medication name, active ingredient, or drug class; the updated values SHALL take effect on the next allergy check that references that medication.
3. WHEN an admin deletes a medication record, THE EDAAS SHALL remove it from the medication database; the deletion SHALL take effect on the next allergy check that would have referenced that medication.
4. WHEN an admin edits an allergy record (for example to correct a data entry error), THE EDAAS SHALL update the stored allergen name, reaction type, or severity; the updated values SHALL take effect on the next allergy check for the affected patient.
5. IF a user who does not hold the "admin" role attempts to add, edit, or delete a medication record or edit an allergy record via the API, THEN THE EDAAS SHALL reject the request with an authorisation error.
6. WHEN an admin adds, edits, or deletes a medication record or edits an allergy record, THE EDAAS SHALL append an entry to the Audit_Log recording the actor's user ID, action type, target resource, previous values (for edit/delete), new values (for add/edit), and a UTC timestamp.

---

### Requirement 6: Dispense or Block Medication

**User Story:** As a pharmacist, I want to dispense a medication after a conflict-free check or a valid doctor override, so that I can complete the medication workflow with a reliable safety record.

#### Acceptance Criteria

1. WHEN a pharmacist initiates a dispense action after a "no conflict" result, THE EDAAS SHALL permit the dispense and SHALL record the event in the Audit_Log with the pharmacist's identity, timestamp, medication identifier, and a reference to the allergy check result.
2. WHEN a pharmacist initiates a dispense action after a valid Override is on record, THE EDAAS SHALL permit the dispense and SHALL record the event in the Audit_Log with the pharmacist's identity, timestamp, medication identifier, and a reference to the Override record.
3. WHILE an unresolved Conflict exists for the current medication check and no valid Override is on record, THE EDAAS SHALL keep the dispense control disabled and SHALL NOT process any dispense action.
4. IF a dispense action is attempted without a completed allergy check result or a valid Override on record for the current medication check, THEN THE EDAAS SHALL block the dispense action, display a prominent alert (not a dismissible toast) indicating the reason for the block, and preserve the current medication check state without modification.

---

### Requirement 7: View Allergy History

**User Story:** As a doctor, pharmacist, or nurse, I want to view the full allergy history for a patient, so that I can review all recorded allergens and their verification status during patient care.

#### Acceptance Criteria

1. WHEN a doctor, pharmacist, or nurse requests the allergy history for a Patient that exists in the system, THE EDAAS SHALL return all Allergy_Records for that Patient ordered by record creation date, from most recent to oldest.
2. WHEN allergy history is displayed, THE EDAAS SHALL show, for each record: allergen name, reaction type, severity, Verification_Status, and the identifier of the user who added the record.
3. WHEN a doctor, pharmacist, or nurse requests the allergy history for a Patient that exists in the system and no Allergy_Records are found for that Patient, THE EDAAS SHALL return an empty list with an indication that no allergy records exist for that Patient.
4. IF a Patient identifier in an allergy history request does not match any existing Patient, THEN THE EDAAS SHALL reject the request with an error indicating the Patient was not found.
5. IF a request for allergy history is made without valid authentication credentials, THEN THE EDAAS SHALL reject the request with an authentication error.
6. IF a request for allergy history is made by an authenticated user who does not hold the "doctor", "pharmacist", or "nurse" role, THEN THE EDAAS SHALL reject the request with an authorisation error.

---

### Requirement 8: Patient Self-Reported Allergies

**User Story:** As a patient, I want to view my own allergy records and add new self-reported allergy entries, so that my care team has up-to-date information about my allergies.

#### Acceptance Criteria

1. WHEN a patient requests their own allergy history, THE EDAAS SHALL return all Allergy_Records associated with that patient's identifier.
2. WHEN a patient submits a new self-reported allergy entry, THE EDAAS SHALL require an allergen name (1–100 characters), reaction type (one of: rash, swelling, anaphylaxis, nausea, or respiratory distress), and severity (mild, moderate, or severe), then save the record with Verification_Status "unverified".
3. IF a patient attempts to view allergy records for any patient other than themselves, THEN THE EDAAS SHALL reject the request with an authorisation error.
4. IF a patient attempts to edit or delete any allergy record, THEN THE EDAAS SHALL reject the request with an authorisation error.
5. IF a patient attempts to access medication check, override, or dispense functions, THEN THE EDAAS SHALL reject the request with an authorisation error.

---

### Requirement 9: Verify Allergy

**User Story:** As a doctor, I want to confirm or reject a patient's self-reported allergy, so that the system accurately reflects which allergies have been clinically reviewed.

#### Acceptance Criteria

1. WHEN a doctor confirms an Allergy_Record, THE EDAAS SHALL change the Verification_Status of that record to "verified".
2. WHEN a doctor rejects an Allergy_Record, THE EDAAS SHALL change the Verification_Status of that record to "rejected".
3. WHEN a doctor confirms or rejects an Allergy_Record, THE EDAAS SHALL atomically record the doctor's user ID, full name, and timestamp alongside the Verification_Status change in the same operation, such that no status change is persisted without its associated audit data.
4. IF a user who does not hold the "doctor" role attempts to verify or reject an Allergy_Record via the API, THEN THE EDAAS SHALL reject the action with an error response indicating insufficient permissions.
5. THE EDAAS SHALL NOT expose verify or reject action controls in the UI to nurse, pharmacist, admin, or patient roles.
6. IF a doctor attempts to verify or reject an Allergy_Record whose Verification_Status is not "unverified", THEN THE EDAAS SHALL reject the action with an error response indicating the record has already been reviewed.

---

### Requirement 10: Nurse Read-Only View

**User Story:** As a nurse, I want to view patient allergy records, so that I can review allergy information during patient care without being able to modify any data.

#### Acceptance Criteria

1. WHEN a nurse accesses the patient allergy view, THE EDAAS SHALL display all Allergy_Records for the selected Patient in a read-only format.
2. THE EDAAS SHALL NOT render action controls for medication check, override, dispense, or allergy editing in the nurse's UI.
3. WHEN a nurse attempts to perform a check, override, dispense, or edit action via the API, THE EDAAS SHALL reject the request with an authorisation error.

---

### Requirement 11: Manage User Accounts

**User Story:** As an admin, I want to create staff accounts and assign roles, so that authorised personnel can access EDAAS with the correct permissions.

#### Acceptance Criteria

1. WHEN an admin creates a new staff account, THE EDAAS SHALL require a role selection of "doctor", "nurse", or "pharmacist".
2. WHEN a new staff account is created, THE EDAAS SHALL persist the assigned role so that the account's permissions reflect that role on all subsequent authenticated requests.
3. WHEN an admin deactivates a staff account, THE EDAAS SHALL prevent that account from authenticating, and IF that account has an active session at the time of deactivation, THEN THE EDAAS SHALL immediately terminate that session.
4. THE EDAAS SHALL NOT present "admin" as a selectable role option in the account creation UI, so that privilege escalation via the UI is not possible.
5. IF a request to assign the "admin" role is received by the server, THEN THE EDAAS SHALL reject that request with an error response indicating an unauthorised role assignment, regardless of the requestor's role.
6. WHEN an admin submits a new staff account creation form, THE EDAAS SHALL require a full name (1–100 characters), a unique email address (valid format, maximum 254 characters), a password (8–128 characters), and a role selection before creating the account.

---

### Requirement 12: Audit Log

**User Story:** As an admin, I want to view an immutable log of every significant action in EDAAS, so that I can audit system activity and investigate any safety-relevant events.

#### Acceptance Criteria

1. WHEN a medication check is completed, THE EDAAS SHALL append an entry to the Audit_Log containing the actor's user ID, the actor's role at the time of the action, the action type, the target Patient and Medication identifiers, and a timestamp recorded in ISO 8601 format with second-level precision in UTC.
2. WHEN an Override is saved, THE EDAAS SHALL append an entry to the Audit_Log containing the actor's user ID, the actor's role, the action type, the Override reason, the target resource, and a timestamp recorded in ISO 8601 format with second-level precision in UTC.
3. WHEN a dispense or block action is recorded, THE EDAAS SHALL append an entry to the Audit_Log containing the actor's user ID, the actor's role, the action type, the target resource, and a timestamp recorded in ISO 8601 format with second-level precision in UTC.
4. WHEN an Allergy_Record verification action is saved, THE EDAAS SHALL append an entry to the Audit_Log containing the actor's user ID, the actor's role, the action type, the target Allergy_Record, and a timestamp recorded in ISO 8601 format with second-level precision in UTC.
5. WHEN a role assignment or account status change is made, THE EDAAS SHALL append an entry to the Audit_Log containing the actor's user ID, the action type, the affected user, and a timestamp recorded in ISO 8601 format with second-level precision in UTC.
6. WHEN an admin requests the Audit_Log, THE EDAAS SHALL display all log entries to the admin ordered by timestamp in descending order, with the most recent entry first.
7. IF appending an Audit_Log entry fails, THEN THE EDAAS SHALL reject the triggering action, preserve the system state as it was before the action was attempted, and present the actor with an error message indicating that the action could not be completed due to a logging failure.
8. WHEN a user who does not hold the "admin" role requests the Audit_Log via the API, THE EDAAS SHALL reject the request with an authorisation error.
9. THE EDAAS SHALL NOT provide any API endpoint or UI control that allows an Audit_Log entry to be edited or deleted.

---

## Non-Functional Requirements

### Requirement 13: Performance

**User Story:** As a pharmacist, I want allergy checks to complete quickly, so that the medication workflow is not unnecessarily delayed.

#### Acceptance Criteria

1. WHEN an allergy check RPC call is made with up to 10 concurrent users each submitting up to 1 request per second, THE EDAAS SHALL return the allergy check result within 5 seconds, with at least 95% of responses returned within 2 seconds.
2. WHEN the RPC response is received by the frontend, THE EDAAS SHALL render the Conflict_Alert or the safe result visible to the pharmacist within 500 milliseconds.
3. IF an allergy check RPC call does not receive a response within 5 seconds, THEN THE EDAAS SHALL display an error message indicating that the check timed out and prompt the pharmacist to retry.

---

### Requirement 14: Safety and Fail-Safe Behaviour

**User Story:** As a system operator, I want EDAAS to block medication dispensing whenever a check cannot be completed, so that no medication is dispensed without a confirmed safe result or authorised override.

#### Acceptance Criteria

1. IF the allergy check RPC call does not return a response within 5 seconds, THEN THE EDAAS SHALL block the dispense button and display a warning message indicating that the allergy check could not be completed.
2. IF the allergy check RPC call returns a non-success response, THEN THE EDAAS SHALL block the dispense button and display a warning message indicating that the allergy check returned an error.
3. IF the network is unavailable at the time an allergy check is initiated, THEN THE EDAAS SHALL block the dispense button and display a warning message indicating that the allergy check could not be reached.
4. WHILE an unresolved Conflict exists, THE EDAAS SHALL keep the dispense button disabled.
5. THE EDAAS SHALL never permit a Medication to be dispensed without either a "no conflict" result or a valid Override on record for that check.
6. WHEN a fail-safe condition is resolved — by the allergy check completing successfully, the network becoming available, or a valid Override being recorded — THE EDAAS SHALL re-enable the dispense button.

---

### Requirement 15: Security

**User Story:** As a system operator, I want all role checks to be enforced server-side, so that a malicious client cannot bypass access controls.

#### Acceptance Criteria

1. WHEN a role-restricted action is requested, THE EDAAS SHALL evaluate the caller's role exclusively via the server-side `has_role()` SECURITY DEFINER function before permitting the action.
2. IF `has_role()` returns false for the requested action, THEN THE EDAAS SHALL reject the request and return an error response indicating insufficient permissions, without performing the restricted action.
3. THE EDAAS SHALL apply RLS policies to every database table so that each role (doctor, nurse, pharmacist, patient, admin) can only read or write rows that belong to their permitted scope as defined in the RLS policy matrix.
4. IF a database operation violates an RLS policy, THEN THE EDAAS SHALL reject the operation and return an error response indicating access denied, without returning or modifying any restricted row.
5. THE EDAAS SHALL NOT store secrets — including database connection strings, API keys, JWT secrets, and service role keys — as plaintext in any file committed to the source code repository.

---

### Requirement 16: Traceability and Quality

**User Story:** As a course assessor, I want every functional requirement to trace to a design element, an implementation task, and a test, so that V-Model compliance can be verified.

#### Acceptance Criteria

1. THE EDAAS documentation SHALL include a traceability matrix, located in the requirements document, that contains one row per functional requirement and columns mapping each requirement to at least one design element (a class, component, or sequence diagram step defined in the design document), at least one implementation task (a named task in the task list), and at least one test case (identified by test case ID).
2. THE EDAAS traceability matrix SHALL contain an entry for every functional requirement defined in the requirements document, with no functional requirement left unmapped across any of the three columns.
3. THE EDAAS codebase SHALL not contain committed file content that includes student ID numbers or personal names of individual contributors; all attribution within file content SHALL use the group identifier "Group 9".

---

## Traceability Matrix

| Requirement | Design Element | Implementation Task | Test Case |
|---|---|---|---|
| REQ-1 Register Allergy | TBD | TBD | TBD |
| REQ-2 Check Medication | TBD | TBD | TBD |
| REQ-3 Display Alert | TBD | TBD | TBD |
| REQ-4 Override Warning | TBD | TBD | TBD |
| REQ-5 Manage Allergy Database | TBD | TBD | TBD |
| REQ-6 Dispense/Block | TBD | TBD | TBD |
| REQ-7 View Allergy History | TBD | TBD | TBD |
| REQ-8 Patient Self-Reported Allergies | TBD | TBD | TBD |
| REQ-9 Verify Allergy | TBD | TBD | TBD |
| REQ-10 Nurse Read-Only View | TBD | TBD | TBD |
| REQ-11 Manage User Accounts | TBD | TBD | TBD |
| REQ-12 Audit Log | TBD | TBD | TBD |
| REQ-13 Performance | TBD | TBD | TBD |
| REQ-14 Safety/Fail-Safe | TBD | TBD | TBD |
| REQ-15 Security | TBD | TBD | TBD |
| REQ-16 Traceability | TBD | TBD | TBD |

(This table will be fully populated when design.md and tasks.md are completed.)

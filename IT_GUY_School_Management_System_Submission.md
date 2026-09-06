# School Management System — Zoho CRM + Zoho Creator

> **Implementation status:** CRM schema (14 core modules — Academic Years through Payments) implemented and tested in a Zoho sandbox. Integration Events, Attendance Alerts, Deluge business logic, and the Creator parent portal are designed in full detail below but not yet built, due to time constraints on this assignment window.

## Technical implementation proposal

**Role:** Lead Zoho Solutions Architect  
**Platform:** Zoho CRM (administration and system of record) + Zoho Creator (authenticated parent portal)  
**Design principle:** CRM owns operational data. Creator contains only a minimal, parent-safe read projection. No parent can create, edit, or directly query unrestricted CRM student records.

> API names, connection names and Creator owner/application links below are implementation placeholders. They must be replaced with the tenant-generated values and tested in a Zoho sandbox before deployment.

---

## 1. Architecture and boundaries

```text
Admission webform → CRM Lead → controlled admission function
                                      ↓
                Contact + Student + Parent–Student Link + Enrolment
                                      ↓
          Attendance / Results / Invoices / Installments / Payments
                                      ↓
              CRM workflow outbox → Creator projection upsert
                                      ↓
       Creator Customer Portal → authenticated parent-only reports
```

### System-of-record decision

| Area | Authoritative system | Reason |
|---|---|---|
| Admissions, academics, attendance, exams, fees, payments | Zoho CRM | School staff administer, validate and audit these transactions there. |
| Parent identity and authorization | CRM Contact + Parent–Student Link | Supports siblings, multiple guardians and rapid revocation. |
| Parent portal display records | Zoho Creator | A small read-only projection improves portal performance and limits data exposure. |
| Parent changes to school records | Not permitted in this release | Prevents conflicting sources of truth. Change requests can be added later as a controlled workflow. |

Creator is deliberately **not** queried live against CRM on every parent page load. Live querying creates avoidable latency, expands the blast radius of a broad CRM connection, and consumes API capacity under concurrent portal use. A projection is synchronized from CRM and reconciled on a schedule.

---

## 2. CRM data architecture

### 2.1 Master and transactional modules

| Module | Core fields / relationships | Integrity rule |
|---|---|---|
| Academic Years | Name, Start Date, End Date, Status | One `Current` year at a time; dates cannot overlap. |
| Classes | Name, Display Order, Active | Master grade/class catalogue. |
| Sections | Class lookup, Name, Capacity, Active | Composite business key: Class + Section Name. |
| Subjects | Name, Code, Active | Subject Code is unique. |
| Teachers | Work Email, CRM User lookup (optional), Active | Work Email unique. |
| Class Subjects | Academic Year, Class, Subject, Teacher, Maximum Marks | Unique key: Year + Class + Subject. |
| Students | Student ID, legal/profile fields, Status, Current Enrolment lookup | Student ID is immutable and unique. No editable current class text. |
| Student Enrolments | Student, Academic Year, Class, Section, Roll Number, Status, dates | Unique key: Student + Academic Year. Preserves history. |
| Parent Student Links | Parent (Contact), Student, Relationship, Active | Unique key: Contact + Student. This is the authorization source. |
| Attendance Records | Student Enrolment, Student, Date, Status, correction fields | Unique key: Student + Date. |
| Examinations | Academic Year, Class, dates, Status | Only `Published` results are visible to parents. |
| Results | Enrolment, Student, Examination, Class Subject, marks, grade | Unique key: Student + Exam + Class Subject. |
| Fee Structures | Academic Year, Class, Total Fee, Active | Unique key: Year + Class. |
| Fee Invoices | Enrolment, Student, Fee Structure, total/collected/outstanding, status | One active invoice per enrolment per fee structure/version. |
| Fee Installments | Fee Invoice, installment no., due date, due/paid/balance/status | Unique key: Invoice + Installment Number. |
| Payments | Invoice, Installment, Student, date, amount, method, reference, status | Append-only financial ledger; reversals, not deletions. |
| Integration Events | Entity type, CRM record ID, operation, version/modified time, status, retries, error | Durable integration outbox and audit trail. |
| Attendance Alerts | Enrolment, threshold, current percentage, owner, status, last notified | Deduplicated operational case. |

### 2.2 Relationship rationale

`Student` is the long-lived person record. `Student Enrolment` is the year-specific academic placement. This distinction prevents an annual promotion from overwriting history and makes attendance, results and invoicing unambiguous.

```text
Contact (parent) ──< Parent Student Link >── Student ──< Student Enrolment
                                                        ├──< Attendance Record
                                                        ├──< Result >── Examination
                                                        └──< Fee Invoice ──< Fee Installment ──< Payment

Academic Year ──< Class Subject >── Subject
                     │
                    Class ──< Section
```

Use CRM lookups for record navigation and related lists. CRM does not provide a database-style cascading master-detail relationship with complete referential guarantees for every custom-module scenario, so the design adds controlled creation/edit functions, lookup filters, validation rules, and restricted delete permissions.

### 2.3 Required validation and protection

- Make all business keys unique and read-only to normal users. Construct them from immutable CRM IDs, not names.
- Enforce one active enrolment per student. A new enrolment may be activated only after the former enrolment is completed or withdrawn.
- Filter Section by selected Class; filter Class Subject by the examination’s Year and Class; validate the same relationships again in Deluge because UI filters are not security controls.
- Restrict deletion of enrolments, attendance, invoices, installments and payments. Use status changes/reversal records and mandatory correction reasons for auditable history.
- Use CRM roles/profiles: Admissions, Teachers, Accounts, Attendance Administrator and Management. Parents have no CRM access.
- Ensure `Current Enrolment` is changed only by an approved enrolment workflow/function.

---

## 3. Controlled business logic

### 3.1 Admissions

1. A CRM Lead webform captures student name, DOB, requested year/class, guardian information and consent.
2. An Admissions user confirms the Lead through a custom transition/button.
3. The function finds or creates the Contact using a normalized parent email/mobile policy, creates the Student, Parent–Student Link and one Student Enrolment inside a controlled flow.
4. Before creating any record, it checks existing admission/enrolment keys. If a partial prior run is found, it returns the existing record IDs for review instead of blindly duplicating records.

Native Lead conversion is not used as the primary logic because the assignment requires a custom Student plus a year-specific enrolment. The custom function keeps those records consistent and records the originating Lead ID.

### 3.2 Attendance

**Business rules**

- A record is allowed only for an active enrolment on the selected date.
- The date cannot be in the future, outside the enrolment date range, or in a non-current/closed academic period.
- Exactly one attendance record exists for each Student + Date, regardless of staff member.
- `Present` and `Late` count as present; `Excused` is excluded from the denominator; the convention is displayed in the portal.
- A correction updates the original record only for an Attendance Administrator and requires Correction Reason, Corrected By and Corrected On.

The controlled function checks the key for a friendly message. The unique `Attendance_Key` field is the final concurrency guard: simultaneous users may both pass a read check, but only one create succeeds. The duplicate-key API failure is caught and returned as “attendance already recorded.”

```deluge
// Pseudocode / CRM function: attendance.record(enrolmentId, attendanceDate, status, remarks)
enrolment = zoho.crm.getRecordById("Student_Enrolments", enrolmentId.toLong());
if(enrolment.isEmpty() || enrolment.get("Status") != "Active") return "No active enrolment.";
if(attendanceDate > zoho.currentdate) return "Future attendance is not allowed.";
if(!List("Present","Absent","Late","Excused").contains(status)) return "Invalid status.";

studentId = enrolment.get("Student").get("id");
key = studentId + "-" + attendanceDate.toString("yyyy-MM-dd");
existing = zoho.crm.searchRecords("Attendance_Records", "(Attendance_Key:equals:" + key + ")");
if(!existing.isEmpty()) return "Attendance already recorded.";

payload = {"Student_Enrolment":enrolmentId,"Student":studentId,
 "Attendance_Date":attendanceDate,"Status":status,"Remarks":remarks,"Attendance_Key":key};
response = zoho.crm.createRecord("Attendance_Records", payload);
// Interpret duplicate-value response as a conflict, and log unexpected errors.
return response;
```

Attendance percentage is maintained on `Student Enrolment`, not Student, because it is academic-year specific. A post-save function recalculates only the affected enrolment; it retrieves attendance in pages when necessary and updates the one summary record. A nightly reconciliation processes only enrolments updated since the last successful checkpoint. It never loops all students and runs searches inside that loop.

### 3.3 Results

- Examination must belong to the same year and class as the enrolment and class subject.
- `Maximum Marks > 0`; `0 ≤ Marks Obtained ≤ Maximum Marks`.
- Grade is derived from a documented grade scale, not freely entered.
- Results may be edited by authorized staff while Draft; parents receive only records from a Published examination.
- The unique result key prevents duplicate entries for the same student/exam/subject.

### 3.4 Fees and payments

At enrolment approval, the system selects one active Fee Structure for that year/class and creates a Fee Invoice and its planned installments. The plan is validated before records are created:

- Each installment has a positive due amount and due date.
- Installment amounts must equal the invoice total after currency rounding. Any rounding remainder is assigned deterministically to the final installment and logged.
- A payment has a positive amount, references one invoice and one installment, and cannot exceed that installment’s current balance.
- Payment references are unique when provided. Duplicate gateway callbacks use an idempotency/reference key.
- Payments are never deleted. A reversal creates/marks a reversing transaction, requires an approval reason, and recalculates balances from non-reversed received payments.

```deluge
// Pseudocode / CRM function: fees.recalculateInvoice(invoiceId)
invoice = zoho.crm.getRecordById("Fee_Invoices", invoiceId.toLong());
// Fetch all related Payments in pages; retain Received records only.
payments = getAllRelatedPages("Payments", "Fee_Invoices", invoiceId);
collected = 0.0;
for each payment in payments
{
  if(payment.get("Payment_Status") == "Received") collected = collected + ifnull(payment.get("Amount"),0).toDecimal();
}
total = ifnull(invoice.get("Total_Fees"),0).toDecimal();
outstanding = total - collected;
if(outstanding < 0) return "Integrity error: collected exceeds invoice total.";
status = if(outstanding == 0) ? "Paid" : (collected > 0 ? "Partially Paid" : "Not Due");
zoho.crm.updateRecord("Fee_Invoices", invoiceId.toLong(), {"Amount_Collected":collected,
 "Outstanding_Amount":outstanding,"Payment_Status":status});
```

A scheduled job updates installment and invoice status to `Overdue` only when the due date has passed and a balance remains. The job performs filtered, paginated queries and uses checkpointing; it does not scan or update every invoice each run.

---

## 4. CRM–Creator integration design

### 4.1 Parent-safe projection

Creator forms are all internal/synchronized forms; parent reports are read-only.

| Creator form | Key | Data allowed |
|---|---|---|
| Parent_Profile | CRM Contact ID / Portal email | Authenticated parent mapping only. |
| Parent_Student_Access | Parent Profile + CRM Student ID | Active authorization relationship. |
| Student_Snapshot | CRM Student ID | Parent-safe profile and current academic placement. |
| Attendance_Snapshot | CRM Attendance ID | Date and status only. |
| Result_Snapshot | CRM Result ID | Published examination, subject, marks, grade only. |
| Fee_Snapshot | CRM Invoice ID | Total, collected, outstanding, due/status. |
| Payment_Snapshot | CRM Payment ID | Parent-safe payment receipt/history. |

Internal notes, staff contact data, correction notes, lead information, risk alert notes and unrelated children are never copied to Creator.

### 4.2 Reliable synchronization

CRM workflows fire after committed creates/updates on Student, Enrolment, Parent–Student Link, Attendance, Result, Invoice and Payment. Instead of making the workflow itself responsible for an unrecoverable direct call, it creates or updates an `Integration Event` outbox record with entity ID, operation and modified timestamp.

A scheduled worker processes pending events in bounded batches:

1. Read the current CRM record and construct an allow-listed Creator payload.
2. Upsert Creator by immutable CRM ID—not by parent email or a display name.
3. On success, mark the outbox event complete with the processed timestamp.
4. On transient failure, retry with bounded exponential backoff; store the error and alert an administrator after the retry threshold.
5. When events for the same record arrive out of order, skip an older event if its source modified time is earlier than the projection’s last source modified time.
6. A nightly reconciliation compares records changed since the last checkpoint and repairs missed projections.

This approach is idempotent: running the same event more than once has the same result as running it once. It also avoids API calls in per-record nested loops and respects page/batch limits.

---

## 5. Creator parent-portal security

### 5.1 Authentication and authorization

- Use Creator Customer Portal with invited parent users only; disable public sign-up.
- Provision portal access from the active CRM Contact/Parent–Student Link process. Normalize email before invitation and provide an operational process for changed emails.
- Map the authenticated Creator user to exactly one `Parent_Profile` record.
- Use `Parent_Student_Access` as the only authorization source. A parent may have many active child rows; a child may have many authorized guardians.
- On Parent–Student Link deactivation, process the access event with high priority and immediately deactivate the related Creator access row. Remove portal access when the parent has no remaining active child link, subject to school policy.

### 5.2 Record-level criteria

Every parent-facing report/page applies this server-side authorization test, including exports, search, drill-down pages and custom actions:

```deluge
parent = Parent_Profile[Portal_User_Email == zoho.loginuserid];
if(parent.count() != 1) return "Unauthorized";
allowedStudentIds = Parent_Student_Access[
 Parent_Profile == parent.ID && Active == true].CRM_Student_ID.getAll();
if(!allowedStudentIds.contains(input.Selected_Student_CRM_ID)) return "Unauthorized";
// Query the relevant snapshot where CRM_Student_ID is in allowedStudentIds.
```

The child selector is populated from allowed access rows only. A hidden field, client-side filter, URL parameter or a submitted Student ID is never trusted as authorization. Creator form permissions deny parent add/edit/delete rights; page/report criteria enforce the same parent–student relationship independently.

---

## 6. Additional feature: attendance early-warning case management

### Business problem

Schools often discover a student’s attendance risk only at term end or after a parent raises a concern. A generic email list does not establish ownership, prevent duplicate notifications, or show whether intervention occurred.

### Solution

A nightly job evaluates only active enrolments changed since the last run. Once an enrolment has at least 10 attendance days and falls below 75%, it creates or updates one `Attendance Alert` assigned to the class teacher/attendance administrator.

- Alerts are keyed by enrolment + threshold period, preventing duplicates.
- The first threshold crossing notifies the owner.
- Further notices are suppressed until the percentage changes by at least 2 percentage points, seven days pass, or the case is escalated.
- The owner records contact/action and resolves the alert only when conditions improve or an approved exception is recorded.
- Management sees open alerts, ageing and resolution rates on a CRM dashboard.
- The portal displays factual attendance only; it does not expose confidential staff intervention notes.

This is a scalable operational workflow, rather than simply a scheduled email, because it has ownership, deduplication, measurable follow-through and auditability.

---

## 7. Performance, limits and observability

- Batch and paginate CRM and Creator reads; never assume `searchRecords` returns an entire dataset.
- Avoid queries or integration calls inside a student/payment loop. Fetch related records in pages, aggregate in memory, then issue one update per parent entity.
- Use scheduled jobs with saved checkpoint timestamps and maximum batch sizes. Record the checkpoint only after successful processing.
- Use OAuth connections with minimum scope: CRM→Creator can modify only projection forms; Creator→CRM is unnecessary for normal parent reads.
- Log every integration event, retry count, response/error and last successful source version. Add a management report for failed events and oldest pending event.
- Treat CRM API and Creator task limits as capacity constraints. Monitor daily calls, response latency, queue depth and retry failures before increasing batch size.
- Test race conditions explicitly: two attendance submissions, duplicate payment callback, simultaneous parent-link changes, and CRM updates arriving in a different order than they were created.

---

## 8. End-to-end acceptance test script

| Test | Expected evidence |
|---|---|
| Admission | Webform Lead is confirmed; exactly one Contact, Student, Parent–Student Link and active Enrolment are created. Re-running the action produces no duplicate records. |
| Academic history | Complete an old enrolment and activate a new one; Student shows the current enrolment while both year records remain visible. |
| Attendance duplicate | Mark a student once; a second concurrent/same-date request is rejected by the unique key. |
| Attendance correction | Unauthorized user cannot edit. Authorized correction requires reason and recalculates the correct enrolment percentage. |
| Result integrity | Invalid marks, unmatched class/year and duplicate exam-subject result are rejected. Only published results reach the parent portal. |
| Fee integrity | Plan total matches invoice; overpayment is rejected; partial payment recalculates balance; reversal restores the correct balance without deleting history. |
| Parent isolation | Parent A sees only linked child/children; manually supplied sibling/unrelated CRM ID returns no report data; Parent B cannot access Parent A’s URL/report data. |
| Sync recovery | Force one Creator sync failure; event is retried, logged and later reconciled without duplicate Creator records. |
| Alert workflow | Attendance crosses threshold; one owned alert is created; repeated runs do not send duplicate alerts; escalation/resolution is auditable. |

---

## 9. Interview defense questions and concise answers

1. **Why is Student Enrolment separate from Student?**  
   A Student is a person; an enrolment is their time-bound class/year placement. Separating them preserves history and prevents attendance, results and fees from being linked ambiguously after promotion.

2. **How do you prevent a race condition in daily attendance?**  
   The function checks for usability, but an immutable Student+Date unique key is the final database-level guard. Duplicate-key errors are handled gracefully, so two simultaneous requests cannot create two records.

3. **Why not authorize parents using their email on Student?**  
   Email is an attribute, not an authorization relationship. An explicit Parent–Student junction supports multiple guardians/siblings, revocation and changed contact details, and is evaluated against the authenticated portal user.

4. **How is the CRM–Creator sync recoverable?**  
   A durable outbox event is processed idempotently with retries, source-version ordering, error logging and a reconciliation job. A failed workflow call cannot silently create permanent data drift.

5. **How would this scale to 10,000 students?**  
   Use projections for the portal, paginated filtered reads, checkpoints, bounded batches and aggregate functions. Avoid search calls inside loops, use one update per affected parent record, and monitor API/queue metrics.

---

## 10. Submission summary

Zoho CRM is the authoritative school administration platform. The model separates person, annual enrolment, academic transactions and finance transactions, with immutable keys and controlled workflows to preserve integrity. Zoho Creator is an invited, read-only parent portal powered by a minimal CRM-synchronized projection. Parent access is enforced through explicit active Parent–Student Link records and authenticated portal criteria, not email matching or client-side filters. Integration is idempotent, queued, monitored and reconciled. The additional attendance-warning feature creates owned and deduplicated intervention cases, demonstrating measurable business optimization rather than a generic notification.

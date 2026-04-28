# Smart Campus Hub — Full System Context Document

> **Purpose of this file:** This document is a comprehensive reference for any LLM, developer, or team member picking up this project. It consolidates every design decision, class definition, relationship, requirement, diagram specification, and scope boundary in one place. Read this before working on any diagram, code, or documentation for the Smart Campus Hub.

---

## Table of Contents

1. [Project Identity](#1-project-identity)
2. [System Overview](#2-system-overview)
3. [Problem Statement](#3-problem-statement)
4. [Scope — In and Out](#4-scope--in-and-out)
5. [Objectives](#5-objectives)
6. [Actors and Stakeholders](#6-actors-and-stakeholders)
7. [Functional Requirements](#7-functional-requirements)
8. [Non-Functional Requirements](#8-non-functional-requirements)
9. [Software Process Model](#9-software-process-model)
10. [Domain Classes — Full Detail](#10-domain-classes--full-detail)
11. [Class Relationships](#11-class-relationships)
12. [BCE Model](#12-bce-model)
13. [Use Cases — Full Detail](#13-use-cases--full-detail)
14. [Diagrams Inventory](#14-diagrams-inventory)
15. [Database Schema Notes](#15-database-schema-notes)
16. [Key Constraints and Business Rules](#16-key-constraints-and-business-rules)
17. [Diagram Division Among Team Members](#17-diagram-division-among-team-members)
18. [Tooling Decisions](#18-tooling-decisions)
19. [Decisions and Rationale Log](#19-decisions-and-rationale-log)

---

## 1. Project Identity

| Field | Value |
|---|---|
| **Project Name** | Smart Campus Hub |
| **Type** | Web-based Academic Management Platform |
| **Architecture** | MVC (Model-View-Controller) |
| **Backend** | MySQL (normalised relational schema) |
| **Process Model** | Unified Process (UP) — iterative, use-case-driven |
| **Version** | 1.0 |
| **Platform** | Browser-based only (no mobile native app) |
| **Report Chapter** | OOAD (Object-Oriented Analysis and Design) |

---

## 2. System Overview

The Smart Campus Hub is a unified web platform that consolidates fragmented university operations — student records, course registrations, attendance tracking, grading, faculty interactions, and administrative configuration — into a single authenticated interface.

It replaces multiple standalone portals (each with separate credentials, interfaces, and update cycles) with one consistent system that delivers:
- Real-time data updates immediately after faculty action
- Role-based access control (students see only their own records; faculty act only on assigned sections)
- Full audit trails for every grade and attendance change
- Automated system processes (registration locking, GPA computation, backups, alerts)

The system is built on a **normalised MySQL backend** (10 core tables) and follows the **MVC pattern** where Views handle UI, Controllers handle business logic, and Models/Entities map directly to database tables.

---

## 3. Problem Statement

The Hub directly addresses five concrete university problems:

| # | Problem | How the Hub Solves It |
|---|---|---|
| 1 | **Fragmented portals** — separate systems for registration, transcripts, attendance, announcements with different credentials | Consolidates all workflows behind one authenticated entry point |
| 2 | **Data redundancy and sync errors** — duplicated data across departmental DBs, updates don't propagate | Single normalised source of truth eliminates inconsistent records |
| 3 | **Registration bottlenecks** — no central capacity check, scheduling clashes, server overloads at peak load | Transaction-controlled enrollment with section capacity limits enforced at DB layer |
| 4 | **Opaque communication** — grades and attendance reach students through informal channels with delays | Authenticated views updated immediately after faculty action, with audit trails |
| 5 | **Lack of central oversight** — admins can't see system-wide activity | Centralised reporting on enrollments, attendance trends, grade distributions |

---

## 4. Scope — In and Out

### In Scope (Version 1.0)
- User authentication and role-based access control
- Student academic profiles
- Faculty profiles
- Course catalogue and per-semester course offerings (sections)
- Enrollment with capacity enforcement and prerequisite control
- Attendance recording and reporting
- Grade submission, GPA computation, and transcripts
- Administrative configuration (users, semesters, section assignments, classroom allocation)
- Institutional reporting (minimum 5 standard reports)
- Full audit logging of all grade and attendance changes (via DB triggers)

### Out of Scope (Version 1.0) — explicitly excluded
- **Learning content delivery** — lecture videos, online quizzes → handled by a separate LMS
- **Finance module** — fee vouchers, payment verification, scholarships, financial holds → reserved for v2.0
- **Financial banking integration and payment gateways**
- **Alumni management**
- **Library systems**
- **Hostel management**
- **Transport tracking**
- **Mobile native application** (browser-based only)
- **Submit Assignments** — while listed in FR-05, this is LMS territory and should be greyed out or noted as placeholder in diagrams
- **View Exam Seating Plans** — listed in FR-06 but no backing table exists in the schema; treat as placeholder

> **Important for diagrams:** Finance Officer actor and their use cases must be shown in diagrams but rendered in grey/dashed style to indicate out-of-scope status. Do NOT remove them — their presence signals future integration points.

---

## 5. Objectives

| ID | Objective |
|---|---|
| OBJ-01 | Deliver a unified web platform replacing at least 3 legacy portals within one academic year of deployment |
| OBJ-02 | Enforce zero duplicate enrollments per student-section pair via DB-level UNIQUE constraints |
| OBJ-03 | Achieve sub-2-second response times on indexed queries (enrollment lookup, attendance retrieval, transcript view) at peak load |
| OBJ-04 | Reduce registration-period support tickets by at least 40% vs. legacy system |
| OBJ-05 | Maintain full auditability of every grade and attendance change with zero manual logging effort |
| OBJ-06 | Compute end-of-semester GPAs for entire student body within 30 minutes of grade submission cutoff |
| OBJ-07 | Provide administrators with at least 5 standard institutional reports without external tooling |
| OBJ-08 | Maintain 99.5% monthly uptime during active academic periods, supported by daily automated backups |

---

## 6. Actors and Stakeholders

There are **6 actors** in the system. All human actors interact through role-restricted browser interfaces.

### Actor 1: Student
- **Role:** Primary end user
- **Capabilities:** Register for course sections, drop sections, view attendance and grades, view transcript and CGPA, download fee vouchers (placeholder), apply for final clearance
- **Access level:** Read access to own records only; cannot see other students' data
- **Key constraint:** Cannot enroll if section is at capacity, prerequisite not met, or financial hold exists

### Actor 2: Faculty
- **Role:** Academic staff
- **Capabilities:** Record attendance for assigned sections, upload/modify grades, manage course materials, view timetable, create assessments, review anonymous feedback
- **Access level:** Acts only on sections they are assigned to; cannot modify another faculty member's section data
- **Key constraint:** Cannot record attendance for a future date (DB trigger rejects it)

### Actor 3: Academic Advisor / Head of Department (HOD)
- **Role:** Academic oversight
- **Capabilities:** Approve/reject student registration requests, monitor academic progress (CGPA, credits, pending requirements), grant prerequisite overrides, resolve timetable conflicts
- **Access level:** Department-scoped; can only act on students/sections within their department

### Actor 4: Finance Officer
- **Role:** Out of scope for v1.0
- **Status in diagrams:** Present but greyed out / dashed
- **Future role:** Will handle fee vouchers, payment verification, scholarships, financial holds in v2.0

### Actor 5: Administrator
- **Role:** System-wide management; highest privilege level
- **Capabilities:** Create/deactivate/manage all user accounts with role assignment, configure academic semesters (dates and deadlines), assign faculty to sections, allocate classrooms/labs, generate institutional reports
- **Access level:** Full system access; no data restrictions

### Actor 6: System / Timer (Automated / Non-human)
- **Role:** Scheduled background processes
- **Capabilities:** Auto-lock course registration at configured deadline, trigger low-attendance email alerts when threshold crossed, compute end-of-semester CGPAs in batch, execute daily database backups
- **Note:** This is a non-human actor; it does not log in. It appears in use case diagrams as an external system actor.

---

## 7. Functional Requirements

| ID | Actor | Requirement |
|---|---|---|
| FR-01 | Student | Allow student to register for available course sections within active registration window |
| FR-02 | Student | Display each student's transcript and current CGPA computed from grades |
| FR-03 | Student | Display daily attendance per enrolled section with status (present, absent, late) |
| FR-04 | Student | Allow students to download semester fee vouchers as PDF (placeholder — future Finance module) |
| FR-05 | Student | Allow students to submit assignments to assigned sections before deadline (placeholder — LMS scope) |
| FR-06 | Student | Display exam seating plans for active semester (placeholder — no backing table in v1.0) |
| FR-07 | Student | Allow students to apply for final clearance and degree |
| FR-08 | Student | Allow student to drop an enrolled section before drop deadline; update enrollment status to 'dropped' |
| FR-09 | Faculty | Allow faculty to record attendance for assigned sections on per-class-date basis; reject future dates |
| FR-10 | Faculty | Allow faculty to upload and modify grades for students in assigned sections; all changes captured in audit log |
| FR-11 | Faculty | Allow faculty to upload and manage course materials per section |
| FR-12 | Faculty | Display faculty member's teaching timetable for active semester |
| FR-13 | Faculty | Allow faculty to create assignments and quizzes |
| FR-14 | Faculty | Allow faculty to view anonymous course feedback aggregated by section |
| FR-15 | Advisor | Allow advisors to approve or reject student registration requests |
| FR-16 | Advisor | Allow advisors to grant prerequisite overrides on per-student-per-section basis |
| FR-17 | Advisor | Display each advisee's academic progress including CGPA, completed credit hours, pending requirements |
| FR-18 | Admin | Allow administrators to create, deactivate, and manage all user accounts with role assignment |
| FR-19 | Admin | Allow administrators to configure academic semesters with start dates, end dates, registration deadlines |
| FR-20 | Admin | Allow administrators to assign faculty to course sections |
| FR-21 | Admin | Allow administrators to allocate classrooms and laboratories to sections |
| FR-22 | Admin | Generate institutional reports on enrollments, grade distribution, and attendance trends |
| FR-23 | System | Automatically lock course registration at configured deadline |
| FR-24 | System | Trigger low-attendance email alerts when student's attendance in any section drops below configured threshold |
| FR-25 | System | Compute end-of-semester CGPAs in batch following grade submission cutoff |
| FR-26 | System | Execute daily database backup at configured off-peak time |
| FR-27 | All Users | Provide secure password reset workflow via verified email link |

---

## 8. Non-Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-01 | Performance | Indexed queries (enrollment, attendance, audit lookups) respond within 2 seconds at peak load |
| NFR-02 | Security | All passwords stored as bcrypt hashes; raw passwords never logged or returned |
| NFR-03 | Data Integrity | CHECK and FOREIGN KEY constraints reject all out-of-range data at insertion (end_date > start_date; marks_obtained ≤ total_marks; grade_points 0.00–4.00) |
| NFR-04 | Consistency | ENUM types on role, status, action columns guarantee closed value sets |
| NFR-05 | Concurrency | Enrollment is transactional; concurrent registration into a section at capacity cannot exceed max_capacity under any race condition |
| NFR-06 | Auditability | Every INSERT, UPDATE, DELETE on grades and attendance captured automatically in audit_log via DB triggers — zero application-side responsibility |
| NFR-07 | Availability | 99.5% monthly uptime during active academic periods; daily automated backups |
| NFR-08 | Scalability | Schema and indexing support at least 5× increase in students and enrollments without redesign |
| NFR-09 | Usability | All user-facing functions accessible through standard web browser with no plugin installation |
| NFR-10 | Maintainability | MVC structure isolates presentation, business logic, and persistence so changes in one layer don't require changes in others |

---

## 9. Software Process Model

**Adopted:** Unified Process (UP) — iterative, use-case-driven, architecture-centric, risk-focused.

**Justification:** The Hub is fundamentally object-oriented. Actors map to use cases → use cases drive class structure → class structure drives the schema. UP is built around exactly this flow. Its iterative nature suits academic projects where requirements evolve through review cycles. Four-phase structure gives natural milestones.

### Phases

| Phase | Key Activities |
|---|---|
| **Inception** | Define vision, identify primary actors and use cases, establish scope boundaries |
| **Elaboration** | Build architectural baseline; produce all UML diagrams; finalise DB schema; resolve major risks (capacity race conditions, audit integrity) |
| **Construction** | Implement iteratively module by module — auth → academic → attendance → grading → audit → admin — with continuous testing |
| **Transition** | UAT, deployment, training, handover, final documentation |

---

## 10. Domain Classes — Full Detail

The system has **10 domain classes**, each mapping one-to-one to a database table.

---

### Class 1: User
**Maps to:** `users` table  
**Role:** Base class; parent of Student and Faculty via inheritance  
**Color in diagrams:** Grey / neutral

| Attribute | Type | Constraints |
|---|---|---|
| user_id | INT | PK, AUTO_INCREMENT |
| username | VARCHAR(50) | UNIQUE, NOT NULL |
| password | VARCHAR(255) | bcrypt hash; never stored plain |
| role | ENUM | ('student', 'faculty', 'advisor', 'admin') |
| is_active | BOOLEAN | DEFAULT true |
| created_at | TIMESTAMP | DEFAULT NOW() |

| Method | Signature | Description |
|---|---|---|
| login() | login() : Boolean | Validates credentials, creates session |
| logout() | logout() : void | Destroys session |
| changePassword() | changePassword(newPwd) : void | Hashes and updates password |
| validateRole() | validateRole(requiredRole) : Boolean | Checks current user has required role |

---

### Class 2: Student *(extends User)*
**Maps to:** `students` table  
**Inheritance:** Generalisation from User (user_id is FK)  
**Color in diagrams:** Blue

| Attribute | Type | Constraints |
|---|---|---|
| student_id | INT | PK, AUTO_INCREMENT |
| user_id | INT | FK → users.user_id |
| first_name | VARCHAR(50) | NOT NULL |
| last_name | VARCHAR(50) | NOT NULL |
| email | VARCHAR(100) | UNIQUE |
| dob | DATE | — |
| program | VARCHAR(100) | — |
| batch_year | YEAR | — |

| Method | Signature | Description |
|---|---|---|
| registerForSection() | registerForSection(sectionId) : Enrollment | Creates enrollment after capacity/prerequisite checks |
| dropSection() | dropSection(enrollmentId) : void | Sets enrollment status to 'dropped' |
| viewTranscript() | viewTranscript() : List\<Grade\> | Returns all grades with computed GPA |
| viewAttendance() | viewAttendance(sectionId) : List\<Attendance\> | Returns attendance records for a section |

---

### Class 3: Faculty *(extends User)*
**Maps to:** `faculty` table  
**Inheritance:** Generalisation from User (user_id is FK)  
**Color in diagrams:** Green

| Attribute | Type | Constraints |
|---|---|---|
| faculty_id | INT | PK, AUTO_INCREMENT |
| user_id | INT | FK → users.user_id |
| first_name | VARCHAR(50) | NOT NULL |
| last_name | VARCHAR(50) | NOT NULL |
| email | VARCHAR(100) | UNIQUE |
| department | VARCHAR(100) | — |
| designation | VARCHAR(100) | — |

| Method | Signature | Description |
|---|---|---|
| recordAttendance() | recordAttendance(sectionId, date, statuses) : void | Marks all students for a class date; rejects future dates |
| uploadGrade() | uploadGrade(enrollmentId, marks) : Grade | Inserts/updates grade; triggers audit log |
| viewTimetable() | viewTimetable(semesterId) : List\<CourseSection\> | Returns assigned sections for semester |
| uploadMaterial() | uploadMaterial(sectionId, file) : void | Attaches material to section |

---

### Class 4: Course
**Maps to:** `courses` table  
**Role:** Master catalogue entry; independent of semester  
**Color in diagrams:** Yellow

| Attribute | Type | Constraints |
|---|---|---|
| course_id | INT | PK, AUTO_INCREMENT |
| course_code | VARCHAR(20) | UNIQUE (e.g. CS-301) |
| course_name | VARCHAR(150) | NOT NULL |
| credit_hours | TINYINT | — |

| Method | Signature | Description |
|---|---|---|
| addCourse() | addCourse(code, name, credits) : Course | Creates new course catalogue entry |
| updateCourse() | updateCourse(courseId, data) : void | Updates course metadata |
| listSections() | listSections(semesterId) : List\<CourseSection\> | Returns all sections of this course in a semester |

---

### Class 5: Semester
**Maps to:** `semesters` table  
**Role:** Defines the active academic period; controls registration windows  
**Color in diagrams:** Yellow

| Attribute | Type | Constraints |
|---|---|---|
| semester_id | INT | PK, AUTO_INCREMENT |
| name | VARCHAR(50) | e.g. "Fall 2025" |
| start_date | DATE | — |
| end_date | DATE | CHECK: end_date > start_date |
| is_active | BOOLEAN | DEFAULT false; only one active at a time |

| Method | Signature | Description |
|---|---|---|
| activate() | activate() : void | Sets is_active = true; deactivates others |
| deactivate() | deactivate() : void | Closes the semester |
| setRegistrationDeadline() | setRegistrationDeadline(date) : void | Configures the auto-lock deadline |

---

### Class 6: CourseSection
**Maps to:** `course_sections` table  
**Role:** A specific offering of a course in a semester, taught by a faculty member  
**Color in diagrams:** Purple  
**Key concept:** The junction between Course, Semester, and Faculty

| Attribute | Type | Constraints |
|---|---|---|
| section_id | INT | PK, AUTO_INCREMENT |
| course_id | INT | FK → courses.course_id |
| semester_id | INT | FK → semesters.semester_id |
| faculty_id | INT | FK → faculty.faculty_id |
| section_code | VARCHAR(20) | e.g. "CS301-A" |
| max_capacity | SMALLINT | Enforced at DB layer via transaction |

| Method | Signature | Description |
|---|---|---|
| checkCapacity() | checkCapacity() : Boolean | Returns true if current enrollment < max_capacity |
| assignFaculty() | assignFaculty(facultyId) : void | Assigns/reassigns faculty to section |
| getRoster() | getRoster() : List\<Student\> | Returns all actively enrolled students |

---

### Class 7: Enrollment
**Maps to:** `enrollments` table  
**Role:** Junction/association class between Student and CourseSection  
**Color in diagrams:** Red/Pink  
**Critical constraint:** UNIQUE(student_id, section_id) — prevents duplicate enrollment  
**Composition owner:** Enrollment owns Attendance and Grade (cascade delete)

| Attribute | Type | Constraints |
|---|---|---|
| enrollment_id | INT | PK, AUTO_INCREMENT |
| student_id | INT | FK → students.student_id |
| section_id | INT | FK → course_sections.section_id |
| enrolled_at | TIMESTAMP | DEFAULT NOW() |
| status | ENUM | ('active', 'dropped', 'completed') |

**Additional constraint:** UNIQUE(student_id, section_id) enforced at DB layer

| Method | Signature | Description |
|---|---|---|
| enroll() | enroll() : void | Creates enrollment record inside a transaction |
| drop() | drop() : void | Sets status = 'dropped' before drop deadline |
| markCompleted() | markCompleted() : void | Sets status = 'completed' at semester end |

---

### Class 8: Attendance
**Maps to:** `attendance` table  
**Role:** Records whether a student was present/absent/late for a specific class date  
**Color in diagrams:** Blue (same family as Student — attendance is student-centric)  
**Owned by:** Enrollment (composition — deleting enrollment deletes attendance)

| Attribute | Type | Constraints |
|---|---|---|
| attendance_id | INT | PK, AUTO_INCREMENT |
| enrollment_id | INT | FK → enrollments.enrollment_id |
| class_date | DATE | DB trigger rejects future dates |
| status | ENUM | ('present', 'absent', 'late') |
| marked_by | INT | FK → faculty.faculty_id |

**Additional constraint:** UNIQUE(enrollment_id, class_date) — no duplicate attendance per class date

| Method | Signature | Description |
|---|---|---|
| mark() | mark(status) : void | Records initial attendance |
| modify() | modify(status) : void | Updates attendance; triggers audit log |
| getMonthlyReport() | getMonthlyReport(enrollmentId) : Map | Aggregates attendance stats for a month |

---

### Class 9: Grade
**Maps to:** `grades` table  
**Role:** Stores the grade outcome for an enrollment  
**Color in diagrams:** Green (same family as Faculty — grades are faculty-submitted)  
**Owned by:** Enrollment (composition — deleting enrollment deletes grade)  
**Multiplicity:** 0..1 per enrollment (a grade may not exist yet)

| Attribute | Type | Constraints |
|---|---|---|
| grade_id | INT | PK, AUTO_INCREMENT |
| enrollment_id | INT | FK → enrollments.enrollment_id |
| marks_obtained | DECIMAL(5,2) | CHECK: marks_obtained ≤ total_marks |
| total_marks | DECIMAL(5,2) | — |
| letter_grade | VARCHAR(3) | Computed from marks ratio (e.g. A, B+, C) |
| grade_points | DECIMAL(3,2) | CHECK: 0.00–4.00 |

| Method | Signature | Description |
|---|---|---|
| compute() | compute(marks, total) : String | Calculates letter_grade and grade_points from marks |
| update() | update(newMarks) : void | Updates marks; change captured by audit trigger |
| getGradePoints() | getGradePoints() : Decimal | Returns grade_points for GPA computation |

---

### Class 10: AuditLog
**Maps to:** `audit_log` table  
**Role:** Immutable record of every change to grades and attendance  
**Color in diagrams:** Grey/neutral  
**Population method:** Written exclusively by **database triggers** — not by application code  
**Important:** logChange() is never called from application layer; it is the trigger function

| Attribute | Type | Constraints |
|---|---|---|
| log_id | INT | PK, AUTO_INCREMENT |
| table_name | VARCHAR(50) | Which table was changed (e.g. 'grades') |
| record_id | INT | PK of the changed row |
| action | ENUM | ('INSERT', 'UPDATE', 'DELETE') |
| old_value | JSON | Previous state of the row |
| new_value | JSON | New state of the row |
| changed_by | INT | FK → users.user_id |
| changed_at | TIMESTAMP | DEFAULT NOW() |

| Method | Signature | Description |
|---|---|---|
| logChange() | logChange() : void | **DB trigger only** — inserts audit row automatically |

---

## 11. Class Relationships

### Inheritance (Generalisation)
| Relationship | Type | Description |
|---|---|---|
| Student → User | Generalisation | Student extends User; shares user_id FK |
| Faculty → User | Generalisation | Faculty extends User; shares user_id FK |

### Associations
| From | To | Multiplicity | Label | Notes |
|---|---|---|---|---|
| Faculty | CourseSection | 1 → * | teaches | One faculty teaches many sections |
| Course | CourseSection | 1 → * | offers | One course offered in many sections/semesters |
| Semester | CourseSection | 1 → * | hosts | One semester hosts many sections |
| User | AuditLog | 1 → * | changed_by | Every audit entry references the user who made the change |

### Aggregation (open diamond ◇)
| Whole | Part | Multiplicity | Description |
|---|---|---|---|
| CourseSection | Student (via Enrollment) | 1 ◇— * | Section has a roster of students; students exist independently of sections |

### Composition (filled diamond ◆)
| Whole | Part | Multiplicity | Description |
|---|---|---|---|
| Enrollment | Attendance | 1 ◆— * | Cascade delete: removing enrollment removes all attendance records |
| Enrollment | Grade | 1 ◆— 0..1 | Cascade delete: removing enrollment removes its grade |

### Dependency / Trigger
| From | To | Type | Description |
|---|---|---|---|
| AuditLog | Grade | «triggers» | DB trigger fires on Grade INSERT/UPDATE/DELETE → writes to AuditLog |
| AuditLog | Attendance | «triggers» | DB trigger fires on Attendance INSERT/UPDATE/DELETE → writes to AuditLog |

---

## 12. BCE Model

The BCE (Boundary–Control–Entity) pattern maps directly onto the MVC layers:

| Boundary (UI / View) | Control (Logic / Controller) | Entity (Data / Model) |
|---|---|---|
| LoginForm | AuthController | User |
| CourseRegistrationForm | EnrollmentController | Student, CourseSection, Enrollment |
| AttendanceForm | AttendanceController | Enrollment, Attendance |
| GradeEntryForm | GradeController | Enrollment, Grade |
| AdminDashboard | AdminController, AuditController | User, Semester, CourseSection, AuditLog |

---

## 13. Use Cases — Full Detail

### Use Case 1: Register for Courses
| Field | Detail |
|---|---|
| Actor | Student |
| Precondition | Student is authenticated; registration window is open; no active financial hold |
| Main Flow | 1. Student logs in → 2. Browses available sections → 3. System checks section capacity and prerequisites → 4. Student confirms enrollment → 5. System creates enrollments row inside a transaction → 6. Returns confirmation |
| Postcondition | New row in enrollments with status='active'; section capacity counter incremented |
| Exceptions | Capacity full → rejected; prerequisite missing → override required; duplicate enrollment → rejected by UNIQUE constraint |
| Includes | Login / Authenticate |
| Extended by | Grant Prerequisite Override (when prerequisite check fails) |

### Use Case 2: Record Attendance
| Field | Detail |
|---|---|
| Actor | Faculty |
| Precondition | Faculty is authenticated and assigned to selected section |
| Main Flow | 1. Faculty selects section and class date → 2. System loads enrolled student list → 3. Faculty marks each student present/absent/late → 4. System inserts attendance rows → 5. Confirmation displayed |
| Postcondition | Attendance rows exist for all students for chosen date; UNIQUE(enrollment_id, class_date) prevents duplicates |
| Exceptions | Future date → trigger rejects insert; duplicate → rejected; faculty not assigned → access denied |
| Extended by | Trigger Low-Attendance Alert (when threshold crossed) |

### Use Case 3: Upload / Modify Grades
| Field | Detail |
|---|---|
| Actor | Faculty |
| Precondition | Faculty is authenticated and assigned to section; grade entry window is open |
| Main Flow | 1. Faculty selects section → 2. System loads enrolled roster → 3. Faculty enters marks_obtained and total_marks → 4. System computes letter_grade and grade_points → 5. INSERT or UPDATE on grades → 6. Trigger writes audit_log row → 7. Notification sent to student |
| Postcondition | grades row reflects new marks; corresponding audit_log row exists |
| Exceptions | Marks exceed total_marks → CHECK constraint rejects; faculty not assigned → access denied |
| Includes | Select Course Section |

### Use Case 4: Manage System User Accounts
| Field | Detail |
|---|---|
| Actor | Administrator |
| Precondition | Administrator is authenticated with admin role |
| Main Flow | 1. Admin opens user management → 2. Creates/edits/deactivates account with role → 3. System hashes new password with bcrypt → 4. Persists row in users table → 5. Confirmation displayed |
| Postcondition | users table reflects requested change; profile rows in students or faculty linked through user_id |
| Exceptions | Username not unique → rejected; weak password → rejected; deactivating self → rejected |
| Includes | Authenticate as Admin |

### Use Case 5: Auto-lock Registration at Deadline
| Field | Detail |
|---|---|
| Actor | System / Timer |
| Precondition | Active semester has a configured registration_deadline |
| Main Flow | 1. Scheduler fires at deadline → 2. System sets flag closing registration window → 3. Subsequent enrollment attempts rejected at application layer → 4. Notification sent to students who started but did not complete |
| Postcondition | Registration window closed; no new enrollments accepted for semester |
| Exceptions | Scheduler down → fallback manual close by administrator; transient lock failure → retried per configured policy |

---

## 14. Diagrams Inventory

All diagrams required for Chapter 2 (System Analysis and Modeling) and Chapter 5 (4+1 Architectural View).

| # | Diagram | Status | Tool | File |
|---|---|---|---|---|
| 1 | Use Case Diagram | ✅ Generated | draw.io XML | SmartCampusHub_UseCaseDiagram.drawio |
| 2 | Class Diagram | ✅ Generated | draw.io XML | SmartCampusHub_ClassDiagram.drawio |
| 3 | Activity Diagram 1 — Course Registration Workflow | ⏳ Pending | PlantUML | — |
| 4 | Activity Diagram 2 — Grade Submission Workflow | ⏳ Pending | PlantUML | — |
| 5 | Sequence Diagram 1 — Student Course Enrollment | ⏳ Pending | PlantUML | — |
| 6 | Sequence Diagram 2 — Faculty Grade Submission with Audit | ⏳ Pending | PlantUML | — |
| 7 | Component Diagram | ⏳ Pending | PlantUML / draw.io | — |
| 8 | Deployment Diagram | ⏳ Pending | PlantUML | — |
| 9 | Timing Diagram | ⏳ Pending | WaveDrom | — |
| 10 | 4+1 Logical View | ⏳ Pending | PlantUML | — |
| 11 | 4+1 Process View | ⏳ Pending | PlantUML | — |
| 12 | 4+1 Development View | ⏳ Pending | PlantUML | — |
| 13 | 4+1 Physical View | ⏳ Pending | PlantUML | — |
| 14 | 4+1 Use Case View (+1) | ⏳ Pending | PlantUML | — |

### Activity Diagram Requirements
- Must include: swimlanes, decision diamonds, fork/join bars for concurrency
- Swimlanes for Diagram 1: Student | System | Database
- Swimlanes for Diagram 2: Faculty | System | Database | Audit

### Sequence Diagram Requirements
- Must include: activation bars, return messages, lifelines for all participants
- Diagram 1 participants: Student, Browser, EnrollmentController, CourseSection, Enrollment, Database
- Diagram 2 participants: Faculty, Browser, GradeController, Grade, AuditLog, Database, Notification Service

---

## 15. Database Schema Notes

The system has exactly **10 tables** matching the 10 domain classes:

```
users
students          (FK: user_id → users)
faculty           (FK: user_id → users)
courses
semesters
course_sections   (FK: course_id, semester_id, faculty_id)
enrollments       (FK: student_id, section_id) + UNIQUE(student_id, section_id)
attendance        (FK: enrollment_id, marked_by) + UNIQUE(enrollment_id, class_date)
grades            (FK: enrollment_id)
audit_log         (FK: changed_by → users)
```

### Key Indexes (for NFR-01 performance)
- `enrollments(student_id)`, `enrollments(section_id)`
- `attendance(enrollment_id)`, `attendance(class_date)`
- `audit_log(table_name, record_id)`, `audit_log(changed_at)`
- `course_sections(semester_id)`, `course_sections(faculty_id)`

### Database Triggers (for NFR-06 auditability)
- AFTER INSERT on `grades` → writes to `audit_log`
- AFTER UPDATE on `grades` → writes to `audit_log`
- AFTER DELETE on `grades` → writes to `audit_log`
- AFTER INSERT on `attendance` → writes to `audit_log`
- AFTER UPDATE on `attendance` → writes to `audit_log`
- BEFORE INSERT on `attendance` → rejects if class_date > CURDATE()

---

## 16. Key Constraints and Business Rules

These are non-negotiable rules enforced at the **database layer** (not just application layer):

| Rule | Enforcement |
|---|---|
| No duplicate enrollment for same student-section pair | UNIQUE(student_id, section_id) on enrollments |
| No attendance record for a future date | BEFORE INSERT trigger on attendance |
| No duplicate attendance for same enrollment on same date | UNIQUE(enrollment_id, class_date) on attendance |
| marks_obtained cannot exceed total_marks | CHECK constraint on grades |
| grade_points must be between 0.00 and 4.00 | CHECK constraint on grades |
| semester end_date must be after start_date | CHECK constraint on semesters |
| All grade and attendance changes are automatically logged | AFTER INSERT/UPDATE/DELETE triggers → audit_log |
| Passwords never stored in plain text | bcrypt hashing enforced at application layer before any DB write |
| Section enrollment cannot exceed max_capacity | Transaction-level check in EnrollmentController |
| Role values are restricted to known set | ENUM('student','faculty','advisor','admin') on users.role |
| Enrollment status is restricted to known set | ENUM('active','dropped','completed') on enrollments.status |
| Attendance status is restricted to known set | ENUM('present','absent','late') on attendance.status |

---



---

## 18. Tooling Decisions

use draw.io format that is the xml format
Both Use Case and Class diagrams have been generated as native draw.io XML files for this reason

---

## 19. Decisions and Rationale Log

This section tracks design decisions made during diagram creation, with rationale, so future sessions don't revisit resolved questions.

| Decision | Rationale |
|---|---|
| **Submit Assignments and View Exam Seating are greyed out / out-of-scope in Use Case diagram** | Submit Assignments is explicitly LMS scope (out of scope per Chapter 1.iii). View Exam Seating has no backing table in the v1.0 schema. Keeping them active in the diagram would contradict the project's own scope section. |
| **Authentication shown as single note + 3 key <<include>> arrows, not 20+ arrows** | Drawing an <<include>> Login arrow from every use case creates unreadable spider web. Textbook standard for large systems is a diagram note stating "all use cases require authentication" with <<include>> shown only for the most architecturally significant cases. |
| **Finance Officer greyed out but present in all diagrams** | Out of scope for v1.0 but included in greyed/dashed form as a future integration signal. Removing it would misrepresent the system's planned evolution. |
| **2 Activity Diagrams (not more)** | Template requires "major workflows" with decisions, concurrency, and swimlanes. Course Registration and Grade Submission are the two most complex and representative workflows. Quality over quantity. |
| **2 Sequence Diagrams (not more)** | Template says "important use cases." Student Enrollment and Grade Submission with Audit are the highest-value interactions showing real multi-object communication with business rules and side effects. |
| **AuditLog.logChange() is a DB trigger, not an application method** | NFR-06 explicitly states "no application-side responsibility." The method is shown in the class diagram for completeness/traceability but annotated as «populated by DB triggers». |
| **Composition used for Enrollment → Attendance and Enrollment → Grade** | These entities have no meaningful existence outside their parent enrollment. If enrollment is deleted, the attendance and grade records have no referential context. This is the textbook definition of composition (strong ownership, cascade delete). |
| **Aggregation used for CourseSection → Student (via Enrollment)** | Students exist independently of sections. A student who drops or completes a section still exists in the system. This is shared ownership, not strong ownership — correct use of aggregation. |

---

*Last updated: Session covering Use Case Diagram + Class Diagram generation and design review.*  
*Generated diagrams: SmartCampusHub_UseCaseDiagram.drawio, SmartCampusHub_ClassDiagram.drawio*
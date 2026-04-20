# ACTIVITY_DIAGRAMS.md — CampusFind: Smart Campus Lost & Found System
## Assignment 8: Activity Workflow Modeling

---

## Overview

This document defines 8 complex workflows in the CampusFind system using UML activity diagrams rendered in Mermaid. Each diagram includes start/end nodes, actions, decision points, parallel actions, and swimlanes showing which actor or system component is responsible for each step.

---

## Workflow 1: User Registration

```mermaid
flowchart TD
    Start([🟢 Start]) --> A

    subgraph Student ["👤 Student"]
        A[Navigate to Registration Page]
        B[Fill in name, university email, password]
        C[Tick POPIA consent checkbox]
        D[Click Register]
        M[Click verification link in email]
    end

    subgraph System ["⚙️ System / API Gateway"]
        E{Email domain valid?}
        F{Email already registered?}
        G[Hash password with bcrypt cost factor 12]
        H[Create inactive user account in PostgreSQL]
        I[Log consent timestamp]
        J[Generate unique verification token]
        N{Token expired?}
        O[Activate account in PostgreSQL]
        P[Redirect to login page]
        Q[Display error: use university email]
        R[Display error: email already exists]
        S[Delete expired inactive account]
    end

    subgraph EmailService ["📧 SendGrid"]
        K[Send verification email with unique link]
        L[Email delivered to student inbox]
    end

    A --> B --> C --> D
    D --> E
    E -- No --> Q --> End1([🔴 End])
    E -- Yes --> F
    F -- Yes --> R --> End2([🔴 End])
    F -- No --> G --> H --> I --> J
    J --> K --> L --> M
    M --> N
    N -- Yes --> S --> End3([🔴 End])
    N -- No --> O --> P --> End4([🟢 End])
```

### Explanation

This workflow maps directly to **FR-01** (user registration with university email validation) and **US-001**. The decision diamond at "Email domain valid?" prevents non-university emails from creating accounts, addressing the **Data Privacy Officer's** concern (S7) that only institutional users access the system. The parallel actions of hashing the password (security) and logging consent (POPIA compliance) happen simultaneously before the verification email is sent. The token expiry check addresses the alternative flow in **UC-01**.

---

## Workflow 2: Submit Lost Item Report

```mermaid
flowchart TD
    Start([🟢 Start]) --> A

    subgraph Student ["👤 Student"]
        A[Navigate to Report Lost Item page]
        B[Fill in item name, category, description, location, date]
        C[Select up to 3 photos]
        D[Click Submit]
    end

    subgraph API ["⚙️ API Gateway"]
        E{All mandatory fields present?}
        F{Photos valid? size ≤ 5MB, image type}
        G[Display field validation errors]
        H[Display photo size error]
    end

    subgraph PhotoService ["☁️ Cloudinary"]
        I[Upload photos to Cloudinary]
        J{Upload successful?}
        K[Return photo URLs]
        L[Retry upload once]
        M[Save report without photos, notify student]
    end

    subgraph ReportService ["⚙️ Report Service"]
        N[Save report to PostgreSQL with status Open]
        O[Enqueue AI matching job in Redis]
        P[Display success confirmation to student]
        Q[Report appears on student dashboard]
    end

    A --> B --> C --> D
    D --> E
    E -- No --> G --> B
    E -- Yes --> F
    F -- No --> H --> C
    F -- Yes --> I --> J
    J -- Yes --> K --> N
    J -- No --> L --> J
    L -- Failed again --> M --> N
    N --> O --> P --> Q --> End([🟢 End])
```

### Explanation

This workflow maps to **FR-03** and **US-003**. The two decision points (field validation and photo validation) implement the acceptance criteria defined in Assignment 5 TC-006 and TC-007. The photo upload retry loop addresses the alternative flow in **UC-02** (network failure during upload). The parallel outcome paths — successful upload vs. report saved without photos — ensure that a network glitch never prevents a student from filing a report at all. This addresses **Stakeholder S1's** concern that the process must be fast and resilient.

---

## Workflow 3: AI Matching Analysis

```mermaid
flowchart TD
    Start([🟢 Start]) --> A

    subgraph Worker ["⚙️ Background Worker"]
        A[Dequeue matching job from Redis]
        B[Fetch new report from PostgreSQL]
        C[Fetch all open reports of opposite type]
        D{Any candidate reports exist?}
        E[Mark job complete - no candidates]
    end

    subgraph AIService ["🤖 AI Matching Service"]
        F[Encode text fields into semantic embeddings]
        G[Compute cosine text similarity for all candidates]
        H{Text similarity > 40%?}
        I[Fetch candidate photos from Cloudinary]
        J[Extract image feature vectors via MobileNetV2]
        K[Compute image cosine similarity]
        L[Aggregate: text score x0.6 + image score x0.4]
        M{Final score >= 70%?}
        N[Return ranked match candidates]
    end

    subgraph Database ["🗄️ PostgreSQL"]
        O[Save match records above threshold]
        P[Enqueue notification jobs for matched report owners]
    end

    A --> B --> C --> D
    D -- No --> E --> End1([🔴 End - No Match])
    D -- Yes --> F --> G --> H
    H -- No --> M
    H -- Yes --> I --> J --> K --> L --> M
    M -- No --> End2([🔴 End - Below Threshold])
    M -- Yes --> N --> O --> P --> End3([🟢 End - Match Found])
```

### Explanation

This workflow maps to **FR-05** and **US-005**. The text similarity pre-filter (> 40% before running image analysis) is a key performance optimisation — image analysis via MobileNetV2 is computationally expensive, so it only runs on candidates that have already passed the text similarity check. This addresses the **IT Department's** concern (S5) about server load and the **NFR-08** requirement that matching completes within 60 seconds for up to 10,000 reports. The two separate end states (No Match vs. Match Found) map directly to the alternative flows in **UC-04**.

---

## Workflow 4: Admin Match Review and Confirmation

```mermaid
flowchart TD
    Start([🟢 Start]) --> A

    subgraph Admin ["👤 Campus Admin"]
        A[Navigate to Matches section]
        B[View pending matches sorted by confidence score]
        C[Select a match to review]
        D[Review side-by-side: lost report vs found report]
        E{Decision}
        F[Click Confirm Match]
        G[Click Dismiss]
        H[Add note and leave in Pending Review]
    end

    subgraph System ["⚙️ System"]
        I[Update both reports to Matched status]
        J[Create handover record]
        K[Log dismissal with admin ID and timestamp]
        L[Remove match from pending queue]
        M[Save note to match record]
    end

    subgraph Notification ["📧 Notification Service"]
        N[Notify student: item confirmed found]
        O[Send pickup instructions]
    end

    A --> B --> C --> D --> E
    E -- Confirm --> F --> I
    I --> J --> N --> O --> L --> End1([🟢 End - Handover Initiated])
    E -- Dismiss --> G --> K --> L --> End2([🔴 End - Match Dismissed])
    E -- Defer --> H --> M --> End3([🟡 End - Pending Review])
```

### Explanation

This workflow maps to **FR-07** and **US-007**. The three-way decision (Confirm / Dismiss / Defer) reflects the real operational reality that admins sometimes need more information before making a decision. The parallel actions after confirmation — updating report statuses, creating the handover record, and sending the student notification — happen concurrently to minimise delay, addressing **Stakeholder S3's** concern that the match-to-notification time must be as short as possible. This maps to **TC-011** and **TC-012** in the test cases from Assignment 5.

---

## Workflow 5: Digital Handover Process

```mermaid
flowchart TD
    Start([🟢 Start]) --> A

    subgraph Admin ["👤 Campus Admin"]
        A[Open handover record]
        B[Set pickup location and time window]
        C[Click Notify Student]
        R[Record Manual Override with reason]
    end

    subgraph System ["⚙️ System"]
        D[Send pickup notification to student]
        E[Start 7-day collection countdown]
        F{Student collects within 3 days?}
        G[Send Day 3 reminder]
        H{Student collects within day 7?}
        I[Send Day 7 reminder and alert admin]
        J{Student collects after escalation?}
        K[Record digital confirmation with timestamp]
        L[Update both reports to Resolved]
        M[Archive handover record with full audit trail]
        N[Mark item as Unclaimed]
        O[Admin notified for disposal]
    end

    subgraph Student ["👤 Student"]
        P[Receive notification]
        Q[Collect item and click confirmation link]
    end

    A --> B --> C --> D --> P
    P --> E --> F
    F -- Yes --> Q --> K --> L --> M --> End1([🟢 End - Resolved])
    F -- No --> G --> H
    H -- Yes --> Q --> K --> L --> M --> End1
    H -- No --> I --> J
    J -- Yes --> Q --> K --> L --> M --> End1
    J -- Manual --> R --> K --> L --> M --> End1
    J -- No --> N --> O --> End2([🔴 End - Unclaimed])
```

### Explanation

This workflow maps to **FR-08** and **US-008**. The escalating reminder sequence (Day 3 → Day 7 → Escalated) addresses the **Campus Admin's** concern (S3) that uncollected items need a structured follow-up process. The Manual Override path ensures that even if a student cannot access the app (e.g., no smartphone), the handover can still be completed and recorded digitally. The full audit trail at the end addresses the **University Administrator's** concern (S4) about legal accountability for item handovers.

---

## Workflow 6: Student Receives and Responds to Match Notification

```mermaid
flowchart TD
    Start([🟢 Start]) --> A

    subgraph NotificationService ["⚙️ Notification Service"]
        A[Dequeue notification job from Redis]
        B[Fetch match details and student email from PostgreSQL]
        C[Create in-app notification record]
        D{Email notifications enabled for student?}
        E[Send email via SendGrid]
        F{Email delivered?}
        G[Retry after 10 minutes]
        H{Retry successful?}
        I[Log delivery failure, alert IT admin]
        J[Update match record to Notified status]
    end

    subgraph Student ["👤 Student"]
        K[Receive email notification]
        L[Log in to CampusFind]
        M[View in-app notification badge]
        N[Click to view match details]
        O{Satisfied this is their item?}
        P[Wait for admin handover workflow]
        Q[Contact admin to report false match]
    end

    A --> B --> C --> D
    D -- Yes --> E --> F
    F -- Yes --> J
    F -- No --> G --> H
    H -- Yes --> J
    H -- No --> I --> J
    D -- No --> J
    J --> K
    K --> L --> M --> N --> O
    O -- Yes --> P --> End1([🟢 End - Awaiting Handover])
    O -- No --> Q --> End2([🔴 End - False Match Reported])
```

### Explanation

This workflow maps to **FR-06** and **US-006**. The email opt-out path reflects the acceptance criterion that students can disable email notifications from their profile. The retry logic (up to 1 retry after 10 minutes) maps to the alternative flow in **UC-05**. The student's response decision (satisfied vs. false match) feeds back into the admin workflow — a false match report triggers admin re-review and potential dismissal of the match. This addresses **Stakeholder S1's** concern about being kept informed and **Stakeholder S3's** concern about match accuracy.

---

## Workflow 7: Admin Generates Statistics Report

```mermaid
flowchart TD
    Start([🟢 Start]) --> A

    subgraph Admin ["👤 University Admin / Super Admin"]
        A[Navigate to Statistics dashboard]
        B[Select date range filter]
        C[View statistics on screen]
        D{Export needed?}
        E[Click Export CSV]
        F[Download CSV to local device]
    end

    subgraph System ["⚙️ System / Stats Service"]
        G{Results cached in Redis?}
        H[Return cached results]
        I[Query PostgreSQL for aggregated statistics]
        J[Cache results in Redis with 10-minute TTL]
        K[Display: recovery rate, top locations, categories, avg resolution time]
        L{Query completes in time?}
        M[Generate anonymised CSV - no personal data]
        N[Send CSV download link via email]
    end

    A --> B --> G
    G -- Yes --> H --> K
    G -- No --> I --> J --> K
    K --> C --> D
    D -- No --> End1([🟢 End])
    D -- Yes --> E --> L
    L -- Under 10s --> M --> F --> End2([🟢 End - CSV Downloaded])
    L -- Over 10s --> N --> End3([🟢 End - Email Sent])
```

### Explanation

This workflow maps to **FR-10** and **US-010**. The Redis caching layer (10-minute TTL) is critical for performance — the statistics query aggregates across potentially tens of thousands of reports, and running it on every page load would degrade system performance. The cache means repeat visits to the statistics page within 10 minutes are near-instantaneous. The email fallback for slow exports addresses **NFR-12** (page load under 3 seconds) — rather than making the admin wait for a slow query, the system responds immediately and delivers the result asynchronously. This addresses **Stakeholder S4's** need for institutional reporting data.

---

## Workflow 8: Automated POPIA Data Archiving and Deletion

```mermaid
flowchart TD
    Start([🟢 Start : Nightly job at 02:00]) --> A

    subgraph Worker ["⚙️ Background Worker"]
        A[Query PostgreSQL for reports resolved more than 90 days ago]
        B{Any reports to archive?}
        C[Mark reports as Archived status]
        D[Query for Archived reports older than 30 more days]
        E{Any reports to delete?}
        F[Delete personal data: name, email, student number, photo URLs]
        G[Delete associated photos from Cloudinary]
        H[Retain anonymised tombstone record: category, location month-year, resolution status]
        I[Log archiving and deletion counts]
        J[Send daily summary email to super admin]
        K[Mark job as complete]
    end

    subgraph Database ["🗄️ PostgreSQL + Cloudinary"]
        L[(Update report records)]
        M[(Delete personal fields)]
        N[(Cloudinary: destroy photo assets)]
    end

    A --> B
    B -- No --> D
    B -- Yes --> C --> C --> L --> D
    D -- No --> I
    D -- Yes --> F --> M
    F --> G --> N
    G --> H --> I
    I --> J --> K --> End([🟢 End])
```

### Explanation

This workflow maps to **FR-11** and **US-011**. It is the most important compliance workflow in the system, directly addressing the requirements of the **Data Privacy Officer (S7)** and POPIA legislation. The two-phase process — archive at 90 days, delete personal data at 120 days — gives the institution a 30-day window to address any disputes or administrative queries before personal data is permanently removed. The anonymised tombstone record ensures that the **University Administrator (S4)** can still generate multi-year statistics without retaining any personal data. The daily summary email ensures the super admin has an auditable record of every deletion cycle.

---

## Traceability Summary

| Diagram | Functional Requirement | User Story | Sprint Task |
|---|---|---|---|
| Workflow 1: User Registration | FR-01 | US-001 | T-003 to T-007 |
| Workflow 2: Submit Lost Report | FR-03 | US-003 | T-012 to T-015 |
| Workflow 3: AI Matching | FR-05 | US-005 | Sprint 2 |
| Workflow 4: Admin Match Review | FR-07 | US-007 | Sprint 2 |
| Workflow 5: Digital Handover | FR-08 | US-008 | Sprint 3 |
| Workflow 6: Match Notification | FR-06 | US-006 | Sprint 2 |
| Workflow 7: Statistics Report | FR-10 | US-010 | Sprint 3 |
| Workflow 8: POPIA Archiving | FR-11 | US-011 | Sprint 3 |
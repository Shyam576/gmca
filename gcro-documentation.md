# GCRO System — Technical Documentation & System Overview

**Gelephu Corporate Registration Office (GCRO)**  
Version: 1.0 | Last Updated: May 2026

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Services Reference](#3-services-reference)
   - [Client Frontend](#31-client-frontend)
   - [Admin Frontend](#32-admin-frontend)
   - [Corporate Registry API](#33-corporate-registry-api)
   - [Admin Auth API](#34-admin-auth-api)
   - [Small Business API](#35-small-business-api)
   - [Payment Service](#36-payment-service)
4. [Entity Relationship — Corporate Registry](#4-entity-relationship--corporate-registry)
5. [Entity Relationship — Admin Auth](#5-entity-relationship--admin-auth)
6. [Entity Relationship — Payment Service](#6-entity-relationship--payment-service)
7. [Cross-Service Data Relationships](#7-cross-service-data-relationships)
8. [Key Workflows](#8-key-workflows)
9. [EOI-to-Incorporation Full Lifecycle](#9-eoi-to-incorporation-full-lifecycle)
   - [Phase 1 — EOI Submission](#phase-1--eoi-submission)
   - [Phase 2 — EOI Review & Decision](#phase-2--eoi-review--decision)
   - [Phase 3 — Credential Issuance & Form Submission](#phase-3--credential-issuance--form-submission)
   - [Phase 4 — Form Review & Approval](#phase-4--form-review--approval)
   - [Phase 5 — Payment](#phase-5--payment)
   - [Phase 6 — Company Incorporation](#phase-6--company-incorporation)
   - [Phase 7 — Post-Incorporation Steps](#phase-7--post-incorporation-steps)
   - [Status State Machines](#status-state-machines)
10. [Form Engine — Deep Dive](#10-form-engine--deep-dive)
    - [Data Model](#data-model)
    - [Field Types](#field-types)
    - [Conditional Logic](#conditional-logic)
    - [Repeatable Groups](#repeatable-groups)
    - [Field Mappings](#field-mappings)
    - [Form Lifecycle](#form-lifecycle)
    - [Complete Example](#complete-example)
11. [Master Data Reference](#11-master-data-reference)
12. [Technology Stack Summary](#12-technology-stack-summary)
13. [Environment & Deployment](#13-environment--deployment)

---

## 1. System Overview

GCRO is a multi-service platform for managing corporate registrations in Gelephu. It enables applicants to:

- Submit an **Expression of Interest (EOI)** to register a company
- Complete dynamic **form-based applications** assigned by the registry
- Track their **company lifecycle** (incorporation, directors, shareholders, charges)
- Make **payments** for registration services
- Communicate with the registry through **escalations and remarks**

The system is split into **six deployable services** across two frontend applications and four backend APIs. All backends are built on **NestJS + TypeORM + PostgreSQL**. Frontends are built on **Next.js**.

---

## 2. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            GCRO Platform                                    │
│                                                                             │
│   ┌─────────────────────┐        ┌────────────────────────────────────┐    │
│   │   CLIENT FRONTEND   │        │        ADMIN FRONTEND              │    │
│   │   (Next.js · gcro)  │        │  (Next.js · gelephu-corp-reg-off)  │    │
│   │   Port 3000         │        │  Port 3001                         │    │
│   └────────┬────────────┘        └──────────────┬─────────────────────┘    │
│            │                                     │                          │
│            │ REST/HTTP                           │ REST/HTTP                │
│            │                                     │                          │
│   ┌────────▼────────────────────────────────────▼─────────────────────┐   │
│   │                   CORPORATE REGISTRY API                           │   │
│   │              (NestJS · awesome-nestjs-boilerplate)                 │   │
│   │                          Port 3003                                 │   │
│   │                  Database: corporate_registry DB                   │   │
│   └──────────────────────────────────┬─────────────────────────────────┘   │
│                                      │                                      │
│                          NATS Microservice Bus                              │
│                                      │                                      │
│   ┌────────────────┐   ┌─────────────▼──────────┐  ┌────────────────────┐  │
│   │ ADMIN AUTH API │   │  SMALL BUSINESS API    │  │   PAYMENT SERVICE  │  │
│   │   (NestJS)     │   │     (NestJS)           │  │     (NestJS)       │  │
│   │   Port 3004    │   │     Port 3005          │  │     Port 3006      │  │
│   │   DB: auth_db  │   │     DB: small_biz_db   │  │   DB: payment_db   │  │
│   └────────────────┘   └────────────────────────┘  └────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

External Integrations:
  ├── RMA Gateway (Royal Monetary Authority — local bank transfers)
  └── Stripe (card payments)
```

> **Note:** The Corporate Registry API acts as the central orchestrator. It communicates with the Payment Service over NATS (microservice messaging) for payment event synchronisation.

---

## 3. Services Reference

### 3.1 Client Frontend

| Property | Value |
|----------|-------|
| **Path** | `client/` |
| **Framework** | Next.js (App Router) |
| **Package name** | `gcro` |
| **Purpose** | Public-facing portal for applicants to submit EOIs, fill forms, track applications, view their company dashboard, raise escalations, and access the public register |

**Key Features / Pages:**

| Feature Module | Description |
|----------------|-------------|
| `new-registration` | Multi-step EOI submission form for new company applicants |
| `user-dashboard` | Authenticated applicant view of their applications and company status |
| `company-dashboard` | Detailed company view — directors, shareholders, documents, events |
| `escalation-form` | Raise and track escalations against form responses |
| `public-register` | Public search of registered companies by UEN / name |

**Auth Flow:** JWT-based authentication via the Corporate Registry API. Applicants are created automatically when an EOI is approved (credentials issued by the registry).

---

### 3.2 Admin Frontend

| Property | Value |
|----------|-------|
| **Path** | `admin/` |
| **Framework** | Next.js (App Router) |
| **Package name** | `gelephu-corporate-registration-office` |
| **Purpose** | Internal portal for GCRO staff and administrators to manage applications, forms, companies, users, roles, CSPs, and analytics |

**Key Features / Pages:**

| Feature Module | Description |
|----------------|-------------|
| `analytics/` | Dashboard with registration stats and reporting metrics |
| Form Builder | Create/publish form templates with sections, fields, conditions, and field mappings |
| EOI Management | Review, approve, reject, and assign CSPs to expressions of interest |
| Company Management | View company details, directors, shareholders, auditors, charges, capital |
| User / Role Management | Administered via Admin Auth API (RBAC) |
| Notifications | In-app notification centre fed by the registry service |

---

### 3.3 Corporate Registry API

| Property | Value |
|----------|-------|
| **Path** | `coporate-registry/` |
| **Framework** | NestJS + TypeORM + PostgreSQL |
| **Base URL** | `/api/v1` |
| **Documentation** | Swagger auto-generated at `/documentation` |
| **Messaging** | NATS microservice client (`NATS_SERVICE`) for async events |

This is the **core backend service**. It owns the application lifecycle from EOI submission through company incorporation. It also contains the entire form engine.

**Modules and Responsibilities:**

| Module | Table | Responsibility |
|--------|-------|----------------|
| `auth` | — | JWT auth for applicant users (client portal) |
| `user` | `users` | Applicant user accounts, linked to EOI groups |
| `expression-of-interest` | `expression_of_interests` | EOI lifecycle (submit → review → approve/reject → credentials) |
| `eoi-group` | `eoi_groups` | Groups related EOIs and users together under one company journey |
| `eoi-document` | `eoi_documents` | Documents uploaded during EOI |
| `eoi-remark` | `eoi_remarks` | Remarks made by staff on an EOI |
| `eoi-remark-comment` | `eoi_remark_comments` | Threaded comments on EOI remarks |
| `csp` | `csps` | Corporate Service Providers (registry-level entities) |
| `csp-detail` | `csp_details` | Detailed profile of a CSP |
| `csp-document` | `csp_documents` | Documents submitted by a CSP |
| `csp-assignment` | `company_csp_assignments` | Assignment of a CSP to a company |
| `entity-type` | `entity_types` | Types of legal entities (Private Limited, etc.) |
| `entity-type-prefix` | `entity_type_prefixes` | Registration number prefix rules per entity type |
| `form-template` | `form_templates` | Dynamic form definitions (publishable, version-controlled) |
| `form-section` | `form_sections` | Sections within a form template |
| `form-field` | `form_fields` | Individual fields within a section |
| `form-field-condition` | `form_field_conditions` | Conditional display rules for fields |
| `form-field-mapping` | `form_field_mappings` | Maps form responses to company entity fields |
| `form-repetable-group` | `form_repetable_groups` | Repeatable field groups (e.g., multiple directors) |
| `form-response` | `form_responses` | An applicant's submitted answers to a form |
| `form-response-answer` | `form_response_answers` | Individual field answers |
| `form-response-answer-audit-trail` | `form_response_answer_audit_trails` | Change history for answers |
| `form-response-repetable-group` | `form_response_repetable_groups` | Filled repeatable groups in a response |
| `form-action` | `form_actions` | Actions taken on a form response (approve/reject/request-info) |
| `assignee` | `assignees` | Staff assigned to review a form response |
| `auditor` | `auditors` | Registered auditing firms |
| `company` | `companies` | Incorporated companies |
| `company-user` | `company_users` | M2M join: users with roles within a company |
| `company-event` | `company_events` | Timeline events on a company (AGM, AR, etc.) |
| `company-auditor` | `company_auditors` | Auditors appointed to a company |
| `company-capital` | `company_capitals` | Issued/paid-up share capital |
| `company-charge` | `company_charges` | Registered charges against the company |
| `company-shareholder` | `company_shareholders` | Shareholders of the company |
| `company-guarantor` | `company_guarantors` | Guarantors (for companies limited by guarantee) |
| `company-remark` | `company_remarks` | Internal staff remarks on a company |
| `director` | `directors` | Directors of a company |
| `secretary` | `secretaries` | Company secretaries |
| `owner` | `owners` | Owner records |
| `subscriber` | `subscribers` | Initial subscribers of the company |
| `third-party` | `third_parties` | Third-party relationships |
| `payment` | `payments` | Company-level payment reference (links company to transaction) |
| `payment-master` | `payment_masters` | Configurable payment fee schedule |
| `documents` | `documents` | Company-level document storage |
| `notification` | `notifications` | In-app notifications delivered to users |
| `escalation` | `escalations` | Escalation threads raised on form responses |
| `escalation-comment` | `escalation_comments` | Comments on an escalation |
| `contact-us` | `contactus` | Public contact form entries |
| `contact-us-response` | `contact_us_responses` | Staff replies to contact form entries |
| `certificate-template` | `certificate_templates` | Certificate templates per entity type |
| `office-address-details` | `office_address_details` | Registered office addresses |
| `faq` | `faqs` | FAQ entries |
| `professional-service` | `professional_services` | Professional service type registry |
| `reporting` | — | Aggregated reporting endpoints |
| `eoi-assignment` | `eoi_assignments` | Staff assignments to EOI reviews |
| `health-checker` | — | `GET /health` liveness probe |

---

### 3.4 Admin Auth API

| Property | Value |
|----------|-------|
| **Path** | `admin-auth/` |
| **Framework** | NestJS + TypeORM + PostgreSQL |
| **Purpose** | Authentication and RBAC (Role-Based Access Control) for GCRO **staff / admin** users only |

This service is intentionally separate from the Corporate Registry API so that staff identity management is isolated from applicant data.

**Modules:**

| Module | Table | Responsibility |
|--------|-------|----------------|
| `auth` | — | Login, register, password reset, 2FA (TOTP), logout |
| `user` | `users` | Admin/staff user accounts with 2FA and role assignments |
| `user-settings` | `user_settings` | Per-user preferences |
| `role` | `roles` | Named roles (e.g., Registrar, Reviewer, CSP Officer) |
| `permission` | `permissions` | Fine-grained permission slugs |
| `guidelines` | `guidelines` | In-app guidance content for staff |
| `health-checker` | — | Liveness probe |

**RBAC Model:**  
`User` ←M2M→ `Role` ←M2M→ `Permission`  
Roles and permissions are resolved at login and embedded in the JWT. Permissions use a slug-based system (e.g., `companies.view`, `eoi.approve`).

---

### 3.5 Small Business API

| Property | Value |
|----------|-------|
| **Path** | `small-business/` |
| **Framework** | NestJS + TypeORM + PostgreSQL |
| **Purpose** | Auth and user management for the **Small Business** registration track — mirrors the Admin Auth API structure but for a different user domain |

**Modules:**

| Module | Table | Responsibility |
|--------|-------|----------------|
| `auth` | — | Login, register, password reset, 2FA, logout |
| `user` | `users` | Small-business applicant user accounts |
| `user-settings` | `user_settings` | Per-user preferences |
| `role` | `roles` | Roles within the small-business context |
| `permission` | `permissions` | Fine-grained permission slugs |
| `guidelines` | `guidelines` | In-app guidance content |
| `health-checker` | — | Liveness probe |

> The Small Business API shares the same schema shape as Admin Auth. It is kept separate to allow independent scaling and per-track policy enforcement.

---

### 3.6 Payment Service

| Property | Value |
|----------|-------|
| **Path** | `payment/` |
| **Framework** | NestJS + TypeORM + PostgreSQL |
| **Purpose** | Central payment processing hub for all GCRO registration fees, supporting multiple payment gateways |

**Supported Gateways:**

| Gateway | Description |
|---------|-------------|
| **RMA Gateway** | Royal Monetary Authority — local bank / BFS transfer protocol |
| **Stripe** | International card payments |

**Modules:**

| Module | Table | Responsibility |
|--------|-------|----------------|
| `auth` | `users` | JWT auth for the payment service (service-to-service) |
| `payment` | `payments` | Main payment record linked to an `application_id` |
| `payment-master` | `payment_masters` | Configurable fee schedule (name, amount, currency) |
| `payment-method` | `payment_methods` | Available methods (code, name, transaction fee) |
| `transaction` | `transactions` | Immutable transaction log with order number |
| `rma-transaction` | `rma_transactions` | Full BFS/RMA message payload for bank transfers |
| `stripe` | `stripe_logs` | Stripe webhook and event log |
| `exchange-rate-log` | `exchange_rate_logs` | Exchange rate applied to each payment conversion |
| `rma-gateway` | — | Outbound BFS protocol handler for RMA |
| `health-checker` | — | Liveness probe |

---

## 4. Entity Relationship — Corporate Registry

The diagram below shows the primary relationships between entities within the Corporate Registry database.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         CORPORATE REGISTRY DB                            │
│                                                                          │
│  ┌─────────────┐        ┌──────────────────────┐                        │
│  │  EoiGroup   │───1:N──│  ExpressionOfInterest│                        │
│  │             │        │  (application_no,    │                        │
│  │  id         │        │   status, email,     │                        │
│  │  name       │        │   entityTypeId,      │                        │
│  │             │───1:N──│   eoiGroupId)        │                        │
│  │             │        └──────────┬───────────┘                        │
│  │             │                   │1:N                                  │
│  │             │        ┌──────────▼───────────┐                        │
│  │             │        │    EoiDocument       │                        │
│  │             │        │    EoiRemark         │                        │
│  │             │        │    ClientRemark      │                        │
│  │             │        └──────────────────────┘                        │
│  │             │                                                         │
│  │             │───1:N──┌──────────────────────┐                        │
│  │             │        │       User            │                        │
│  │             │        │  (eoiGroupId FK,     │                        │
│  │             │        │   role, email,       │                        │
│  │             │        │   userType)          │                        │
│  │             │        └──────────┬───────────┘                        │
│  │             │                   │1:N                                  │
│  │             │        ┌──────────▼───────────┐                        │
│  │             │        │    FormResponse       │                        │
│  │             │        │    Notification       │                        │
│  │             │        └──────────┬───────────┘                        │
│  │             │                   │1:N                                  │
│  │             │        ┌──────────▼───────────┐                        │
│  │             │        │  FormResponseAnswer  │                        │
│  │             │        │  FormResponseRep.Grp │                        │
│  │             │        │  Assignee            │                        │
│  │             │        │  FormAction          │                        │
│  │             │        │  Escalation ─────────┼──1:N──EscalationComment│
│  │             │        └──────────────────────┘                        │
│  │             │                                                         │
│  │             │───1:1──┌──────────────────────┐                        │
│  │             │        │       Company         │                        │
│  └─────────────┘        │  (uen, companyName,  │                        │
│                         │   status,            │                        │
│                         │   companyType,       │                        │
│                         │   entityTypeId)      │                        │
│                         └──────────┬───────────┘                        │
│                                    │                                     │
│               ┌────────────────────┼────────────────────────────┐       │
│               │                    │                            │       │
│          1:N  │             1:N    │  1:N                1:N   │       │
│    ┌──────────▼───┐   ┌───────────▼──┐  ┌──────────────┐  ┌───▼────┐  │
│    │  CompanyUser  │   │   Director   │  │  Secretary   │  │Payment │  │
│    │ (userId FK,   │   │ (companyId,  │  │ (companyId,  │  │(compId,│  │
│    │  companyId,   │   │  fullName,   │  │  directorId?,│  │amount, │  │
│    │  role)        │   │  status)     │  │  cspId?)     │  │status) │  │
│    └──────┬────────┘   └──────────────┘  └──────────────┘  └────────┘  │
│           │                                                              │
│           │                                                              │
│    ┌──────▼──────────────┐                                              │
│    │    User (see above) │                                              │
│    └─────────────────────┘                                              │
│                                                                          │
│    Company ──1:N──► CompanyShareholder  (name, shares, shareType)       │
│    Company ──1:N──► CompanyGuarantor    (name, amountGuaranteed)        │
│    Company ──1:N──► CompanyCapital      (issuedAmount, paidUpAmount)    │
│    Company ──1:N──► CompanyCharge       (chargee, amountSecured)        │
│    Company ──1:N──► CompanyAuditor ─────ManyToOne──► Auditor            │
│    Company ──1:N──► CompanyEvent        (type, date)                    │
│    Company ──1:N──► CompanyRemark       (staff remarks)                 │
│    Company ──1:N──► Documents           (file storage)                  │
│    Company ──ManyToOne──► EntityType    (Private Ltd, etc.)             │
│                                                                          │
│    ── FORM ENGINE ──────────────────────────────────────────────────    │
│                                                                          │
│    EntityType ──1:N──► FormTemplate                                     │
│    FormTemplate ──1:N──► FormSection                                    │
│    FormSection  ──1:N──► FormField                                      │
│    FormField    ──1:N──► FormFieldCondition                             │
│    FormField    ──1:N──► FormFieldMapping (maps field → company column) │
│    FormSection  ──1:N──► FormRepetableGroup                             │
│                                                                          │
│    FormResponse ──ManyToOne──► FormTemplate                             │
│    FormResponse ──ManyToOne──► User                                     │
│    FormResponse ──1:N──► FormResponseAnswer                             │
│    FormResponse ──1:N──► FormResponseRepetableGroup                     │
│    FormResponse ──1:N──► Assignee                                       │
│    FormResponse ──1:1──► FormAction                                     │
│                                                                          │
│    ── CSP TRACK ────────────────────────────────────────────────────    │
│                                                                          │
│    Csp (companyId) ──ManyToOne──► Company                               │
│    CspDetail (eoiId) ──► ExpressionOfInterest                           │
│    CspDocument (eoiId / cspId)                                          │
│                                                                          │
│    ── NOTIFICATIONS ────────────────────────────────────────────────    │
│                                                                          │
│    Notification ──ManyToOne──► User                                     │
│    (metadata: escalationId, responseId, companyId)                      │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Key Entity Descriptions — Corporate Registry

#### `EoiGroup`
The top-level container that ties together all related entities for a single company journey. A group has many EOIs, many users, and eventually one Company.

```
eoi_groups
  id (UUID PK)
  name
  description
  ─── Relations ───
  expressionOfInterests[]  (1:N)
  users[]                  (1:N)
  company?                 (1:1 — set after incorporation)
```

#### `ExpressionOfInterest`
Captures an applicant's initial intent to register a company. Goes through a multi-stage lifecycle.

```
expression_of_interests
  id (UUID PK)
  applicationNo
  email
  entityTypeId
  eoiGroupId (FK → eoi_groups)
  applicantType        (enum: INDIVIDUAL | CORPORATE | CSP)
  companyInformation   (JSON)
  businessPlans
  country
  onlinePresence
  isCsp
  cspRequired / cspAssigned
  credentialsIssued
  status               (enum: SUBMITTED | UNDER_REVIEW | APPROVED | REJECTED | REAPPLIED)
  approvedById / approvedByName / approvedAt
  rejectedById / rejectedByName / rejectedRemarks
  assessmentRemarks
  ─── Relations ───
  eoiDocuments[]       (1:N)
  eoiRemarks[]         (1:N)
  clientRemarks[]      (1:N)
  entityType?          (ManyToOne → entity_types)
  eoiGroup?            (ManyToOne → eoi_groups)
```

#### `Company`
The incorporated entity. Created after an EOI is fully approved and a form response is submitted and processed.

```
companies
  id (UUID PK)
  applicationId        (FK → form_responses via application_no)
  responseId           (FK → form_responses)
  entityTypeId
  uen                  (Unique Entity Number)
  companyName
  formerName
  registrationNumber
  registrationDate
  incorporationDate
  nameChangeDate
  financialYearEnd
  lastAgmDate / lastArDate
  status               (enum: PENDING | APPROVED | REJECTED | ACTIVE | DISSOLVED)
  companyStatus        (enum: LIVE_COMPANY | STRUCK_OFF | DISSOLVED ...)
  companyType          (enum: based on entity category)
  registeredOfficeAddress
  companyEmail
  businessActivities
  certificateEmailSent / certificateConfirmed / certificateConfirmationDate
  ─── Relations ───
  companyUsers[]       (1:N → company_users)
  events[]             (1:N → company_events)
  companyAuditors[]    (1:N → company_auditors)
  companyCharges[]     (1:N → company_charges)
  companyCapitals[]    (1:N → company_capitals)
  secretaries[]        (1:N → secretaries)
  companyShareholders[](1:N → company_shareholders)
  companyGuarantors[]  (1:N → company_guarantors)
  documents[]          (1:N → documents)
```

#### `Director`
```
directors
  id (UUID PK)
  companyId  (FK → companies)
  fullName / nationality / passportNo / dateOfBirth
  residentialAddress / phoneNumber / email
  positionHeld / dateOfAppointment
  isResidentDirector
  status          (enum: PENDING | ACTIVE | REMOVED)
  approvedById / approvedByName / approvedAt
  rejectedById / rejectedByName / rejectedReason / rejectedAt
  signatureDocument / employmentPass
```

#### `Secretary`
```
secretaries
  id (UUID PK)
  companyId   (FK → companies)
  directorId? (FK → directors — when director serves as secretary)
  cspId?      (FK → csps — when CSP provides secretary)
  fullName / nationality / passportNo / email / phoneNumber
  occupation / residentialAddress / dateOfBirth
  appointmentDate / employmentPass / kycDocument
  status      (enum: PENDING | ACTIVE | REMOVED)
```

#### `FormResponse` (Application Submission)
Links a user's answers to a specific form template for a given EOI.

```
form_responses
  id (UUID PK)
  formTemplateId  (FK → form_templates)
  eoiId           (FK → expression_of_interests)
  userId          (FK → users)
  applicationNo
  status          (enum: DRAFT | SUBMITTED | UNDER_REVIEW | APPROVED | REJECTED)
  submittedAt / sentForApproval
  ─── Relations ───
  formTemplate     (ManyToOne)
  user             (ManyToOne)
  answers[]        (1:N → form_response_answers)
  repetableGroups[](1:N → form_response_repetable_groups)
  assignees[]      (1:N → assignees)
  formAction?      (1:1 → form_actions)
```

---

## 5. Entity Relationship — Admin Auth

```
admin_auth DB

  ┌───────────────┐        ┌──────────────────┐
  │     User      │ M2M    │      Role         │ M2M   ┌────────────────┐
  │               │◄──────►│                  │◄─────►│   Permission   │
  │  id           │        │  id              │       │                │
  │  name         │        │  name            │       │  id            │
  │  email        │        │  description     │       │  name          │
  │  password     │        │  slug            │       │  description   │
  │  phone        │        │  permissions[]   │       │  slug          │
  │  avatar       │        │  users[]         │       │  roles[]       │
  │  userType     │        └──────────────────┘       └────────────────┘
  │  isActive     │
  │  isTwoFaEnabled│       Join table: user_roles (user_id, role_id)
  │  twoFactorSecret│      Join table: role_permissions (role_id, permission_id)
  │  roles[]      │
  │  settings?    │        ┌──────────────────┐
  └───────┬───────┘        │  UserSettings    │
          │                │  id              │
          │1:1             │  userId (FK)     │
          └───────────────►│  (preferences)   │
                           └──────────────────┘

  ┌──────────────────┐
  │   Guidelines     │
  │  (standalone)    │
  │  id, title,      │
  │  content, slug   │
  └──────────────────┘
```

**Pivot tables:**

| Table | Columns |
|-------|---------|
| `user_roles` | `user_id` (FK → users), `role_id` (FK → roles) |
| `role_permissions` | `role_id` (FK → roles), `permission_id` (FK → permissions) |

---

## 6. Entity Relationship — Payment Service

```
payment DB

  ┌─────────────────┐
  │  PaymentMaster  │  (fee schedule — configured by admin)
  │  id             │
  │  name           │
  │  amount         │
  │  currency       │
  └─────────────────┘

  ┌─────────────────┐     1:1    ┌──────────────────┐
  │  PaymentMethod  │◄──────────│    Payment        │
  │  id             │            │  id              │
  │  name / code    │            │  applicationId   │ ← references Corporate Registry
  │  description    │            │  serviceType     │
  │  currency       │            │  amount          │
  │  transactionFee │            │  currency        │
  │  status         │            │  exchangeRate    │
  └─────────────────┘            │  paymentMethodId │
                                 │  transactionNo   │
                                 │  transactionDate │
                                 │  status (enum)   │
                                 │  createdById     │
                                 │  createdByName   │
                                 └────────┬─────────┘
                                          │ 1:N
                             ┌────────────▼─────────────┐
                             │    RmaTransaction         │
                             │  (full BFS message log    │
                             │   for RMA bank transfers) │
                             │  paymentId (FK)           │
                             │  applicationNo            │
                             │  bfsMsgType               │
                             │  bfsTxnAmount             │
                             │  bfsRemitterEmail         │
                             │  ...                      │
                             └──────────────────────────┘

  ┌──────────────────┐
  │   Transaction    │  (order-level record, immutable)
  │  id              │
  │  applicationId   │ ← references Corporate Registry
  │  orderNo (unique)│
  │  currency        │
  │  amount          │
  │  paymentMethodId │
  │  transactionNo   │
  │  transactionDate │
  │  status (enum)   │
  └──────────────────┘

  ┌──────────────────┐
  │  ExchangeRateLog │  (audit trail for currency conversions)
  │  id              │
  │  paymentId (FK)  │
  │  amount          │
  │  exchangeRate    │
  │  newAmount       │
  │  status          │
  └──────────────────┘

  ┌──────────────────┐
  │    StripeLog     │  (Stripe webhook event log)
  │  id              │
  │  ...             │
  └──────────────────┘
```

---

## 7. Cross-Service Data Relationships

Because each service has its own isolated database, cross-service data is linked by **shared identifiers** (not foreign key constraints). The table below documents these logical linkages:

| From Service | Field | To Service | Meaning |
|---|---|---|---|
| Payment | `payments.application_id` | Corporate Registry | Links a payment record to a company application |
| Payment | `transactions.application_id` | Corporate Registry | Links an order to a company application |
| Corporate Registry | `company.applicationId` | Corporate Registry `form_responses` | The form response that produced this company |
| Corporate Registry | `expression_of_interests.entityTypeId` | Corporate Registry `entity_types` | The legal entity type selected in the EOI |
| Corporate Registry | `company_users.userId` | Corporate Registry `users` | Links portal users to companies with a role |
| Corporate Registry | `payments.company_id` (in registry) | Corporate Registry `companies` | Registry-side payment reference (not the Payment Service payment) |
| Admin Auth | JWT `roles` / `permissions` | Admin Frontend | JWT claims drive admin UI access control |
| Corporate Registry | NATS event publish | Payment Service | Triggers payment requests when application is approved |

### Inter-Service Communication

```
Corporate Registry API
        │
        │  NATS Microservice (publish event)
        │  e.g., "payment.initiate" { applicationId, amount, serviceType }
        ▼
  Payment Service
        │
        │  Processes payment (RMA / Stripe)
        │  Updates payment status
        │
        │  NATS event (publish back)
        │  e.g., "payment.confirmed" { applicationId, transactionNo }
        ▼
  Corporate Registry API
        │
        └─► Updates company / form response status
```

---

## 8. Key Workflows

### 8.1 Company Registration (Full Journey)

```
1. Public User visits Client Frontend
   └─► Submits EOI (new-registration form)
       └─► POST /api/v1/expression-of-interest
           ├─► Creates ExpressionOfInterest record
           ├─► Creates EoiGroup
           └─► Sends confirmation email

2. Admin Staff reviews EOI in Admin Frontend
   ├─► Views EOI list, assigns reviewer (EoiAssignment)
   ├─► May assign CSP (cspRequired = true)
   └─► Approves EOI
       └─► PATCH /api/v1/expression-of-interest/:id/approve
           ├─► Issues credentials (creates User account)
           ├─► Sends login email to applicant
           └─► Sets credentialsIssued = true

3. Applicant logs in to Client Frontend
   └─► Fills the dynamic Form (FormTemplate → FormResponse)
       ├─► Saves answers as FormResponseAnswers
       └─► Submits form
           └─► POST /api/v1/form-response/:id/submit

4. Admin reviews Form Response
   ├─► Assigns reviewer (Assignee)
   ├─► May raise Escalation (field-level clarification)
   │   └─► Applicant responds via Client Frontend escalation-form
   └─► Approves → FormAction created
       └─► Form field mappings write data to Company entity
           (FormFieldMapping: formFieldId → company column)

5. Payment is initiated
   └─► NATS event → Payment Service creates Payment record
       ├─► Applicant chooses payment method
       └─► Completes payment (RMA or Stripe)
           └─► NATS confirmation → Corporate Registry updates company status

6. Company is now ACTIVE
   └─► Certificate is generated and emailed
       └─► Company visible in Public Register
```

### 8.2 Form Engine

The form engine is fully dynamic — no hardcoded forms.

```
EntityType (e.g., "Private Limited Company")
    └─► FormTemplate (published form definition)
        └─► FormSection[] (logical groupings)
            └─► FormField[] (question definitions)
                ├─► fieldType (text, select, date, file, repeatable...)
                ├─► FormFieldCondition[] (show/hide rules)
                └─► FormFieldMapping[] (writes answer to company.columnName)
```

When a FormResponse is approved, the system iterates `FormFieldMapping` records and writes the corresponding `FormResponseAnswer` values directly to the appropriate `Company`, `Director`, `Secretary`, etc. columns.

### 8.3 RBAC (Admin)

```
Admin logs in (Admin Auth API)
  └─► JWT issued with embedded role slugs / permission slugs
      └─► Admin Frontend reads JWT → shows/hides UI elements
          └─► API calls include JWT
              └─► Corporate Registry API verifies JWT
                  └─► Guards check required permission slug
```

---

---

## 9. EOI-to-Incorporation Full Lifecycle

This section documents the complete journey from the moment a public applicant submits an EOI through to a fully incorporated company, and every step that follows.

---

### Phase 1 — EOI Submission

**Actor:** Public user (no account required)  
**Entry point:** Client Frontend → `new-registration` page

```
Applicant fills out EOI form
  │
  ├── Selects Entity Type  (e.g., Private Company Limited by Shares)
  ├── Selects Applicant Type
  │     ├── APPLICANT       → standard registration path
  │     └── CSP_PROVIDER    → Corporate Service Provider path
  │                            (additional CspDetail fields collected)
  ├── Enters:
  │     companyInformation  (company name, proposed activities, etc.)
  │     businessPlans
  │     onlinePresence
  │     country
  │     email               (this becomes the login email on approval)
  │
  └── Submits
        │
        POST /api/v1/expression-of-interest
        │
        ├── Generates unique applicationNo
        │     format: EOI-YYYYMMDD-XXXX (retry on collision)
        │
        ├── Creates EoiGroup
        │     (one group per company journey — all future users
        │      and the eventual company are linked to this group)
        │
        ├── Creates ExpressionOfInterest
        │     status = SUBMITTED
        │
        ├── [If applicant is CSP] Creates CspDetail record
        │
        ├── Sends EOI confirmation email to applicant
        │
        └── Sends in-app notification + email to all
              Intermediate Officers and Processing Officers
              (resolved from Admin Auth via NATS → get-role-based-user)
```

**EOI record at this point:**
```
status            = submitted
credentialsIssued = false
cspAssigned       = false
cspRequired       = false
```

---

### Phase 2 — EOI Review & Decision

**Actor:** Admin Staff (Intermediate Officer / Processing Officer)  
**Entry point:** Admin Frontend → EOI Management

```
Admin opens EOI list → selects a submitted EOI

  ├── [Optional] Assign reviewer
  │     POST /api/v1/eoi-assignment
  │     Creates EoiAssignment record
  │
  ├── [Optional] Mark CSP required
  │     PATCH /api/v1/expression-of-interest/:id/csp-required
  │     Sets cspRequired = true
  │
  ├── [Optional] Assign a CSP company
  │     POST /api/v1/company-csp-assignment
  │     Links a registered CSP company to the EOI
  │     Sets cspAssigned = true
  │
  ├── [Optional] Add EOI remarks (internal notes)
  │     POST /api/v1/eoi-remark
  │     → threaded via EoiRemarkComment
  │
  ├── [Optional] Add client-visible remarks / request more info
  │     POST /api/v1/client-remark
  │     → Applicant sees these in client portal
  │     → Applicant responds → EOI status set to RESUBMITTED
  │
  ├── APPROVE
  │     PATCH /api/v1/expression-of-interest/:id/approve
  │     Sets:
  │       status          = approved
  │       approvedById    = staff user id
  │       approvedByName  = staff user name
  │       approvedAt      = now()
  │       assessmentRemarks
  │
  └── REJECT
        PATCH /api/v1/expression-of-interest/:id/reject
        Sets:
          status           = rejected
          rejectedById     = staff user id
          rejectedByName   = staff user name
          rejectedRemarks  = reason
          rejectedAt       = now()
        Sends rejection email to applicant
```

**EOI Status Transitions:**
```
submitted → accessed → approved
                    → rejected → [applicant reapplies] → resubmitted → accessed → ...
         → resubmitted (after client-remark response)
```

---

### Phase 3 — Credential Issuance & Form Submission

**Actor:** Admin Staff issues credentials; Applicant submits form  

```
Admin issues credentials
  PATCH /api/v1/expression-of-interest/:id/issue-credentials
  │
  ├── Validates credentialsIssued = false (prevents double-issuance)
  ├── Generates a secure random password
  ├── Creates User account
  │     email     = EOI email
  │     eoiGroupId = EOI's group id (links user to the journey)
  │     userType  = user
  │
  ├── Sets credentialsIssued = true on the EOI
  │
  ├── Sends welcome email to applicant (email + password)
  │
  └── [Optional] Sends same credentials to extra emails
        (for cases with multiple stakeholders)

──────────────────────────────────────────────

Applicant logs into Client Frontend
  │
  ├── Sees their EOI in user-dashboard
  │
  └── Assigned FormTemplate is available to fill
        (FormTemplate is linked to the EntityType selected in EOI)
        │
        POST /api/v1/form-response  (creates DRAFT)
        │
        ├── Fills sections progressively (auto-saved as DRAFT)
        ├── Uploads files per file-upload fields
        │
        └── Submits
              PATCH /api/v1/form-response/:id/submit
              Sets status = submitted
```

---

### Phase 4 — Form Review & Approval

**Actor:** Admin Staff  
**Entry point:** Admin Frontend → Form Responses / Applications list

```
Admin opens submitted form response
  │
  ├── Assigns reviewer
  │     POST /api/v1/assignee
  │     Creates Assignee record (assignedToId, assignedToName, reasons)
  │     Triggers ASSIGNMENT notification to assigned staff member
  │
  ├── Reviews each section's answers
  │
  ├── [If clarification needed] Raises Escalation
  │     POST /api/v1/escalation
  │     escalationType = ADMIN  (raised by staff)
  │     or             = CLIENT (raised by applicant)
  │     priority: low / medium / high
  │     status:   open / pending / resolved
  │     ─── Triggers ESCALATION notification ───
  │     Applicant responds in client portal → adds EscalationComment
  │     Admin resolves → status = resolved
  │
  ├── PRELIMINARY APPROVE
  │     Sets FormResponse.status = prelim_approved
  │     Records: FormAction.prelimApprovedById/Name/Remarks
  │     → Triggers SENT_FOR_APPROVAL step
  │
  ├── PRELIMINARY REJECT
  │     Sets FormResponse.status = prelim_rejected
  │     Applicant can resubmit → status = resubmitted
  │
  ├── ASSESS
  │     Sets FormResponse.status = assessed
  │     Records: FormAction.assessedById/Name
  │
  └── FINAL APPROVE
        Sets FormResponse.status = approved
        Records: FormAction.approvedById/Name
        │
        └── TRIGGERS FIELD MAPPING EXECUTION
              Iterates FormFieldMapping records for the template
              Writes FormResponseAnswer values to:
                - Company entity columns
                - Director records
                - Secretary records
                - Shareholder records
                - etc.
              (see Form Engine section for mapping details)
```

**FormResponse Status Flow:**
```
draft → submitted → action_to_be_taken → assessed → sent_for_approval
                                                   → prelim_approved → approved ✓
                                                   → prelim_rejected → resubmitted → ...
```

---

### Phase 5 — Payment

**Actor:** Applicant pays; Payment Service processes  

```
After form approval, payment is initiated
  │
  ├── Admin or system triggers payment request
  │     Publishes NATS event → Payment Service
  │     Payload: { applicationId, serviceType, amount, currency }
  │
  ├── Payment Service creates Payment record
  │     applicationId = form response application number
  │     status        = pending
  │     exchangeRate  = current rate (logged in ExchangeRateLog)
  │
  ├── Applicant selects payment method in client portal
  │     Available methods defined in payment_methods table
  │
  ├── PATH A — RMA / BFS Bank Transfer
  │     Outbound BFS message sent to RMA gateway
  │     RmaTransaction record created (full BFS payload stored)
  │     Bank confirms → webhook callback received
  │     Payment.status → paid
  │     Transaction record created (orderNo, transactionNo)
  │
  └── PATH B — Stripe Card Payment
        Stripe checkout session created
        Stripe webhook fires on success
        StripeLog record created
        Payment.status → paid
        Transaction record created
```

---

### Phase 6 — Company Incorporation

**Actor:** System (automatic after payment confirmation) / Admin

```
Payment confirmed event received by Corporate Registry API (via NATS)
  │
  ├── Looks up company record (created from field mappings in Phase 4)
  │
  ├── Updates Company
  │     status        = approved → issued
  │     companyStatus = live_company
  │     uen           = assigned (Unique Entity Number)
  │     incorporationDate = today
  │
  ├── Generates Certificate of Incorporation
  │     CertificateTemplate looked up by entityTypeId + CertificateType
  │     Template body rendered with company data
  │     Certificate sent to company email
  │     Sets certificateEmailSent = true
  │
  └── Company appears in Public Register (client/public-register)
```

**Company Status after each phase:**

| Phase | `status` | `companyStatus` |
|-------|----------|-----------------|
| Form approved (field mapping done) | `pending` | `live_company` |
| Payment confirmed | `approved` | `live_company` |
| Certificate issued | `issued` | `live_company` |

---

### Phase 7 — Post-Incorporation Steps

Once a company is incorporated, the following activities are managed through the **Company Dashboard** in the client portal and the **Admin Frontend**.

#### 7.1 Resident Director Assignment

A Bhutanese resident director is required. This is tracked separately from directors added during the form.

```
Client Portal → Company Dashboard → Directors tab

Applicant submits Resident Director details:
  POST /api/v1/director
    companyId
    fullName / nationality / passportNo / dateOfBirth
    residentialAddress / phoneNumber / email
    positionHeld / dateOfAppointment
    isResidentDirector = true
    employmentPass    (document upload)
    signatureDocument (document upload)
    status            = pending

Admin reviews and acts:
  PATCH /api/v1/director/:id/approve
    status          = approved
    approvedById / approvedByName / approvedAt

  PATCH /api/v1/director/:id/reject
    status          = rejected
    rejectedById / rejectedByName / rejectedReason / rejectedAt

CompanyEvent created:
  type     = director_assignment
  dueDate  = appointment date
  Notifications fired:
    - 1 month before due date (notifiedBefore1m)
    - 1 week  before due date (notifiedBefore1w)
```

**Director Status Flow:**
```
pending → approved ✓
        → rejected → [re-submit]
```

#### 7.2 Company Secretary Assignment

```
Client Portal → Company Dashboard → Secretaries tab

Applicant submits Secretary details:
  POST /api/v1/secretary
    companyId
    fullName / nationality / passportNo
    email / phoneNumber / occupation
    residentialAddress / dateOfBirth
    appointmentDate
    directorId?   (if a director is serving as secretary)
    cspId?        (if a CSP company provides the secretary)
    employmentPass (document)
    kycDocument    (document)
    status         = pending

Admin approves or rejects (same pattern as Director)
  status → approved / rejected

CompanyEvent created:
  type = secretary_assignment
```

**Secretary Status Flow:**
```
pending → approved ✓
        → rejected → [re-submit]
```

#### 7.3 Registered Office Address

```
Client Portal → Company Dashboard → Office Address tab

Applicant submits office address:
  POST /api/v1/office-address-details
    companyId
    buildingName / floorUnitNumber
    addressLine1 / addressLine2
    plotNumber
    effectiveFromDate
    officePhoneNumber / officeEmailAddress
    leaseAgreementDocumentPath   (uploaded document)
    occupancyCertificatePath     (uploaded document)
    status = draft → submitted

Admin reviews:
  PATCH /api/v1/office-address-details/:id/approve
    status          = approved
    reviewedById / reviewedByName / reviewedAt

  PATCH /api/v1/office-address-details/:id/reject
    status          = rejected
    remarks         = reason for rejection
```

**Office Address Status Flow:**
```
draft → submitted → approved ✓
                  → rejected → reapply → submitted → ...
```

#### 7.4 Auditor Assignment

```
Admin Frontend → Company → Auditors tab

Admin assigns an auditor from the Auditor master list:
  POST /api/v1/company-auditor
    companyId
    auditorId         (FK → auditors master table)
    contactPersonName
    appointmentDate
    documents         (appointment letter, etc.)

CompanyEvent created:
  type = auditor_assignment
  dueDate set for compliance tracking
```

#### 7.5 Shareholders & Share Capital

```
Added via form field mappings (Phase 4) OR
directly via company dashboard:

CompanyShareholder:
  companyId, name, nationality, address
  documentNumber, sharesNumber, shareType
  currency, occupation, email, contactNumber
  dateOfBirth, sharesHeldTrust, nameOfTrust

CompanyCapital:
  companyId
  issuedAmount / issuedCurrency / issuedShareType / issuedNoOfShares
  paidUpAmount / paidUpCurrency / paidUpShareType / paidUpNoOfShares
```

#### 7.6 Company Charges

```
Registered charges against company assets:
  POST /api/v1/company-charge
    companyId
    dateRegistered
    currency / amountSecured
    chargee (name of the charge holder)
```

#### 7.7 Company Guarantors (for CLG companies)

```
For Public Companies Limited by Guarantee:
  POST /api/v1/company-guarantor
    companyId, name, nationality, address
    documentNumber, amountGuaranteed, positionHeld
```

#### 7.8 Annual Compliance Events

CompanyEvents are automatically tracked for upcoming deadlines:

| Event Type | Description | Reminders |
|---|---|---|
| `director_assignment` | Director appointment due | 1 month + 1 week before |
| `secretary_assignment` | Secretary appointment due | 1 month + 1 week before |
| `auditor_assignment` | Auditor appointment due | 1 month + 1 week before |

AGM and Annual Return dates are stored directly on the `Company` entity (`lastAgmDate`, `lastArDate`, `financialYearEnd`).

#### 7.9 Certificate Confirmation

After the Certificate of Incorporation is emailed, the applicant must confirm receipt:

```
Client portal → Certificate confirmation screen
  PATCH /api/v1/company/:id/confirm-certificate
    certificateConfirmed          = true
    certificateConfirmationDate   = now()
    certificateConfirmationIp     = client IP

Triggers CERTIFICATE_CONFIRMATION notification to admin
  metadata: { confirmed, feedback, confirmationDate, clientIp }
```

---

### Status State Machines

#### EOI Status

```
                      ┌─────────────┐
                      │  SUBMITTED  │◄──────── public / logged-in user submits
                      └──────┬──────┘
                             │ admin opens EOI
                             ▼
                      ┌─────────────┐
                      │  ACCESSED   │
                      └──────┬──────┘
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌────────────┐
        │ APPROVED │  │ REJECTED │  │  REAPPLY   │ ← staff requests reapply
        └──────────┘  └────┬─────┘  └─────┬──────┘
                           │               │ applicant resubmits
                           │               ▼
                           │         ┌────────────┐
                           │         │RESUBMITTED │
                           │         └─────┬──────┘
                           │               │ admin reviews again
                           │               ▼
                           │          ACCESSED ...
                           │
                      (rejection email sent to applicant)
```

#### FormResponse Status

```
  DRAFT ──────► SUBMITTED ──────► ACTION_TO_BE_TAKEN
                                        │
                         ┌──────────────┼──────────────┐
                         │              │              │
                         ▼              ▼              ▼
                    ASSESSED     PRELIM_REJECTED   PRELIM_APPROVED
                         │              │              │
                         │     (resubmit)│              ▼
                         │              │        SENT_FOR_APPROVAL
                         │              │              │
                         │              └──────────────┤
                         │                             │
                         └────────────────────────────►▼
                                                   APPROVED ✓
                                          (triggers field mapping)
```

#### Company Status (`status` field)

```
  pending ──────► approved ──────► issued ✓
```

#### Director / Secretary Status

```
  pending ──────► approved ✓
              └── rejected  (can re-submit)
```

#### Office Address Status

```
  draft ──► submitted ──► approved ✓
                      └── rejected ──► reapply ──► submitted ──► ...
```

---

## 10. Form Engine — Deep Dive

The form engine is the core mechanism that bridges the EOI/application process with actual company data. It is entirely configuration-driven — no code changes are needed to modify forms, add fields, or change what data goes where.

### Data Model

```
FormTemplate
  └── FormSection[]
        └── FormField[]           (direct fields in the section)
        │     └── FormFieldCondition[]  (show/hide rules)
        │
        └── FormRepetableGroup[]  (groups of repeating fields)
              └── FormField[]     (fields inside the repeatable group)

FormFieldMapping[]                (separate — maps fields to DB columns)

─── Runtime (per applicant) ───

FormResponse
  ├── FormResponseAnswer[]        (one per FormField)
  └── FormResponseRepetableGroup[] (one per instance of a repeatable group)
        └── FormResponseAnswer[]  (answers within that instance)
```

### Field Types

Each `FormField` has a `type` enum (`DataType`):

| Type | Value | Description | Stored In |
|------|-------|-------------|-----------|
| `short-answer` | `SHORTANSWER` | Single-line text input | `singleValue` |
| `paragraph` | `PARAGRAPH` | Multi-line text area | `singleValue` |
| `multiple-choice` | `MULTIPLECHOICE` | Radio button (single select) | `singleValue` |
| `checkboxes` | `CHECKBOXES` | Tick boxes (multi-select) | `arrayValues` |
| `dropdown` | `DROPDWON` | Select dropdown | `singleValue` |
| `file-upload` | `FILEUPLOAD` | Single or multiple file upload | `fileUrl` / `fileUrls` |
| `date` | `DATE` | Date picker | `singleValue` |
| `time` | `TIME` | Time picker | `singleValue` |

**Additional field properties:**

| Property | Description |
|----------|-------------|
| `label` | Display question text |
| `placeholder` | Input hint text |
| `isRequired` | Validation — must be answered |
| `options` | JSON array of choices (for dropdown / multiple-choice / checkboxes) |
| `order` | Display ordering within section |
| `cspOnly` | If true, only visible when applicant type is CSP |
| `isActive` | Whether field is enabled |
| `softDeleted` | Soft-delete flag |

### Conditional Logic

`FormFieldCondition` controls whether a field is shown or hidden based on the answer to another field.

```
FormFieldCondition
  targetFieldId     → the field that will be shown/hidden
  dependsOnFieldId  → the field whose answer is checked
  operator          → comparison operator (e.g., "equals", "not_equals", "contains")
  value             → the value to compare against
  isActive          → toggle the condition on/off
```

**Example:**
```
Show field "Employment Pass Number" (targetField)
  WHEN field "Is Resident Director?" (dependsOnField)
  operator = "equals"
  value    = "true"
```

The **client frontend** evaluates conditions in real-time as the user fills the form. Hidden fields are excluded from submission or stored with null values.

### Repeatable Groups

`FormRepetableGroup` is a named group of fields inside a section that can be filled multiple times (e.g., "Add another director").

```
FormRepetableGroup
  sectionId  → which section it belongs to
  title      → group label (e.g., "Director Details")
  isRequired → at least one instance must be filled
  order      → display ordering
  fields[]   → the FormField definitions inside this group

FormResponseRepetableGroup  (runtime — one record per instance filled)
  formResponseId → which response
  groupId        → which template group definition
  answers[]      → FormResponseAnswer records for this instance
```

**Example:** A "Directors" repeatable group with fields `fullName`, `nationality`, `passportNo` can be filled 3 times = 3 directors, each creating a `FormResponseRepetableGroup` row.

### Field Mappings

`FormFieldMapping` is the critical bridge between form answers and the database. When a `FormResponse` is **finally approved**, the system reads these mappings and writes the answers to the appropriate entity tables.

```
FormFieldMapping
  templateId          → which FormTemplate this mapping belongs to
  fieldId             → the FormField whose answer is used
  entityTable         → target DB table (e.g., "companies", "directors")
  entityColumn        → target column in that table (e.g., "company_name")
  mappingRelation     → ONE_TO_ONE or ONE_TO_MANY
  groupKey            → links mapping to a repeatable group (for ONE_TO_MANY)
  conditionalFieldId  → only apply mapping if this field has a specific value
  conditionValue      → the value to check for the conditional mapping
```

**Mapping Relations:**

| Relation | When to use | Example |
|---|---|---|
| `one_to_one` | Answer maps to a single column in one row | `companies.company_name` ← field answer |
| `one_to_many` | Answer is inside a repeatable group → creates multiple rows | Each director group instance → one row in `directors` table |

**How the mapping execution works on approval:**

```
for each FormFieldMapping of this template:
  1. Find the FormResponseAnswer for fieldId
  2. If mappingRelation = ONE_TO_ONE:
       UPDATE {entityTable} SET {entityColumn} = answer
       WHERE company_id = this company's id
  3. If mappingRelation = ONE_TO_MANY:
       For each FormResponseRepetableGroup instance (matching groupKey):
         INSERT INTO {entityTable} ({entityColumn}, company_id)
         VALUES (answer_for_this_instance, company_id)
  4. If conditionalFieldId is set:
       Only apply if the answer to conditionalFieldId equals conditionValue
```

### Form Lifecycle

```
Admin creates FormTemplate (unpublished)
        │
        ├── Adds Sections
        ├── Adds Fields per section (types, labels, options, order)
        ├── Adds Repeatable Groups with their fields
        ├── Adds Conditions (show/hide rules)
        └── Adds Field Mappings (field → DB column)
                │
                ▼
         PUBLISH FormTemplate
         (isPublished = true, publishedAt = now)
                │
                ▼
         Template is now available to the EntityType it belongs to
                │
                ▼
         Applicant (with approved EOI for that EntityType)
         fills the form in the client portal
                │
                ├── Saves as DRAFT (auto-save)
                └── Submits → FormResponse.status = submitted
                                     │
                                     ▼
                              Admin review cycle
                              (see Phase 4 above)
                                     │
                                     ▼
                              APPROVED → field mappings execute
                                     │
                                     ▼
                              Company entity populated ✓
```

### Complete Example

Given a "Private Limited Company" form template with:

**Section: Company Details**
- Field A: "Company Name" (short-answer) → Mapping: `companies.company_name`, ONE_TO_ONE
- Field B: "Business Activities" (paragraph) → Mapping: `companies.business_activities`, ONE_TO_ONE
- Field C: "Registered Office Address" (paragraph) → Mapping: `companies.registered_office_address`, ONE_TO_ONE

**Section: Directors** _(with Repeatable Group "Director Details")_
- Field D: "Director Full Name" (short-answer) → Mapping: `directors.full_name`, ONE_TO_MANY, groupKey: `directors`
- Field E: "Director Nationality" (short-answer) → Mapping: `directors.nationality`, ONE_TO_MANY, groupKey: `directors`
- Field F: "Is Resident Director?" (multiple-choice) → Mapping: `directors.is_resident_director`, ONE_TO_MANY
- Field G: "Employment Pass Number" (short-answer)  
  - Condition: show only IF Field F = "Yes"  
  - Mapping: `directors.employment_pass`, ONE_TO_MANY

**On approval with 2 director instances filled:**
```
→ companies.company_name           = "Acme Ltd"
→ companies.business_activities    = "Software development..."
→ companies.registered_office_address = "123 Main St..."
→ INSERT directors (full_name="Alice", nationality="Bhutanese", is_resident_director=true, employment_pass="EP123", company_id=X)
→ INSERT directors (full_name="Bob",   nationality="Indian",    is_resident_director=false, company_id=X)
```

---

## 11. Master Data Reference

Master data are configuration records managed by admins that drive the platform's behaviour. They do not change frequently and serve as lookup / reference tables.

### 11.1 Entity Types (`entity_types`)

Defines the legal entity categories available for registration.

| Column | Description |
|--------|-------------|
| `name` | Display name (e.g., "Private Company Limited by Shares") |
| `description` | Short description |
| `slug` | URL-safe identifier |
| `createdByName` / `createdById` | Audit trail |

**Known entity types (from CompanyType enum):**

| Slug / CompanyType | Full Name |
|---|---|
| `PRIVATE_COMPANY_LIMITED_BY_SHARES` | Private Company Limited by Shares |
| `CSP_PRIVATE_COMPANY_LIMITED_BY_SHARES` | CSP — Private Company Limited by Shares |
| `PUBLIC_COMPANY_LIMITED_BY_GUARANTEE` | Public Company Limited by Guarantee |
| `RE_DOMICILED_COMPANIES` | Re-domiciled Companies |
| `FOREIGN_OR_BRANCH_COMPANIES` | Foreign or Branch Companies |
| `SOLE_PROPRIETORSHIP` | Sole Proprietorship |

Each `EntityType` links to:
- One `FormTemplate` (the application form)
- One or more `CertificateTemplates` (the incorporation certificate design)

### 11.2 Entity Type Prefixes (`entity_type_prefixes`)

Maps entity types to their UEN (Unique Entity Number) registration number prefixes.

```
entityTypeId → prefix (e.g., "PLB", "CLG", "REG")
```

Used when assigning a UEN during incorporation.

### 11.3 Certificate Templates (`certificate_templates`)

Stores the HTML/text template for each certificate type.

| Column | Description |
|--------|-------------|
| `entityTypeId` | Which entity type this certificate is for |
| `certificateType` | `COI` / `CRBN` / `CCRT` / `CRFC` (see below) |
| `title` | Certificate title |
| `templateBody` | HTML/text body with placeholders |
| `version` | Version number for change tracking |

**Certificate Types:**

| Code | Full Name |
|------|-----------|
| `COI` | Certificate of Incorporation |
| `CRBN` | Certificate Confirming Registration of Business Name |
| `CCRT` | Certificate Confirming Registration by Transfer of Company |
| `CRFC` | Certificate Confirming Registration of Foreign Company |

### 11.4 Form Templates (`form_templates`)

Managed by admin via the Form Builder. See [Section 10](#10-form-engine--deep-dive) for the full structure.

| Column | Description |
|--------|-------------|
| `name` | Form name |
| `description` | Purpose of the form |
| `entityTypeId` | Which entity type uses this form |
| `isActive` | Toggle without deleting |
| `isPublished` | If true, applicants can use this form |
| `publishedAt` | When it was published |
| `softDeleted` | Soft-delete flag |

> Only **one published, active form template per entity type** should be active at a time. Publishing a new version does not auto-retire the old one.

### 11.5 Payment Masters (`payment_masters`)

Defines the fee schedule for each service type.

| Column | Description |
|--------|-------------|
| `name` | Service name (e.g., "Company Registration Fee") |
| `amount` | Fee amount |
| `currency` | Currency code (`USD` by default, from `Currency` enum) |

Managed by admin. Linked by `serviceType` in the `Payment` entity.

### 11.6 Payment Methods (`payment_methods`)

Defines the available payment channels.

| Column | Description |
|--------|-------------|
| `name` | Display name (e.g., "Bank Transfer via RMA") |
| `code` | Unique code (e.g., `RMA_BFS`, `STRIPE_CARD`) |
| `description` | How to use this method |
| `currency` | Supported currency |
| `transactionFee` | Fee charged per transaction |
| `status` | Active (1) / Inactive (0) |

### 11.7 Auditors (`auditors`)

Registry of approved auditing firms available for appointment to companies.

| Column | Description |
|--------|-------------|
| `empanelmentNo` | Official registration number |
| `name` | Firm name |
| `address` | Office address |
| `noOfStaff` | Number of staff |
| `partners` | JSON array of partner names |
| `email` | JSON array of contact emails |
| `phone` | JSON array of contact phones |
| `firmProfile` | Document path / URL |
| `country` | Country of registration |

### 11.8 Roles & Permissions (Admin Auth)

Managed in the `admin-auth` service. Drives access control for the admin portal.

**System Roles (examples):**

| Role Name | Purpose |
|-----------|---------|
| Intermediate Officer | First-line EOI review |
| Processing Officer | Processes form responses and incorporation |
| Registrar | Final approval authority |
| CSP Officer | Manages CSP applications |
| Super Admin | Full access |

**Permission Slugs (examples):**

| Slug | Description |
|------|-------------|
| `eoi.view` | View EOI list |
| `eoi.approve` | Approve an EOI |
| `eoi.reject` | Reject an EOI |
| `eoi.issue-credentials` | Issue login credentials |
| `companies.view` | View company list |
| `companies.manage` | Edit company details |
| `form-templates.publish` | Publish a form template |
| `payment-master.manage` | Edit fee schedule |
| `auditors.manage` | Manage auditor registry |

### 11.9 FAQs (`faqs`)

Static FAQ content managed by admin, displayed on the public-facing client portal.

### 11.10 Professional Services (`professional_services`)

Reference list of professional service categories that CSPs can offer (used during CSP EOI collection).

### 11.11 Guidelines (`guidelines` — Admin Auth / Small Business Auth)

In-app guidance content displayed to staff in the admin portal. Each guideline has a `title`, `content`, and `slug` for routing.

---

## 12. Technology Stack Summary

| Layer | Technology |
|-------|-----------|
| Frontend Framework | Next.js 14+ (App Router) |
| Frontend State | Redux Toolkit |
| Frontend UI | Radix UI + Tailwind CSS + shadcn/ui |
| Frontend Forms | React Hook Form + Zod |
| Backend Framework | NestJS |
| Language | TypeScript (ESM, Node ≥ 22) |
| ORM | TypeORM |
| Database | PostgreSQL |
| Auth | JWT (RS256 / HS256) + 2FA (TOTP) |
| Messaging | NATS (microservices) |
| File Storage | AWS S3 (configured, service abstracted) |
| Payment Gateways | RMA BFS Protocol, Stripe |
| Containerisation | Docker + Docker Compose |
| API Docs | Swagger / OpenAPI (auto-generated per service) |
| Package Manager | Yarn (backends), pnpm (frontends) |
| Runtime | Node.js ≥ 22 |

---

## 13. Environment & Deployment

Each service is independently containerised. A root-level `docker-compose.yml` per service allows local development. Production uses Kubernetes (`prod-deployment.yaml` in `client/`).

### Port Convention

| Service | Default Port |
|---------|-------------|
| Client Frontend | 3000 |
| Admin Frontend | 3001 |
| Corporate Registry API | 3003 |
| Admin Auth API | 3004 |
| Small Business API | 3005 |
| Payment Service | 3006 |

### Start Commands

```bash
# Corporate Registry API
cd coporate-registry
yarn start:dev

# Admin Auth API
cd admin-auth
yarn start:dev

# Small Business API
cd small-business
yarn start:dev

# Payment Service
cd payment
yarn start:dev

# Client Frontend
cd client
pnpm dev

# Admin Frontend
cd admin
pnpm dev
```

### Database Migrations

All backends use TypeORM migrations:

```bash
# Generate a migration (run from the service directory)
yarn migration:generate src/database/migrations/MigrationName

# Revert last migration
yarn migration:revert
```

### Swagger API Docs

Each backend exposes Swagger at `/documentation` when running in non-production mode.

---

*This document covers the system as of May 2026. Update it whenever new modules are added or cross-service contracts change.*

# 🚀 SAP RAP Managed Scenario — Step-by-Step Guide

![SAP](https://img.shields.io/badge/SAP-0FAAFF?style=for-the-badge&logo=sap&logoColor=white)
![ABAP](https://img.shields.io/badge/ABAP-Cloud-blue?style=for-the-badge)
![RAP](https://img.shields.io/badge/RAP-Managed-green?style=for-the-badge)
![BTP](https://img.shields.io/badge/BTP-Trial-orange?style=for-the-badge)

> A complete, beginner-friendly implementation of an SAP ABAP RESTful Application Programming Model (RAP) **Managed Business Object** — built on SAP BTP ABAP Environment Trial. This repository walks through every layer of the RAP stack from database tables to the Fiori Elements UI, with detailed explanations of *why* each artifact exists, not just *how* to create it.

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Prerequisites](#-prerequisites)
- [Step 1 — Database Table](#-step-1--database-table-main-table)
- [Step 2 — Draft Table](#-step-2--draft-table-why-it-exists)
- [Step 3 — Interface View (Base BO)](#-step-3--interface-view-zr_)
- [Step 4 — Consumption View (Projection)](#-step-4--consumption-view-zc_)
- [Step 5 — Metadata Extension](#-step-5--metadata-extension)
- [Step 6 — Behavior Definition (Interface)](#-step-6--behavior-definition-interface-view)
- [Step 7 — Behavior Definition (Projection)](#-step-7--behavior-definition-projection-view)
- [Step 8 — Behavior Implementation](#-step-8--behavior-implementation-bap)
- [Step 9 — Service Definition & Binding](#-step-9--service-definition--service-binding)
- [Key Takeaways](#-key-takeaways)

---

## 🌐 Overview

### What is RAP?

The **ABAP RESTful Application Programming Model (RAP)** is SAP's modern, cloud-ready framework for building transactional Fiori applications in ABAP Cloud. It replaces older approaches like BOPF and classic ABAP MVC, and is the **standard for SAP S/4HANA Cloud and BTP ABAP Environment** development.

RAP provides:
- A structured, layered architecture from database to UI
- Built-in OData service generation (no manual SEGW needed)
- Framework-managed CRUD operations (Create, Read, Update, Delete)
- Draft handling for save/resume workflows
- Clean Core compliance — no modifications to SAP standard objects

### What is the Managed Scenario?

In RAP, there are two implementation types:

| Type | Who handles CRUD? | When to use |
|---|---|---|
| **Managed** | RAP Framework automatically | New custom BOs — no legacy dependencies |
| **Unmanaged** | Developer writes all logic | Wrapping legacy function modules / BAPIs |

In the **Managed scenario**, the RAP framework automatically handles database `INSERT`, `UPDATE`, `DELETE`, and `SELECT` operations. You only write code for custom business logic like validations, determinations, and custom actions. This makes it the fastest and cleanest way to build a new Business Object from scratch.

---

## 🏗️ Architecture

The RAP stack follows a strict layered architecture. Each layer has a specific responsibility, and no layer should skip or bypass another.

```
┌─────────────────────────────────────────────────────────┐
│                    Fiori Elements UI                    │
│              (Auto-generated from OData)                │
└────────────────────────┬────────────────────────────────┘
                         │  OData V4
┌────────────────────────▼────────────────────────────────┐
│              Service Definition + Service Binding        │
│         (Exposes which entities go into OData)          │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│              PROJECTION LAYER  (ZC_*)                   │
│   CDS Consumption View  +  Projection Behavior Def.     │
│   (Service-specific: UI annotations, field exposure)    │
└────────────────────────┬────────────────────────────────┘
                         │  as projection on
┌────────────────────────▼────────────────────────────────┐
│              INTERFACE LAYER  (ZR_* / ZI_*)             │
│   CDS Interface View  +  Base Behavior Definition       │
│   (Service-agnostic: data model, business logic)        │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                  DATABASE LAYER                         │
│        Main Table (ZDB_*)  +  Draft Table (*_D)         │
└─────────────────────────────────────────────────────────┘
```

---

## ✅ Prerequisites

Before you start, make sure you have:

- [ ] SAP BTP Trial account — [Get one free here](https://account.hanatrial.ondemand.com)
- [ ] Steps to create Trial account - [Steps](https://developers.sap.com/tutorials/hcp-create-trial-account)
- [ ] ABAP Environment trial instance provisioned on BTP [Steps](https://developers.sap.com/tutorials/abap-environment-trial-onboarding) 
- [ ] Eclipse IDE with **ABAP Development Tools (ADT)** plugin installed [Steps](https://developers.sap.com/tutorials/abap-install-adt)
- [ ] Connected to your BTP ABAP system via ADT
- [ ] Basic understanding of ABAP syntax and CDS views

---

## 🗄️ Step 1 — Database Table (Main Table)

### What is it?

The database table is the **foundation of your Business Object**. It is the actual physical table in the SAP HANA database where all your application data is permanently stored. Every field you want to persist — travel ID, agency, customer, price, status — must be defined here first.

### Why do we create it?

Without a database table, there is no place to store data. The entire RAP stack above it — CDS views, behavior definitions, OData services — are all abstractions built on top of this table. The table is the single source of truth for your BO's data.

### Structure of a RAP-compliant database table

A RAP managed table must follow specific conventions:

```abap
@EndUserText.label: 'Travel Data Table'
@AbapCatalog.enhancement.category: #NOT_EXTENSIBLE
@AbapCatalog.tableCategory: #TRANSPARENT
@AbapCatalog.deliveryClass: #A
@AbapCatalog.dataMaintenance: #RESTRICTED
define table ztravel_d {

  key client      : abap.clnt not null;   -- Always first, system-managed
  key travel_id   : /dmo/travel_id not null;  -- Your primary key

  -- Administrative fields (auto-managed by RAP framework)
  created_by      : abap_boolean;
  created_at      : abp_creation_tstmpl;
  last_changed_by : abp_lastchangedby;
  last_changed_at : abp_lastchangedtstmpl;
  local_last_changed_at : abp_locchangedtstmpl;  -- For ETag / concurrency

  -- Business fields
  agency_id       : /dmo/agency_id;
  customer_id     : /dmo/customer_id;
  begin_date      : /dmo/begin_date;
  end_date        : /dmo/end_date;
  description     : /dmo/description;
  total_price     : /dmo/total_price;
  currency_code   : /dmo/currency_code;
  status          : /dmo/overall_status;
}
```

### Key fields explained

| Field | Type | Purpose |
|---|---|---|
| `client` | `abap.clnt` | SAP client field — mandatory first key in every table |
| `travel_id` | Business key | Unique identifier for each travel record |
| `created_by / created_at` | Admin fields | Auto-populated by RAP framework — track who created the record |
| `last_changed_by / last_changed_at` | Admin fields | Auto-populated — track last modifier |
| `local_last_changed_at` | ETag field | Used by RAP for **optimistic locking / concurrency control** |

> **Important:** The `local_last_changed_at` field with type `abp_locchangedtstmpl` is critical for the ETag mechanism. When two users try to update the same record simultaneously, RAP compares this timestamp to detect conflicts and reject stale updates.

---

## 📝 Step 2 — Draft Table (Why It Exists)

### What is a Draft Table?

The draft table is a **temporary storage table** where incomplete, unsaved data is held while a user is still filling in a form. It has an identical structure to your main database table, with some additional draft-specific fields added by the framework.

By convention, draft tables are named the same as the main table with a `_D` suffix:

```
Main table   : ZTRAVEL        ← permanent, saved data
Draft table  : ZTRAVEL_D      ← temporary, in-progress data
```

### Why do we need a Draft Table?

Imagine a user opens a "Create Travel" form in a Fiori app and starts filling in the fields — agency, customer, dates. Halfway through, their browser crashes, or they need to attend a meeting. Without draft handling, all their work is lost.

**With Draft Handling:**

```
User opens form
      │
      ▼
Data saved to DRAFT TABLE immediately (auto-save)
      │
      ├── User continues tomorrow → draft is still there ✅
      ├── User switches device → draft is still there ✅
      └── User clicks "Save" → data moves to MAIN TABLE ✅
                               draft record is deleted ✅
```

The draft table ensures **no data loss** during long-running create/edit operations. It also enables the "Edit" button behavior in Fiori apps — when you click Edit on a saved record, RAP copies the record to the draft table so your edits don't affect the live data until you explicitly save.

### How to define the Draft Table

You don't write the draft table manually field by field. You reference it in your Behavior Definition and let ADT generate it:

```abap
-- In your Behavior Definition:
define behavior for ZR_TravelTP alias Travel
implementation in class ZBP_TravelTP unique

persistent table ztravel          -- ← your main table
draft table ztravel_d             -- ← your draft table (RAP manages this)

...
```

When you activate the behavior definition, ADT automatically creates the draft table with the correct structure including draft-specific fields like `draftentityoperationcode`, `draftadministrativedatauuid`, etc.

### Draft Table vs Main Table — Side by Side

| Aspect | Main Table | Draft Table |
|---|---|---|
| Purpose | Permanent business data | Temporary in-progress data |
| When data lands here | On "Save" action | On every field change (auto-save) |
| Who can see it | All users | Only the user editing |
| Deleted when | Never (business record) | On Save or Discard |
| Named | `ZTRAVEL` | `ZTRAVEL_D` |
| Created by | Developer manually | ADT generates from BDEF |

---

## 👁️ Step 3 — Interface View (ZR_*)

### What is the Interface View?

The Interface View is the **base CDS View Entity** that sits directly on top of your database table. It forms the **service-agnostic data model** of your Business Object — meaning it defines the complete data structure without any UI-specific concerns.

By convention, Interface Views are prefixed with `ZR_` (R = Root) for root entities or `ZI_` (I = Interface).

```abap
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Travel Interface View'
define root view entity ZR_TravelTP
  as select from ztravel
{
  key TravelID        as TravelID,
      AgencyID        as AgencyID,
      CustomerID      as CustomerID,
      BeginDate       as BeginDate,
      EndDate         as EndDate,
      Description     as Description,
      TotalPrice      as TotalPrice,
      CurrencyCode    as CurrencyCode,
      OverallStatus   as OverallStatus,

      -- Administrative fields (mapped for RAP framework)
      CreatedBy       as CreatedBy,
      CreatedAt       as CreatedAt,
      LastChangedBy   as LastChangedBy,
      LastChangedAt   as LastChangedAt,
      LocalLastChangedAt as LocalLastChangedAt,

      -- Associations (for navigation / joins)
      _Agency,
      _Customer
}
```

### Why do we need the Interface View?

The Interface View serves several critical purposes:

**1. Abstraction from the database table**
The database table uses technical field names and types. The Interface View can rename, cast, and join fields into a cleaner business model. If the table structure changes, only the interface view needs to be updated — everything above is unaffected.

**2. It is the root of the composition tree**
The keyword `define root view entity` declares this as the top of a Business Object hierarchy. All child entities (like Booking under Travel) are connected back to this root via `COMPOSITION OF` associations.

**3. It is reusable across multiple services**
The interface view has NO UI annotations, NO Fiori-specific logic. This means the same interface view can power:
- A Fiori Elements app (via projection view → OData)
- A remote API (via a different projection)
- A background job (direct consumption)

**4. Associations are defined here**
Associations to related entities (Agency, Customer) are defined on the interface view so they are available to all consumers.

### The `define root view entity` keyword

This keyword does two important things:
- It declares this CDS view as a **Root Entity** of a Business Object
- It enables it to have its own **Behavior Definition** repository object
- Only root entities can define `composition of` child associations

---

## 🔍 Step 4 — Consumption View (ZC_*)

### What is the Consumption View?

The Consumption View (also called the **Projection View**) sits on top of the Interface View and represents a **service-specific, tailored window** into the Business Object. It is what gets exposed to the outside world — via OData, via a Fiori app, via an API.

By convention, Consumption Views are prefixed with `ZC_` (C = Consumption).

```abap
@Metadata.allowExtensions: true      -- Allows metadata extensions to add UI annotations
@EndUserText.label: 'Travel Consumption View'
define root view entity ZC_TravelTP
  as projection on ZR_TravelTP       -- ← the key syntax: "as projection on"
{
  key TravelID,
      AgencyID,
      CustomerID,
      BeginDate,
      EndDate,
      Description,
      TotalPrice,
      CurrencyCode,
      OverallStatus,

      -- Expose only what this service needs
      CreatedBy,
      CreatedAt,
      LastChangedBy,
      LastChangedAt,
      LocalLastChangedAt,

      -- Redirect associations to their own projection views
      _Agency : redirected to ZC_AgencyTP,
      _Customer : redirected to ZC_CustomerTP
}
```

### Why do we need the Consumption View if we already have the Interface View?

This is the most common question from beginners. The answer is **separation of concerns**:

| Concern | Interface View | Consumption View |
|---|---|---|
| Business data model | ✅ Defined here | Inherited from interface |
| Associations & compositions | ✅ Defined here | Redirected here |
| UI annotations (`@UI.*`) | ❌ Not here | ✅ Via Metadata Extension |
| Value helps (`@Consumption.valueHelpDefinition`) | ❌ Not here | ✅ Defined here |
| Field aliasing for UI | ❌ Not here | ✅ Can rename fields |
| Access control | ✅ Defined here | Additional checks possible |
| Which fields to expose | All fields | **Subset — only what the app needs** |

**Real-world analogy:** Think of the Interface View as a **master employee record** with every piece of data. The Consumption View is a **role-specific form** — an HR manager sees salary and personal data, a project manager sees only skills and availability. Same underlying data, different projections.

### The `as projection on` syntax

This is what makes a view a projection view. It tells the RAP framework:
- "This view is consuming/projecting another view"
- "The underlying BO's behavior definition applies through here"
- "Compositions must be redirected to their own projection equivalents"

### Redirection of Associations

Notice `_Agency : redirected to ZC_AgencyTP`. This is mandatory for compositions in projection views. It ensures that when someone navigates from a Travel to its related Agency through OData, they navigate to the Agency's **projection** view (not its interface view), so the same service-specific rules apply all the way through the navigation.

---

## 🎨 Step 5 — Metadata Extension

### What is a Metadata Extension?

A Metadata Extension (`.ddlx` file) is a separate repository object that holds **UI annotations** for a CDS view, kept completely separate from the view's data model definition. It is applied on top of a Consumption View that has `@Metadata.allowExtensions: true`.

```abap
@Metadata.layer: #CORE
annotate view ZC_TravelTP with
{
  @UI.facet: [
    {
      id: 'Travel',
      purpose: #STANDARD,
      type: #IDENTIFICATION_REFERENCE,
      label: 'Travel Details',
      position: 10
    }
  ]

  @UI: {
    lineItem: [{ position: 10, importance: #HIGH }],
    identification: [{ position: 10 }]
  }
  TravelID;

  @UI: {
    lineItem: [{ position: 20, importance: #HIGH }],
    identification: [{ position: 20 }],
    selectionField: [{ position: 20 }]
  }
  AgencyID;

  @UI: {
    lineItem: [{ position: 30, importance: #HIGH }],
    identification: [{ position: 30 }]
  }
  CustomerID;

  @UI: {
    lineItem: [{ position: 70, importance: #HIGH,
      criticality: 'OverallStatusCriticality' }],
    identification: [{ position: 70 }]
  }
  OverallStatus;
}
```

### Why do we use a Metadata Extension instead of putting annotations directly in the CDS view?

There are three powerful reasons:

**1. Clean Separation of Concerns**
The CDS view defines WHAT data exists. The Metadata Extension defines HOW it looks in the UI. Mixing both in one file creates a massive, hard-to-read view definition. Separating them keeps each file focused and maintainable.

**2. Extensibility (Key User / Partner / Customer layers)**
The `@Metadata.layer` annotation supports a hierarchy:
```
#CORE          ← SAP / developer defines base UI
#CUSTOMER_BASE ← Customer adds their annotations
#PARTNER       ← Partner (SI) adds their annotations  
#CUSTOMER      ← End customer final customization
```
Lower layers can be overridden by higher layers without modifying the original files. This is the foundation of SAP's extensibility model.

**3. Keeps the CDS View Clean for Multiple Consumers**
If you put `@UI.lineItem` directly in the CDS view, those annotations are baked in for everyone. With a metadata extension, different services can have different metadata extensions on the same projection view — different UI layouts for different apps.

### Common UI Annotations in Metadata Extensions

| Annotation | Purpose | Example |
|---|---|---|
| `@UI.lineItem` | Show field in the list table | List page columns |
| `@UI.identification` | Show field in the detail form | Object page fields |
| `@UI.selectionField` | Show field in the filter bar | Search filters |
| `@UI.facet` | Define sections/tabs on the object page | "Travel Details" tab |
| `@UI.hidden` | Hide field from UI entirely | Internal fields |
| `@UI.dataAction` | Add a button for a custom action | "Accept Travel" button |

---

## ⚙️ Step 6 — Behavior Definition (Interface View)

### What is a Behavior Definition?

A Behavior Definition (BDEF) is a repository object that defines **what operations are allowed** on a Business Object. It answers questions like:
- Can this BO be created, updated, deleted?
- Which fields are read-only or mandatory?
- Does it support draft handling?
- What custom actions exist (like "Accept Travel")?
- How are primary keys generated?

### Interface View Behavior Definition

The base BDEF sits on the Interface View and defines the **complete set of capabilities** of the BO:

```abap
managed implementation in class ZBP_TravelTP unique;
strict ( 2 );
with draft;

define behavior for ZR_TravelTP alias Travel
implementation in class ZBP_TravelTP unique

persistent table ztravel
draft table ztravel_d

etag master LocalLastChangedAt          -- ETag for concurrency control
lock master total etag LastChangedAt    -- Locking mechanism

authorization master( global )

{
  -- Standard CRUD operations (framework handles automatically in managed)
  create;
  update;
  delete;

  -- Draft actions (framework-generated)
  draft action Edit;
  draft action Activate;
  draft action Discard;
  draft action Resume;
  draft determine action Prepare;

  -- Field controls
  field ( readonly ) TravelID;                       -- Auto-generated, never editable
  field ( readonly : update ) AgencyID, CustomerID;  -- Set on create, locked on update
  field ( mandatory ) BeginDate, EndDate, Status;    -- Required fields

  -- Auto-numbering (framework generates UUID)
  mapping for ztravel corresponding
  {
    TravelID = travel_id;
    AgencyID = agency_id;
    ...
  }

  -- Custom validations (developer implements)
  validation validateDates   on save { field BeginDate, EndDate; }
  validation validateStatus  on save { field OverallStatus; }

  -- Custom determinations (developer implements)
  determination setInitialStatus on modify { create; }

  -- Custom actions (developer implements)
  action ( features : instance ) acceptTravel result [1] $self;
  action ( features : instance ) rejectTravel result [1] $self;

  -- Associations
  association _Booking { create; with draft; }
}
```

### Key BDEF keywords explained

**`managed`** — Tells the RAP framework to auto-handle all database operations (INSERT, UPDATE, DELETE). You don't write any `MODIFY dbtab` statements yourself.

**`with draft`** — Activates the draft handling mechanism. The framework will use your `draft table` for all in-progress changes.

**`etag master LocalLastChangedAt`** — Enables optimistic locking. Before any update, RAP compares the `LocalLastChangedAt` timestamp in the request with the database to detect if someone else modified the record in between.

**`lock master`** — When a user starts editing (enters draft mode), the record is locked so no other user can simultaneously edit the same record.

**`numbering:managed`** — (when used on a field) Tells the framework to auto-generate a UUID primary key at CREATE time — no manual key assignment needed.

**`validation`** — A method you implement in the Behavior Implementation class. Called by the framework before save. Return error messages to block the save.

**`determination`** — A method that auto-runs when specific fields change. Used for defaulting values (e.g., set status to "Open" when a travel is created).

**`action`** — A custom operation beyond standard CRUD. Adds a button in the Fiori UI. Example: "Accept Travel" changes status to Accepted.

---

## 🔒 Step 7 — Behavior Definition (Projection View)

### What is the Projection Behavior Definition?

The Projection BDEF sits on the **Consumption View** and exposes a **subset** of the base BDEF's operations for a specific service. It acts as a gatekeeper — you decide which operations this particular OData service gets to offer.

```abap
projection;
strict ( 2 );
use draft;

define behavior for ZC_TravelTP alias Travel
{
  use create;    -- Expose create to this service
  use update;    -- Expose update to this service
  use delete;    -- Expose delete to this service

  use action Edit;
  use action Activate;
  use action Discard;
  use action Resume;
  use action Prepare;

  -- Expose only the custom actions this service needs
  use action acceptTravel;
  use action rejectTravel;

  -- Child entity
  use association _Booking { create; with draft; }
}
```

### Interface BDEF vs Projection BDEF — Why both?

| | Interface BDEF | Projection BDEF |
|---|---|---|
| Sits on | Interface View (ZR_*) | Consumption View (ZC_*) |
| Contains | ALL possible operations | SUBSET for this service |
| Implementation | Developer writes code here | Just `use` statements — no code |
| Reusability | One per BO | One per service/app |
| Analogy | Full employee record | Role-specific access form |

Imagine you have two apps using the same Travel BO:
- **Admin App** → projection BDEF exposes: create, update, delete, acceptTravel, rejectTravel
- **Read-Only Report App** → projection BDEF exposes: nothing (read-only by omission)

Same base BO, completely different service behavior — controlled purely by the projection BDEF.

---

## 💻 Step 8 — Behavior Implementation (BAP)

### What is the Behavior Implementation?

The Behavior Implementation is an **ABAP class** that contains the actual code for all custom operations defined in the base Behavior Definition — validations, determinations, and actions. In managed scenarios, you do NOT write code for standard CRUD — the framework handles that. You only write code for your custom business logic.

The class is named in the BDEF with `implementation in class ZBP_TravelTP unique`.

```abap
CLASS zbp_traveltp DEFINITION
  PUBLIC
  ABSTRACT
  FINAL
  FOR BEHAVIOR OF zr_traveltp.
ENDCLASS.

CLASS zbp_traveltp IMPLEMENTATION.
ENDCLASS.
```

The actual logic sits in **local classes** inside the implementation class's `.locals_imp` include:

### Validation Example — validateDates

```abap
METHOD validateDates.
  " Read the fields we need to validate
  READ ENTITIES OF zr_traveltp IN LOCAL MODE
    ENTITY Travel
      FIELDS ( BeginDate EndDate )
      WITH CORRESPONDING #( keys )
    RESULT DATA(lt_travel)
    FAILED failed.

  LOOP AT lt_travel INTO DATA(ls_travel).

    " Validate begin date is not in the past
    IF ls_travel-BeginDate < cl_abap_context_info=>get_system_date( ).
      APPEND VALUE #(
        %tky = ls_travel-%tky
      ) TO failed-travel.

      APPEND VALUE #(
        %tky        = ls_travel-%tky
        %msg        = new_message_with_text(
                        severity = if_abap_behv_message=>severity-error
                        text     = 'Begin date cannot be in the past' )
        %element-BeginDate = if_abap_behv=>mk-on
      ) TO reported-travel.
    ENDIF.

    " Validate end date is after begin date
    IF ls_travel-EndDate < ls_travel-BeginDate.
      APPEND VALUE #(
        %tky = ls_travel-%tky
      ) TO failed-travel.

      APPEND VALUE #(
        %tky        = ls_travel-%tky
        %msg        = new_message_with_text(
                        severity = if_abap_behv_message=>severity-error
                        text     = 'End date must be after begin date' )
        %element-EndDate = if_abap_behv=>mk-on
      ) TO reported-travel.
    ENDIF.
  ENDLOOP.
ENDMETHOD.
```

### Determination Example — setInitialStatus

```abap
METHOD setInitialStatus.
  " Read travels that were just created
  READ ENTITIES OF zr_traveltp IN LOCAL MODE
    ENTITY Travel
      FIELDS ( OverallStatus )
      WITH CORRESPONDING #( keys )
    RESULT DATA(lt_travel)
    FAILED failed.

  " Filter: only set status if it's initial (empty)
  DELETE lt_travel WHERE OverallStatus IS NOT INITIAL.
  CHECK lt_travel IS NOT INITIAL.

  " Set default status to 'Open'
  MODIFY ENTITIES OF zr_traveltp IN LOCAL MODE
    ENTITY Travel
      UPDATE FIELDS ( OverallStatus )
      WITH VALUE #( FOR ls_travel IN lt_travel (
        %tky          = ls_travel-%tky
        OverallStatus = 'O'   " O = Open
      ) )
    REPORTED DATA(lt_reported).

  reported = CORRESPONDING #( DEEP lt_reported ).
ENDMETHOD.
```

### Action Example — acceptTravel

```abap
METHOD acceptTravel.
  " Change status to 'Accepted'
  MODIFY ENTITIES OF zr_traveltp IN LOCAL MODE
    ENTITY Travel
      UPDATE FIELDS ( OverallStatus )
      WITH VALUE #( FOR key IN keys (
        %tky          = key-%tky
        OverallStatus = 'A'   " A = Accepted
      ) )
    FAILED   failed
    REPORTED reported.

  " Read back the updated records to return as result
  READ ENTITIES OF zr_traveltp IN LOCAL MODE
    ENTITY Travel ALL FIELDS WITH CORRESPONDING #( keys )
    RESULT DATA(lt_travel).

  result = VALUE #( FOR ls_travel IN lt_travel (
    %tky   = ls_travel-%tky
    %param = ls_travel
  ) ).
ENDMETHOD.
```

### Important RAP-specific ABAP concepts

**`READ ENTITIES`** — The RAP-specific way to read data from a BO. Always use this inside behavior implementations — never use `SELECT` directly.

**`MODIFY ENTITIES`** — The RAP-specific way to change data inside behavior implementations — never use `MODIFY dbtab` directly.

**`IN LOCAL MODE`** — Bypasses authorization checks and feature controls when called from within the same BO's implementation. Important for internal BO operations.

**`%tky`** — The transactional key — a structure containing all key fields of the entity. Used to identify records throughout RAP's internal processing.

**`failed / reported`** — Standard output parameters for all RAP methods. Populate `failed` to signal which records errored. Populate `reported` to send messages back to the UI.

---

## 🌐 Step 9 — Service Definition & Service Binding

### Service Definition

Defines which CDS entities are exposed and under what names:

```abap
@EndUserText.label: 'Travel OData Service'
define service ZUI_TravelTP {
  expose ZC_TravelTP    as Travel;
  expose ZC_BookingTP   as Booking;
  expose ZC_AgencyTP    as Agency;
  expose ZC_CustomerTP  as Customer;
}
```

### Service Binding

The Service Binding ties a Service Definition to a specific **protocol and usage type**. For Fiori apps, you create an OData V4 binding with type `UI`:

- **Binding Type:** `OData V4 - UI`
- **Activate** the binding to get the OData service URL
- Use **"Preview"** in ADT to open the Fiori Elements app directly

Once activated, your Travel BO is fully functional as a Fiori Elements List Report + Object Page app — with list, filter, create, edit, draft, save, and all your custom actions.

---

## 💡 Key Takeaways

### The Purpose of Each Layer — Summary

```
DATABASE TABLE     → Where data lives permanently
DRAFT TABLE        → Where in-progress data lives temporarily
INTERFACE VIEW     → Clean data model, service-agnostic, reusable
CONSUMPTION VIEW   → Service-specific window — what this app sees
METADATA EXTENSION → UI layout — how fields look in Fiori
BASE BDEF          → All possible BO operations defined
PROJECTION BDEF    → Subset of operations for this specific service
BAP CLASS          → Custom business logic — validations, actions
SERVICE DEFINITION → Which entities go into the OData service
SERVICE BINDING    → Which protocol (OData V4) and port (UI/API)
```

### The Two-Layer Pattern (Interface + Consumption) — Why it matters

The separation of Interface and Consumption layers is the most important design decision in RAP. It gives you:

1. **Reusability** — one BO, multiple services (UI service, API service, integration service)
2. **Flexibility** — each service exposes exactly what it needs
3. **Stability** — changing the UI doesn't break the underlying BO contract
4. **Extensibility** — SAP's own S/4HANA BOs follow this same pattern

### Managed vs Unmanaged at a glance

In this managed scenario, the RAP framework wrote all the database interaction code for you. You only wrote:
- Validation logic (validateDates)
- Determination logic (setInitialStatus)
- Action logic (acceptTravel, rejectTravel)

Everything else — INSERT, UPDATE, DELETE, draft handling, locking, ETag — was handled by the framework automatically.

---

## 📚 Further Reading

- [SAP Help: ABAP RESTful Application Programming Model](https://help.sap.com/docs/abap-cloud/abap-rap/abap-restful-application-programming-model)
- [SAP Help: Business Object Projection](https://help.sap.com/docs/abap-cloud/abap-rap/business-object-projection)
- [SAP Community: RAP Recommendations](https://community.sap.com/t5/technology-blog-posts-by-sap/rap-recommendations/ba-p/14382633)
- [SAP Samples: RAP100 Hands-on](https://github.com/SAP-samples/abap-platform-rap100)
- [openSAP: Building Apps with RAP](https://open.sap.com/courses/cp7)

---

## 🤝 Contributing

Found an error or want to improve this guide? Feel free to:
1. Fork this repository
2. Create a feature branch (`git checkout -b fix/correction`)
3. Commit your changes (`git commit -m 'Fix: corrected BDEF syntax'`)
4. Push to the branch (`git push origin fix/correction`)
5. Open a Pull Request

---

## 👤 Author

**Naveen Kumar**
SAP ABAP Developer | BTP Explorer
- SAP Community Profile: [community.sap.com](https://community.sap.com)
- GitHub: [github.com](https://github.com)

---

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

*If this repository helped you understand RAP, please give it a ⭐ — it helps others find it too!*

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
- [ ] ABAP Environment trial instance provisioned on BTP - [Steps](https://developers.sap.com/tutorials/abap-environment-trial-onboarding) 
- [ ] Eclipse IDE with **ABAP Development Tools (ADT)** plugin installed - [Steps](https://developers.sap.com/tutorials/abap-install-adt)
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
@EndUserText.label : 'Products'
@AbapCatalog.enhancement.category : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table znk_t_product {

  key client            : abap.clnt not null;
  key pro_uuid          : sysuuid_x16 not null;
  key product_id        : abap.char(10) not null;
  description           : abap.char(40);
  @Semantics.quantity.unitOfMeasure : 'znk_t_product.uom'
  qty                   : abap.quan(8,3);
  uom                   : abap.unit(3);
  @Semantics.amount.currencyCode : 'znk_t_product.currency'
  price                 : abap.curr(15,2);
  currency              : abap.cuky;
  mfg_date              : abap.datn;
  exp_date              : abap.datn;
  validdays             : abap.int4;
  plant                 : abap.char(4);
  local_created_by      : abp_creation_user;
  local_created_at      : abp_creation_tstmpl;
  local_last_changed_by : abp_locinst_lastchange_user;
  local_last_changed_at : abp_locinst_lastchange_tstmpl;
  last_changed_at       : abp_lastchange_tstmpl;

}
```

### Key fields explained

| Field | Type | Purpose |
|---|---|---|
| `client` | `abap.clnt` | SAP client field — mandatory first key in every table |
| `product_id` | Business key | Unique identifier for each product record |
| `local_created_byy / local_created_at` | Admin fields | Auto-populated by RAP framework — track who created the record |
| `local_last_changed_by / local_last_changed_at` | Admin fields | Auto-populated — track last modifier |
| `last_changed_at` | ETag field | Used by RAP for **optimistic locking / concurrency control** |

> **Important:** The `last_changed_at` field with type `abp_lastchange_tstmpl` is critical for the ETag mechanism. When two users try to update the same record simultaneously, RAP compares this timestamp to detect conflicts and reject stale updates.

---

## 📝 Step 2 — Draft Table (Why It Exists)

### What is a Draft Table?

The draft table is a **temporary storage table** where incomplete, unsaved data is held while a user is still filling in a form. It has an identical structure to your main database table, with some additional draft-specific fields added by the framework.

By convention, draft tables are named the same as the main table with a `_D` suffix:

```
Main table   : ZNK_T_PRODUCT        ← permanent, saved data
Draft table  : ZNK_T_PRODUCT_D      ← temporary, in-progress data
```

### Why do we need a Draft Table?

Imagine a user opens a "Create Product" form in a Fiori app and starts filling in the fields — Plant, UOM, MFG & Exp dates. Halfway through, their browser crashes, or they need to attend a meeting. Without draft handling, all their work is lost.

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
managed implementation in class zbp_nk_product_i unique;
strict ( 2 );
with draft;
define behavior for znk_product_I //alias <alias_name>
persistent table znk_t_product               -- ← your main table
draft table znk_t_product_d                  -- ← your draft table (RAP manages this)            
...
```

When you activate the behavior definition, ADT automatically creates the draft table with the correct structure.

### Draft Table vs Main Table — Side by Side

| Aspect | Main Table | Draft Table |
|---|---|---|
| Purpose | Permanent business data | Temporary in-progress data |
| When data lands here | On "Save" action | On every field change (auto-save) |
| Who can see it | All users | Only the user editing |
| Deleted when | Never (business record) | On Save or Discard |
| Named | `ZNK_T_PRODUCT` | `ZNK_T_PRODUCT_D` |
| Created by | Developer manually | ADT generates from BDEF |

---

## 👁️ Step 3 — Interface View (Z*_I)

### What is the Interface View?

The Interface View is the **base CDS View Entity** that sits directly on top of your database table. It forms the **service-agnostic data model** of your Business Object — meaning it defines the complete data structure without any UI-specific concerns.

By convention, Interface Views are prefixed with `ZR_` (R = Root) for root entities or `ZI_` (I = Interface).

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Product Interface View'
@Metadata.ignorePropagatedAnnotations: true
define root view entity znk_product_I as select from znk_t_product
association [0..1] to I_Currency as _Currency on $projection.Currency = _Currency.Currency
association [ 0..1] to I_UnitOfMeasure as _Uom on $projection.Uom = _Uom.UnitOfMeasure

{
    key pro_uuid as ProUuid,
    key product_id as ProductId,
    description as Description,
    @Semantics.quantity.unitOfMeasure: 'uom'
    qty as Qty,
    @Consumption.valueHelpDefinition: [{ 
    entity.name: 'I_UnitOfMeasure',
    entity.element: 'UnitOfMeasure',
    useForValidation: true
     }]
    uom as Uom,
    @Semantics.amount.currencyCode: 'currency'
    price as Price,
    @Consumption.valueHelpDefinition: [{ 
    entity.name: 'I_Currency',
    entity.element: 'Currency',
    useForValidation: true
     }]
    currency as Currency,
    mfg_date as MfgDate,
    exp_date as ExpDate,
    plant as Plant,
    validdays as ValidDays,
    @Semantics.user.createdBy: true
    local_created_by as LocalCreatedBy,
    @Semantics.systemDateTime.createdAt: true
    local_created_at as LocalCreatedAt,
    @Semantics.user.lastChangedBy: true
    local_last_changed_by as LocalLastChangedBy,
    @Semantics.systemDateTime.localInstanceLastChangedAt: true
    local_last_changed_at as LocalLastChangedAt,
    @Semantics.systemDateTime.lastChangedAt: true
    last_changed_at as LastChangedAt,
    _Uom as UomDetails,
    _Currency as CurrencyDetails
     // Make association public
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


**4. Associations are defined here**
Associations to related entities (Currency, UnitOfMeasure) are defined on the interface view so they are available to all Products.

### The `define root view entity` keyword

This keyword does two important things:
- It declares this CDS view as a **Root Entity** of a Business Object
- It enables it to have its own **Behavior Definition** repository object
- Only root entities can define `composition of` child associations

---

## 🔍 Step 4 — Consumption View (Z*_C)

### What is the Consumption View?

The Consumption View (also called the **Projection View**) sits on top of the Interface View and represents a **service-specific, tailored window** into the Business Object. It is what gets exposed to the outside world — via OData, via a Fiori app, via an API.

By convention, Consumption Views are prefixed or suffixed with `ZC_` (C = Consumption).

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Product Consumption View'
@Metadata.ignorePropagatedAnnotations: true
@Metadata.allowExtensions: true
define root view entity znk_product_c
  provider contract transactional_query as projection on znk_product_I                           -- ← the key syntax: "as projection on"
  association [1..1] to znk_product_I as _BaseEntity on $projection.ProUuid = _BaseEntity.ProUuid
                                                    and $projection.ProductId = _BaseEntity.ProductId
{
key ProUuid,
key ProductId,
Description,
@Semantics.quantity.unitOfMeasure: 'Uom'
Qty,
@Consumption.valueHelpDefinition:[{ entity:{ name: 'I_UnitOfMeasure',
                                             element: 'UnitOfMeasure'},
                                    useForValidation: true }]
Uom,
@Semantics.amount.currencyCode: 'Currency'
Price,
@Consumption.valueHelpDefinition: [{ entity:{ name: 'I_Currency',
                                     element: 'Currency' },
                                     useForValidation: true  }]
Currency,
MfgDate,
ExpDate,
Plant,
ValidDays,
LocalCreatedBy,
LocalCreatedAt,
LocalLastChangedBy,
LocalLastChangedAt,
LastChangedAt,
/* Associations */
UomDetails,
CurrencyDetails,
_BaseEntity
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

---

## 🎨 Step 5 — Metadata Extension

### What is a Metadata Extension?

A Metadata Extension (`.ddlx` file) is a separate repository object that holds **UI annotations** for a CDS view, kept completely separate from the view's data model definition. It is applied on top of a Consumption View that has `@Metadata.allowExtensions: true`.

```abap
@Metadata.layer: #CORE
@UI.headerInfo: { typeName: 'Product', typeNamePlural: 'Products' }
annotate entity znk_product_c with
{

@UI.facet: [{ id: 'Info',
              type: #IDENTIFICATION_REFERENCE,
              label: 'Product',
              position: 10}]
  @UI:{ lineItem: [{position: 10, label: 'Product ID', importance: #HIGH,hidden: true},
                   { type: #FOR_ACTION, dataAction: 'checkProductValidity', label: 'Check Validity', position: 10 ,emphasized: true},              --- Custom Button Added Here 
                   { type: #FOR_ACTION, dataAction: 'clearValidDays',       label: 'Clear',          position: 20 ,emphasized: true}],             --- Custom Button Added Here   
        identification: [{position: 10, label: 'Product ID', importance: #HIGH, hidden: true},
                         { type: #FOR_ACTION, dataAction: 'checkProductValidity', label: 'Check Validity', position: 10 }] }
  ProUuid;
  @UI:{ lineItem: [{position: 20, label: 'Product ID', importance: #HIGH}],
        identification: [{position: 20, label: 'Product ID', importance: #HIGH}],
        selectionField: [{ position: 10  }] }
  ProductId;
  @UI:{ lineItem: [{position: 30, label: 'Description', importance: #HIGH}],
        identification: [{position: 30, label: 'Description', importance: #HIGH}],
        selectionField: [{ position: 30  }] }
  Description;
   @UI:{ lineItem: [{position: 31, label: 'Qty', importance: #HIGH}],
        identification: [{position: 31, label: 'Quantity', importance: #HIGH}] }
  Qty;
  @UI:{ lineItem: [{position: 40, label: 'UOM', importance: #HIGH, hidden: true}],
        identification: [{position: 40, label: 'Unit Of Measurement', importance: #HIGH, hidden: true}] }
  Uom;
  @UI:{ lineItem: [{position: 50, label: 'Price', importance: #HIGH}],
        identification: [{position: 50, label: 'Price', importance: #HIGH}] }
  Price;
  @UI:{ lineItem: [{position: 60, label: 'Currency', importance: #HIGH,hidden: true}],
        identification: [{position: 60, label: 'Currency', importance: #HIGH, hidden: true}] }
  Currency;
  @UI:{ lineItem: [{position: 70, label: 'MFG Date', importance: #HIGH, hidden: true}],
        identification: [{position: 70, label: 'Manufacture Date', importance: #HIGH}] }
  MfgDate;
  @UI:{ lineItem: [{position: 80, label: 'Exp. Date', importance: #HIGH, hidden: true}],
        identification: [{position: 80, label: 'Expiry Date', importance: #HIGH}] }
  ExpDate;
  @UI:{ lineItem: [{position: 90, label: 'Plant', importance: #HIGH}],
        identification: [{position: 90, label: 'Plant', importance: #HIGH}] }
  Plant;
   @UI:{ lineItem: [{position: 91, label: 'Valid Days', importance: #HIGH}],
        identification: [{position: 91, label: 'Valid Days', importance: #HIGH}],
        selectionField: [{ position: 91  }] }
  ValidDays;
  @UI:{ lineItem: [{position: 100, label: 'Created By', importance: #HIGH}],
        identification: [{position: 100, label: 'Created By', importance: #HIGH}] }
  LocalCreatedBy;
  @UI:{ lineItem: [{position: 110, label: 'Created At', importance: #HIGH }],
        identification: [{position: 110, label: 'Created At', importance: #HIGH}] }
  LocalCreatedAt;
  @UI:{ lineItem: [{position: 120, label: 'Last Changed By', importance: #HIGH, hidden: true }],
        identification: [{position: 120, label: 'Last Changed By', importance: #HIGH}] }
  LocalLastChangedBy;
  @UI:{ lineItem: [{position: 130, label: 'Last Changed At', importance: #HIGH, hidden: true}],
        identification: [{position: 130, label: 'Last Changed At', importance: #HIGH}] }
  LocalLastChangedAt;
  @UI:{ lineItem: [{position: 140, label: 'Last Changed At', importance: #HIGH, hidden: true}],
        identification: [{position: 140, label: 'Product ID', importance: #HIGH, hidden: true}] }
  LastChangedAt;

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
#CUSTOMER      ← Customer adds their annotations
#PARTNER       ← Relates to a partner extension of a product 
#INDUSTRY      ← Relates to an industry-specific extension of a product (SAP Industry Solution)
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
| `@UI.facet` | Define sections/tabs on the object page | "Product Details" tab |
| `@UI.hidden` | Hide field from UI entirely | Internal fields |
| `@UI.dataAction` | Add a button for a custom action | "Valid Days" button |

---

## ⚙️ Step 6 — Behavior Definition (Interface View)

### What is a Behavior Definition?

A Behavior Definition (BDEF) is a repository object that defines **what operations are allowed** on a Business Object. It answers questions like:
- Can this BO be created, updated, deleted?
- Which fields are read-only or mandatory?
- Does it support draft handling?
- What custom actions exist (like "Valid Days")?
- How are primary keys generated?

### Interface View Behavior Definition

The base BDEF sits on the Interface View and defines the **complete set of capabilities** of the BO:

```abap
managed implementation in class zbp_nk_product_i unique;
strict ( 2 );
with draft;
define behavior for znk_product_I //alias <alias_name>
persistent table znk_t_product
draft table znk_t_product_d
lock master                                                         -- Locking mechanism
total etag LastChangedAt
authorization master ( global )
etag master LastChangedAt                                           -- ETag for concurrency control
{

 -- Field controls
 field ( mandatory : create ) ProductId;                            -- Makes Mandatory when create
 field ( readonly )                                                 -- Makes non-editable
 LocalCreatedAt,
 LocalCreatedBy,
 LocalLastChangedAt,
 LocalLastChangedBy,
 LastChangedAt;
 field ( numbering : managed ) ProUuid;                             -- Makes Numbering Managed
 field ( readonly : update ) ProUuid, ProductId;                    -- Makes non-editable when update

   -- Standard CRUD operations (framework handles automatically in managed)
  create;
  update;
  delete;

  -- Custom actions (developer implements)
  action  ( features : instance ) checkProductValidity result [1] $self;
  action  ( features : instance ) clearValidDays result [1] $self;

  -- Draft related Actions
  draft action Edit;
  draft action Activate optimized;
  draft action Discard;
  draft action Resume;
  draft determine action Prepare;

  -- Mapping is very important to store the values to DB 
  mapping for znk_t_product
    {
      ProUuid            = pro_uuid;
      ProductId          = product_id;
      Description        = description;
      Qty                = qty;
      Uom                = uom;
      Price              = price;
      Currency           = currency;
      MfgDate            = mfg_date;
      ExpDate            = exp_date;
      Plant              = plant;
      ValidDays          = validdays;
      LocalCreatedBy     = local_created_by;
      LocalCreatedAt     = local_created_at;
      LocalLastChangedAt = local_last_changed_at;
      LocalLastChangedBy = local_last_changed_by;
      LastChangedAt      = last_changed_at;
    }
}
```

### Key BDEF keywords explained

**`managed`** — Tells the RAP framework to auto-handle all database operations (INSERT, UPDATE, DELETE). You don't write any `MODIFY dbtab` statements yourself.

**`with draft`** — Activates the draft handling mechanism. The framework will use your `draft table` for all in-progress changes.

**`etag master LocalLastChangedAt`** — Enables optimistic locking. Before any update, RAP compares the `LocalLastChangedAt` timestamp in the request with the database to detect if someone else modified the record in between.

**`lock master`** — When a user starts editing (enters draft mode), the record is locked so no other user can simultaneously edit the same record.

**`numbering:managed`** — (when used on a field) Tells the framework to auto-generate a UUID primary key at CREATE time — no manual key assignment needed.

**`validation`** — A method you implement in the Behavior Implementation class. Called by the framework before save. Return error messages to block the save.

**`determination`** — A method that auto-runs when specific fields change. Used for defaulting values (e.g., set status to "Open" when a Product is created).

**`action`** — A custom operation beyond standard CRUD. Adds a button in the Fiori UI. Example: "Valid Days" changes status to Accepted.

---

## 🔒 Step 7 — Behavior Definition (Projection View)

### What is the Projection Behavior Definition?

The Projection BDEF sits on the **Consumption View** and exposes a **subset** of the base BDEF's operations for a specific service. It acts as a gatekeeper — you decide which operations this particular OData service gets to offer.

```abap
projection;
strict ( 2 );
use draft;

define behavior for znk_product_c //alias <alias_name>
{
  use create;                           -- Expose create to this service
  use update;                           -- Expose update to this service
  use delete;                           -- Expose delete to this service

  use action Edit;
  use action Activate;
  use action Discard;
  use action Prepare;
  use action Resume;


  -- Expose only the custom actions this service needs
  use action checkProductValidity;
  use action clearValidDays;
}
```

### Interface BDEF vs Projection BDEF — Why both?

| | Interface BDEF | Projection BDEF |
|---|---|---|
| Sits on | Interface View (Z*_R) | Consumption View (Z*_C) |
| Contains | ALL possible operations | SUBSET for this service |
| Implementation | Developer writes code here | Just `use` statements — no code |
| Reusability | One per BO | One per service/app |
| Analogy | Full employee record | Role-specific access form |

Imagine you have two apps using the same Travel BO:
- **Admin App** → projection BDEF exposes: create, update, delete, ValidDays, Clear
- **Read-Only Report App** → projection BDEF exposes: nothing (read-only by omission)

Same base BO, completely different service behavior — controlled purely by the projection BDEF.

---

## 💻 Step 8 — Behavior Implementation (BAP)

### What is the Behavior Implementation?

The Behavior Implementation is an **ABAP class** that contains the actual code for all custom operations defined in the base Behavior Definition — validations, determinations, and actions. In managed scenarios, you do NOT write code for standard CRUD — the framework handles that. You only write code for your custom business logic.

The class is named in the BDEF with `implementation in class zbp_nk_product_i unique;`.

```abap

"! <p class="shorttext synchronized" lang="en">Behavior Implementation for ZNK_PRODUCT_I</p>
CLASS zbp_nk_product_i DEFINITION PUBLIC ABSTRACT FINAL FOR BEHAVIOR OF znk_product_i.
ENDCLASS.

CLASS zbp_nk_product_i IMPLEMENTATION.
ENDCLASS.
```

The actual logic sits in **local classes** inside the implementation class's `.locals_imp` include:

### Validation Example — validateDates

```abap

CLASS lhc_znk_product_I DEFINITION INHERITING FROM cl_abap_behavior_handler.
  PRIVATE SECTION.
    METHODS get_global_authorizations FOR GLOBAL AUTHORIZATION
    REQUEST requested_authorizations FOR znk_product_I RESULT result.
    METHODS checkProductValidity FOR MODIFY
       keys FOR ACTION znk_product_I~checkProductValidity RESULT result.
    METHODS get_instance_features FOR INSTANCE FEATURES
      keys REQUEST requested_features FOR znk_product_I RESULT result.
    METHODS clearvaliddays FOR MODIFY
       keys FOR ACTION znk_product_i~clearvaliddays RESULT result.
ENDCLASS.

CLASS lhc_znk_product_I IMPLEMENTATION.
  METHOD get_global_authorizations.
  ENDMETHOD.

  METHOD checkProductValidity.

    READ ENTITIES OF znk_product_I IN LOCAL MODE                               --Read entries from Interface view to lt_products
        ENTITY znk_product_I
        FIELDS ( MfgDate ExpDate )
        WITH CORRESPONDING #( keys )
        RESULT DATA(lt_products)
        FAILED DATA(ls_failed).

    MODIFY ENTITIES OF znk_product_I IN LOCAL MODE                             --Modify entries from Interface view to lt_products
        ENTITY znk_product_I
        UPDATE FIELDS ( ValidDays )
        WITH VALUE #(
          FOR ls IN lt_products
          LET lv_days = CONV i( ls-ExpDate - ls-MfgDate )
          IN ( %tky      = ls-%tky
               ValidDays = lv_days ) ).

    LOOP AT lt_products INTO DATA(ls_product).                                  --Validation on to lt_products
      IF ls_product-ExpDate <= ls_product-MfgDate.
        APPEND VALUE #(
          %tky = ls_product-%tky
          %msg = new_message_with_text(
                   severity = if_abap_behv_message=>severity-error
                   text     = 'Expiry Date must be after Manufacturing Date!' )
        ) TO reported-znk_product_i.
      ENDIF.
    ENDLOOP.
    READ ENTITIES OF znk_product_I IN LOCAL MODE
                                   ENTITY znk_product_I
                                   ALL FIELDS
                                   WITH CORRESPONDING #( keys )
                                   RESULT DATA(lt_result).

    result = VALUE #( FOR ls IN lt_result                                        --Return the modified entries to result
                      ( %tky   = ls-%tky
                        %param = ls ) ).   

  ENDMETHOD.

  METHOD get_instance_features.
    READ ENTITIES OF znk_product_I IN LOCAL MODE
     ENTITY znk_product_I
     FIELDS ( ProductId )
     WITH CORRESPONDING #( keys )
     RESULT DATA(lt_products)
     FAILED DATA(ls_failed).

    result = VALUE #( FOR ls IN lt_products
                      ( %tky                          = ls-%tky
                        %action-checkProductValidity  = if_abap_behv=>fc-o-enabled ) ).
  ENDMETHOD.

  METHOD clearValidDays.
    READ ENTITIES OF znk_product_I IN LOCAL MODE
        ENTITY znk_product_I
        FIELDS ( MfgDate ExpDate )
        WITH CORRESPONDING #( keys )
        RESULT DATA(lt_products)
        FAILED DATA(ls_failed).

    MODIFY ENTITIES OF znk_product_I IN LOCAL MODE
        ENTITY znk_product_I
        UPDATE FIELDS ( ValidDays )
        WITH VALUE #(
          FOR ls IN lt_products
          LET lv_days = 0
          IN ( %tky      = ls-%tky
               ValidDays = lv_days ) ).

    READ ENTITIES OF znk_product_I IN LOCAL MODE
                      ENTITY znk_product_I
                      ALL FIELDS
                      WITH CORRESPONDING #( keys )
                      RESULT DATA(lt_result).
    result = VALUE #( FOR ls IN lt_result
        ( %tky   = ls-%tky
          %param = ls ) ).   
  ENDMETHOD.

ENDCLASS.
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

### Action Example — checkProductValidity

```abap
METHOD checkProductValidity.

    READ ENTITIES OF znk_product_I IN LOCAL MODE
        ENTITY znk_product_I
        FIELDS ( MfgDate ExpDate )
        WITH CORRESPONDING #( keys )
        RESULT DATA(lt_products)
        FAILED DATA(ls_failed).

    MODIFY ENTITIES OF znk_product_I IN LOCAL MODE
        ENTITY znk_product_I
        UPDATE FIELDS ( ValidDays )
        WITH VALUE #(
          FOR ls IN lt_products
          LET lv_days = CONV i( ls-ExpDate - ls-MfgDate )
          IN ( %tky      = ls-%tky
               ValidDays = lv_days ) ).

    LOOP AT lt_products INTO DATA(ls_product).
      IF ls_product-ExpDate <= ls_product-MfgDate.
        APPEND VALUE #(
          %tky = ls_product-%tky
          %msg = new_message_with_text(
                   severity = if_abap_behv_message=>severity-error
                   text     = 'Expiry Date must be after Manufacturing Date!' )
        ) TO reported-znk_product_i.
      ENDIF.
    ENDLOOP.
    READ ENTITIES OF znk_product_I IN LOCAL MODE
                                   ENTITY znk_product_I
                                   ALL FIELDS
                                   WITH CORRESPONDING #( keys )
                                   RESULT DATA(lt_result).

    result = VALUE #( FOR ls IN lt_result
                      ( %tky   = ls-%tky
                        %param = ls ) ).   

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
@EndUserText.label: 'Service Definition for Consumption view'
define service Znk_sd_product {
  expose znk_product_c;
}
```

### Service Binding

The Service Binding ties a Service Definition to a specific **protocol and usage type**. For Fiori apps, you create an OData V4 binding with type `UI`:

- **Binding Type:** `OData V4 - UI`
- **Activate** the binding to get the OData service URL
- Use **"Preview"** in ADT to open the Fiori Elements app directly

Once activated, your Product BO is fully functional as a Fiori Elements List Report + Object Page app — with list, filter, create, edit, draft, save, and all your custom actions.

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
- Action logic (checkProductValidity, Clear)

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
- SAP Community Profile: [community.sap.com](https://profile.sap.com/u/iamnaveen)
- GitHub: [github.com](https://github.com/knaveennavi)

---

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

*If this repository helped you understand RAP, please give it a ⭐ — it helps others find it too!*

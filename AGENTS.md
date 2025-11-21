# **AGENTS.md — Case Support (UDW Case Manager)**

**Author:** Tin Huynh
**Purpose:** Technical specification for building a modular Case Support system inside the UDW plugin, following the Tin Structure.  
**Scope:** WordPress module (custom tables + REST), admin UI, reminder engine, member profile integration.

---

## 1) Overview

The Case Support module lets staff open, work, and close support cases linked to UDW members (or family members). It favors lean **custom tables**, **server‑side lists**, and **strict capabilities** over heavyweight helpdesk plugins.

**Core features**
- Case creation & assignment
- Notes & follow‑ups (with reminders)
- Status workflow (`open → in_progress → pending → closed`, with reopen)
- **Category (required):** drives defaults (priority, assignee, SLA)
- Dashboards/lists, “My Cases”, Reminders
- UDW Member profile tab integration
- REST endpoints with permission gates
- Tin Structure modularity (controllers/models/views/handlers)

---

## 2) Roles & Permissions

### 2.1 Roles (organization titles → module authority)
- **Executive Director** — admin of system  
- **Associate Director** — Supervisor; admin of system  
- **Admin** — admin of system  
- **Case Manager** — create & manage their cases  
- **Program Coordinator** — contributes to case support; limited authority

> Map these titles to WP roles and attach the capabilities below (plugin-scoped).

### 2.2 Capabilities (prefix `cs_`)
- **Read/Reports:** `cs_read_case`, `cs_view_reports`  
- **Create/Edit:** `cs_create_case`, `cs_edit_case_assigned`, `cs_edit_case_any`, `cs_add_case_note`, `cs_set_reminder`  
- **Assignment:** `cs_assign_case_self`, `cs_assign_case_any`  
- **Lifecycle:** `cs_close_case_assigned`, `cs_close_case_any`, `cs_reopen_case`, `cs_delete_case` *(rare)*  
- **Administration:** `cs_manage_categories`, `cs_manage_settings`, `cs_manage_sla`, `cs_manage_exports`

### 2.3 Capability matrix (defaults)

| Capability | Program Coord. | Case Manager | Assoc. Director | Exec. Director | Admin |
|---|:--:|:--:|:--:|:--:|:--:|
| cs_read_case | ✅ | ✅ | ✅ | ✅ | ✅ |
| cs_view_reports | ✅ | ✅ | ✅ | ✅ | ✅ |
| cs_create_case | ✅ | ✅ | ✅ | ✅ | ✅ |
| cs_edit_case_assigned | ✅ | ✅ | ✅ | ✅ | ✅ |
| cs_edit_case_any | ❌ | ❌ | ✅ | ✅ | ✅ |
| cs_add_case_note | ✅ | ✅ | ✅ | ✅ | ✅ |
| cs_set_reminder | ✅ | ✅ | ✅ | ✅ | ✅ |
| cs_assign_case_self | ❌ *(opt‑in)* | ✅ | ✅ | ✅ | ✅ |
| cs_assign_case_any | ❌ | ❌ | ✅ | ✅ | ✅ |
| cs_close_case_assigned | ❌ *(request close)* | ✅ | ✅ | ✅ | ✅ |
| cs_close_case_any | ❌ | ❌ | ✅ | ✅ | ✅ |
| cs_reopen_case | ❌ | ❌ | ✅ | ✅ | ✅ |
| cs_delete_case | ❌ | ❌ | ❌ | ❌ | ✅ *(rare)* |
| cs_manage_categories | ❌ | ❌ | ✅ | ✅ | ✅ |
| cs_manage_settings | ❌ | ❌ | ✅ | ✅ | ✅ |
| cs_manage_sla | ❌ | ❌ | ✅ | ✅ | ✅ |
| cs_manage_exports | ❌ | ❌ | ✅ | ✅ | ✅ |

**Per‑record policy**
- **Owner or Assignee** may `cs_edit_case_assigned` / `cs_read_case`.
- **Assign:** Case Manager may **self‑assign**; Directors/Admin assign any.
- **Close:** Case Manager may close **own**; Directors/Admin close **any**.
- Enforce via `map_meta_cap` + record‑level checks.

---

## 3) Workflow & States

```
NEW → OPEN → IN_PROGRESS ↔ PENDING → CLOSED
                 ↑                     ↓
                REOPEN  ←──────────────┘
```

- `NEW` is transient (first save ⇒ `OPEN`).
- `IN_PROGRESS` requires an assignee.
- `PENDING` often has a reminder.
- `CLOSED` requires a resolution note; can be reopened by Directors/Admin.

---

## 4) Category (first‑class)

**Required on every case.** Drives defaults and reporting.

**Fields per category**
- `label`, `slug`, `description`, `active`, `sort`, `color`  
- `default_priority` (`low|normal|high|urgent`)  
- `default_assignee` (WP user id, optional)  
- `sla_first_response_hours` (int)  
- `sla_followup_days` (int)  
- `require_reminder` (bool)  
- `kb_link` (optional)

**Behavior**
- On create/change: prefill **priority**, **assignee**, first **reminder** according to category policy.
- Duplicate warning: if `(member_id, category_id)` already has non‑closed case.

**Seed categories (editable)**
- Benefits & Eligibility  
- Member Support  
- Documentation & Records  
- Health & Care Coordination  
- Financial Assistance  
- Housing & Transportation  
- Technical / Portal Support  
- Other (Needs Triage)

---

## 5) Data Model (tables & indexes)

Use `$wpdb->get_charset_collate()` (utf8mb4).

### 5.1 `cs_cases`
- `id BIGINT PK AI`
- `member_id BIGINT NULL`
- `family_member_id BIGINT NULL`
- `category_id BIGINT NOT NULL`
- `created_by BIGINT NOT NULL`
- `assigned_to BIGINT NULL`
- `title VARCHAR(255) NOT NULL`
- `description TEXT NULL`
- `status ENUM('open','in_progress','pending','closed') DEFAULT 'open'`
- `priority ENUM('low','normal','high','urgent') DEFAULT 'normal'`
- `reminder_date DATETIME NULL`
- `snapshot_json LONGTEXT NULL`
- `created_at DATETIME NOT NULL`
- `updated_at DATETIME NOT NULL`

**Indexes**
- `idx_assignee_status_reminder (assigned_to,status,reminder_date)`
- `idx_status_priority_updated (status,priority,updated_at)`
- `idx_member (member_id)`
- `idx_category_status (category_id,status)`

### 5.2 `cs_case_notes`
- `id BIGINT PK AI`
- `case_id BIGINT NOT NULL`
- `author_id BIGINT NOT NULL`
- `note TEXT NOT NULL`
- `is_follow_up TINYINT(1) DEFAULT 0`
- `created_at DATETIME NOT NULL`  
Index: `idx_case_created (case_id,created_at)`

### 5.3 `cs_reminders_log`
- `id BIGINT PK AI`
- `case_id BIGINT NOT NULL`
- `note_id BIGINT NULL`
- `reminder_date DATETIME NOT NULL`
- `notified TINYINT(1) DEFAULT 0`
- `created_at DATETIME NOT NULL`  
Indexes: `idx_case_reminder (case_id,reminder_date)`, `idx_notified (notified,reminder_date)`

### 5.4 `cs_categories`
- `id BIGINT PK AI`
- `slug VARCHAR(64) UNIQUE`
- `label VARCHAR(128) NOT NULL`
- `description TEXT NULL`
- `active TINYINT(1) DEFAULT 1`
- `default_priority ENUM('low','normal','high','urgent') DEFAULT 'normal'`
- `default_assignee BIGINT NULL`
- `require_reminder TINYINT(1) DEFAULT 0`
- `sla_first_response_hours INT NULL`
- `sla_followup_days INT NULL`
- `color VARCHAR(7) NULL`
- `sort SMALLINT DEFAULT 0`
- `created_at DATETIME NOT NULL`
- `updated_at DATETIME NOT NULL`  
Indexes: `UNIQUE(slug)`, `active_sort (active,sort)`

---

## 6) Tin Structure (folders & responsibilities)

```
/includes/CaseSupport/
  CaseSupport.php                        # bootstrap, menus, routes, install
  /models/
    CasesModel.php                       # SQL for cases (filters/index-aware)
    NotesModel.php                       # SQL for notes
    ReminderModel.php                    # reminder queries + helpers
    CategoriesModel.php                  # CRUD + routing defaults
  /controllers/
    AdminListController.php              # list + filters (server-side)
    AdminDetailController.php            # detail header/snapshot/actions
    NotesController.php                  # notes timeline + add
    ReminderController.php               # reminders views/actions
    AdminCategoriesController.php        # category manager (settings)
  /views/
    /list/index.php
    /detail/detail.php
    /detail/notes.php
    /components/case-card.php
    /components/note-item.php
    /settings/categories.php
  /handlers/
    ajax-create-case.php
    ajax-update-case.php
    ajax-add-note.php
    ajax-complete-reminder.php
  /cron/
    daily-reminder-check.php
  /demo/
    demo-list.php
```

---

## 7) Admin UI & Sitemap

```
UDW Members
└─ Case Support
   ├─ Dashboard
   ├─ Cases
   │   ├─ All Cases
   │   ├─ My Cases
   │   ├─ Unassigned
   │   └─ New Case
   ├─ Reminders (Overdue / Today / Upcoming)
   ├─ Reports (SLA, Aging, Category Volume)
   └─ Settings
       ├─ Categories & Routing
       ├─ Priorities & SLA Policy
       ├─ Notifications
       └─ Privacy & Retention
```

**Member profile tab “Case Support”**
- Member’s cases (with **Category** badge)  
- “Create Case” (category required)  
- Quick open of case detail

**UI**
- Bootstrap 5 cards; badges for status/priority/category  
- Server‑side listing (DataTables‑ready)  
- Overdue reminders highlighted

---

## 8) REST API (namespace `udw-case/v1`)

**Cases**
- `GET /cases` — filters: `q,status[],priority[],category_id,assigned_to,reminder_before,updated_after,start,length,order`  
- `GET /cases/{id}`  
- `POST /cases` — create (requires `category_id`, member link, title, priority)  
- `PATCH /cases/{id}` — update (transitions validated)  
- `POST /cases/{id}/notes` — add note (`is_follow_up` ⇒ updates reminder)

**Categories**
- `GET /categories?active=1`  
- `POST /categories` *(Directors/Admin)*  
- `PATCH /categories/{id}`  
- `DELETE /categories/{id}` *(soft delete → `active=0`)*

**Security**
- Every route defines a strict `permission_callback`; per‑record checks honor owner/assignee vs managers.

**DataTables contract (server‑side)**
- Accept `start`, `length`, `order[column]/order[dir]`  
- Respond `{draw, recordsTotal, recordsFiltered, data:[...]}`  
- Columns: `id, member, title, category, status, priority, assignee, updated, reminder, action`

---

## 9) Reminders & Scheduling

- **Truth:** `cs_cases.reminder_date` (UTC recommended).  
- **Job:** scheduled scan of cases `reminder_date <= NOW()` and `status != 'closed'`; log entry in `cs_reminders_log`; notify assignee.  
- **Cadence:** daily sweep + short interval (e.g., 15 min) if using a queue.  
- **UI:** Reminders screen with **Overdue / Today / Upcoming** tabs + bulk snooze/reassign.  
- **Close:** clearing reminders on close.

---

## 10) Security & Coding Standards

- Nonces on all write actions; capabilities on routes and forms.  
- All SQL via `$wpdb->prepare()`; sanitize in, escape out.  
- Namespaced classes; one feature per file.  
- `map_meta_cap` enables owner/assignee control without global manager caps.  
- Audit status/assignee changes (v2 history log).

---

## 11) Install / Upgrade

- Activation runs `dbDelta()` for:
  - `cs_cases`, `cs_case_notes`, `cs_reminders_log`, `cs_categories`  
  - Adds `category_id` to `cs_cases` when missing; backfill “Other (Needs Triage)”
- Store module version for migrations.  
- Bootstrap capabilities (add to Administrator; optional plugin roles for Case Manager, Program Coordinator, Supervisor).  
- Register admin menus under **UDW Members**.

---

## 12) Development Phases

**Phase 0 – Demo**: folder tree + `demo-list.php` (static UI), temporary “Case Support Demo” menu.  
**Phase 1 – Foundation**: installer, caps, menus, REST scaffolding.  
**Phase 2 – Case List**: `GET /cases` + server‑side table.  
**Phase 3 – Detail & Notes**: view/edit, add notes, follow‑up.  
**Phase 4 – Categories**: CRUD + routing defaults applied on create/change.  
**Phase 5 – Reminders**: scheduler + log + notifications + Reminders UI.  
**Phase 6 – Member Tab / Polish**: integration, reports, accessibility, QA.

---

## 13) Acceptance Criteria

- Category **required**; defaults (priority, assignee, first reminder) applied.  
- Program Coordinator **cannot assign to others** or close; may request close.  
- Case Manager can **self‑assign** and **close own**.  
- Directors/Admin can assign/close/reopen **any**; manage categories/settings.  
- Server‑side list handles large datasets; overdue reminders highlighted.  
- All REST writes enforce caps; per‑record checks respect owner/assignee vs manager.

---

## 14) Snippets (skeletons)

**Activation (caps + install)**
```php
register_activation_hook(__FILE__, function() {
  if ($r = get_role('administrator')) {
    foreach ([
      'cs_read_case','cs_view_reports','cs_create_case','cs_edit_case_assigned','cs_edit_case_any',
      'cs_add_case_note','cs_set_reminder','cs_assign_case_self','cs_assign_case_any',
      'cs_close_case_assigned','cs_close_case_any','cs_reopen_case','cs_delete_case',
      'cs_manage_categories','cs_manage_settings','cs_manage_sla','cs_manage_exports'
    ] as $cap) { $r->add_cap($cap); }
  }
  // dbDelta() for cs_cases, cs_case_notes, cs_reminders_log, cs_categories
});
```

**REST (permission gate)**
```php
add_action('rest_api_init', function(){
  register_rest_route('udw-case/v1', '/cases', [
    'methods'  => 'GET',
    'callback' => ['UDW\\CaseSupport\\Controllers\\AdminListController','rest_list'],
    'permission_callback' => function(){ return current_user_can('cs_read_case'); }
  ]);
});
```

**map_meta_cap (record‑level)**
```php
add_filter('map_meta_cap', function($caps,$cap,$user_id,$args){
  if (!in_array($cap,['cs_read_case','cs_edit_case_assigned','cs_close_case_assigned'],true)) return $caps;
  $case_id = intval($args[0] ?? 0); if (!$case_id) return ['do_not_allow'];
  $case = \UDW\CaseSupport\Models\CasesModel::get($case_id); if (!$case) return ['do_not_allow'];
  if (user_can($user_id,'cs_edit_case_any') || user_can($user_id,'cs_close_case_any')) return ['read'];
  $is_owner = (int)$case->created_by === (int)$user_id;
  $is_assignee = $case->assigned_to && (int)$case->assigned_to === (int)$user_id;
  if (in_array($cap,['cs_read_case','cs_edit_case_assigned'],true) && ($is_owner || $is_assignee)) return ['read'];
  if ($cap === 'cs_close_case_assigned' && $is_assignee) return ['read'];
  return ['do_not_allow'];
},10,4);
```

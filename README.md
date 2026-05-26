# ServiceNow Query Editor — Chrome Extension

A powerful Chrome extension that provides a SQL-like query interface for ServiceNow. Write queries against any ServiceNow table, explore CMDB relationships visually, browse system logs in a dashboard-style Log Explorer, manage saved queries in folders, edit records inline, and much more — all without leaving your browser.

---

## Table of Contents

- [Getting Started](#getting-started)
- [Writing & Executing Queries](#writing--executing-queries)
- [Query Cookbook — by Complexity](#query-cookbook--by-complexity)
- [Inner (Nested) Queries](#inner-nested-queries)
- [WHERE Subqueries (Semi-join / Anti-join)](#where-subqueries-semi-join--anti-join)
- [CMDB Relationship Queries](#cmdb-relationship-queries)
- [CMDB Relationship Diagram](#cmdb-relationship-diagram)
- [Results Grid](#results-grid)
- [Record Management](#record-management)
- [Tabs](#tabs)
- [Query History](#query-history)
- [Saved Queries](#saved-queries)
- [Copy Query](#copy-query)
- [Table Information Panel](#table-information-panel)
- [Export Results](#export-results)
- [Themes](#themes)
- [Settings](#settings)
- [Flow Inspector](#flow-inspector)
- [Log Explorer](#log-explorer)
- [Keyboard Shortcuts](#keyboard-shortcuts)

---

## Getting Started

### Connecting to a ServiceNow Instance

The extension automatically detects any open ServiceNow tabs in your browser. Open the extension, and select an active instance from the dropdown at the top of the page. A connection status indicator (green/red/yellow) shows whether the extension is connected.

If the connection drops, use the **Reconnect** button to re-establish it. The extension uses your existing ServiceNow browser session — no separate credentials are required.

---

## Writing & Executing Queries

### SQL-Like Syntax

Write queries using a familiar SQL-like syntax that the extension translates into ServiceNow REST API calls:

```sql
SELECT field1, field2, field3
FROM table_name
WHERE condition1 = 'value' AND condition2 != 'other'
ORDER BY field1 DESC
LIMIT 100
```

Use `SELECT *` to return all fields, or specify individual field names. The column order in the results grid matches the order in your `SELECT` statement.

### Supported Operators

The extension maps standard SQL operators to their ServiceNow encoded-query equivalents:

| SQL Operator | Description | ServiceNow Equivalent | Example |
|---|---|---|---|
| `=` | Equals | `=` | `active = true` |
| `!=` or `<>` | Not equals | `!=` | `state != 6` |
| `>` | Greater than | `>` | `priority > 2` |
| `<` | Less than | `<` | `priority < 3` |
| `>=` | Greater or equal | `>=` | `impact >= 2` |
| `<=` | Less or equal | `<=` | `impact <= 2` |
| `LIKE '%value%'` | Contains | `LIKE` | `short_description LIKE '%error%'` |
| `LIKE 'value%'` | Starts with | `STARTSWITH` | `name LIKE 'SAP%'` |
| `LIKE '%value'` | Ends with | `ENDSWITH` | `name LIKE '%Server'` |
| `NOT LIKE '%value%'` | Does not contain | `NOT LIKE` | `short_description NOT LIKE '%test%'` |
| `NOT LIKE 'value%'` | Does not start with | `!STARTSWITH` | `name NOT LIKE 'Test%'` |
| `NOT LIKE '%value'` | Does not end with | `!ENDSWITH` | `name NOT LIKE '%dev'` |
| `IN (a, b, c)` | Value in list | `IN` | `state IN (1, 2, 3)` |
| `IN (SELECT ...)` | Value in subquery | `IN` (resolved) | `group IN (SELECT group FROM ...)` |
| `NOT IN (a, b, c)` | Value not in list | `NOT IN` | `state NOT IN (6, 7)` |
| `NOT IN (SELECT ...)` | Value not in subquery | `NOT IN` (resolved) | `group NOT IN (SELECT group FROM ...)` |

Conditions can be combined with `AND` / `OR` and grouped using parentheses.

### SELECT COUNT

Use `SELECT COUNT FROM table` (or `SELECT COUNT(*) FROM table`) to get a fast record count without fetching any rows:

```sql
SELECT COUNT FROM incident WHERE active = true
```

This uses ServiceNow's aggregate API for performance.

### SELECT DISTINCT

Use `DISTINCT` to return only unique rows based on all selected fields:

```sql
SELECT DISTINCT assignment_group, priority
FROM incident
WHERE active = true
```

This fetches records from ServiceNow normally, then deduplicates client-side based on the combination of all fields in the `SELECT` clause. The result toast shows how many unique records were kept — for example, *"87 unique records (from 200 fetched) in 340ms"*.

`DISTINCT` works with all other features including dot-walked fields, `WHERE` subqueries, and `ORDER BY`:

```sql
SELECT DISTINCT caller_id.department
FROM incident
WHERE active = true AND priority < 3
ORDER BY caller_id.department
```

> **Note:** ServiceNow's `LIMIT` is applied server-side before deduplication, so the final unique count may be lower than the `LIMIT`. For example, `LIMIT 100` fetches 100 records, which might yield 60 unique rows after deduplication.

### Query Formatting

Click the **Format** button (or simply use it for readability) to auto-format your query. The formatter puts each clause (`SELECT`, `FROM`, `WHERE`, `AND`, `OR`, `ORDER BY`, `LIMIT`) on its own line and properly indents inner queries.

### Syntax Highlighting

The query editor provides real-time syntax highlighting as you type:

- **Keywords** (SELECT, FROM, WHERE, etc.) are colour-coded
- **Table names** (after FROM) are highlighted differently
- **Field names**, **strings**, **numbers**, **operators**, and **comments** each have distinct colours

### Autocomplete

As you type, the extension offers intelligent autocompletion:

- **Table names** — suggested after `FROM`, sourced from the connected instance's available tables
- **Field names** — suggested after `SELECT`, `WHERE`, `AND`, `OR`, etc., based on the table in your `FROM` clause. Fields show both the API name and human-readable label
- **Dot-walk fields** — when typing a reference field followed by `.`, the extension suggests fields from the referenced table

Navigate suggestions with **Up/Down** arrows, select with **Tab** or **Enter**, and dismiss with **Escape**.

### Execute & Cancel

- Click **Execute** or press **Ctrl+Enter** (Cmd+Enter on Mac) to run the query
- A progress modal shows the number of records fetched so far vs. the total count, with a percentage indicator
- Click **Cancel** at any time to stop fetching and display the records retrieved so far

---

## Query Cookbook — by Complexity

A progressive tour of the editor's query capabilities, from the smallest useful query up to a fully composed real-world one. Each level introduces one new capability; everything below it stays valid as you climb.

### Level 1 — Basic SELECT

Return rows from a table, choosing your own columns and order:

```sql
SELECT number, short_description, state
FROM incident
LIMIT 25
```

- `SELECT *` returns every field on the table.
- Column order in the results grid follows the order in `SELECT`.
- Without `LIMIT`, the editor still fetches in bounded batches (see [Execute & Cancel](#execute--cancel)).

#### Sorting

Single-field `ORDER BY` with optional direction:

```sql
SELECT number, priority, sys_created_on
FROM incident
ORDER BY sys_created_on DESC
LIMIT 100
```

### Level 2 — Dot-walked fields (reference traversal)

Reference fields (`caller_id`, `assigned_to`, `cmdb_ci`, `assignment_group`, …) point at records on another table. Use **dot notation** to read fields on the referenced record. Works in `SELECT`, `WHERE`, and `ORDER BY`:

```sql
SELECT number, caller_id.name, caller_id.email, assignment_group.name
FROM incident
WHERE active = true
ORDER BY caller_id.name ASC
LIMIT 50
```

Chains are supported — every step except the last must itself be a reference field:

```sql
SELECT number,
       caller_id.manager.email,
       caller_id.department.name
FROM incident
LIMIT 25
```

The autocomplete suggests fields one segment at a time as you type each `.`.

### Level 3 — Filtering: AND / OR / parentheses

Combine conditions with `AND` and `OR`, and group with parentheses to control precedence:

```sql
SELECT number, priority, state
FROM incident
WHERE active = true
  AND priority IN (1, 2)
  AND (state = 1 OR state = 2)
ORDER BY priority ASC
```

The same works against dot-walked fields:

```sql
SELECT number, caller_id.name, assignment_group.name
FROM incident
WHERE active = true
  AND (assignment_group.name = 'Network'
       OR assignment_group.name = 'Database')
  AND caller_id.email LIKE '%@acme.com'
LIMIT 200
```

See [Supported Operators](#supported-operators) for the full operator map.

#### Pattern matching with LIKE / NOT LIKE

The wildcard position determines which ServiceNow operator is emitted:

```sql
WHERE short_description LIKE '%network%'   -- contains
WHERE name              LIKE 'SAP%'        -- starts with
WHERE email             LIKE '%@acme.com'  -- ends with
WHERE name              LIKE 'web-prod-01' -- exact match (no wildcards)
```

`NOT LIKE` follows the same rules (`NOT LIKE`, `!STARTSWITH`, `!ENDSWITH`, or `!=` depending on wildcard placement).

#### IN with a literal list

```sql
SELECT number, state
FROM incident
WHERE active = true
  AND state IN (1, 2, 3)
  AND priority NOT IN (4, 5)
```

### Level 4 — IN subqueries (semi-join / anti-join)

Filter parent rows by the results of a second query, without writing a JOIN.

**Semi-join** — group members whose `group` has the `itil` role assigned:

```sql
SELECT user.name, group.name
FROM sys_user_grmember
WHERE user.active = false
  AND group IN (
    SELECT group
    FROM sys_group_has_role
    WHERE role.name = 'itil'
  )
```

**Anti-join** — active incidents whose assignee is *not* a member of the Network group:

```sql
SELECT number, short_description, assigned_to.name
FROM incident
WHERE active = true
  AND assigned_to NOT IN (
    SELECT user
    FROM sys_user_grmember
    WHERE group.name = 'Network'
  )
```

See [WHERE Subqueries](#where-subqueries-semi-join--anti-join) for the two-phase execution model.

### Level 5 — Inner (nested) SELECTs

Fetch related child records inline with each parent — each parent row gets a cell containing its related records, with their own count and modal.

```sql
SELECT number, short_description,
  (SELECT number, state, short_description
   FROM incident_task
   WHERE active = true
   ORDER BY number DESC)
FROM incident
WHERE priority = 1
LIMIT 50
```

Multiple inner queries, each with its own alias (`AS` goes immediately after the inner table name):

```sql
SELECT number,
  (SELECT number, state FROM incident_task AS tasks
   WHERE active = true),
  (SELECT file_name, size_bytes FROM sys_attachment AS attachments)
FROM incident
WHERE active = true
LIMIT 50
```

See [Inner (Nested) Queries](#inner-nested-queries) for the per-parent row cap, the `Counts` toggle (server-side aggregate counts per parent), and the truncation banner.

### Level 6 — DISTINCT and COUNT

**DISTINCT** — return only unique combinations of the selected fields (deduplicated client-side; see [SELECT DISTINCT](#select-distinct)):

```sql
SELECT DISTINCT assignment_group.name, caller_id.department
FROM incident
WHERE active = true
LIMIT 1000
```

**COUNT** — fast row count without fetching rows, via ServiceNow's aggregate API (see [SELECT COUNT](#select-count)):

```sql
SELECT COUNT
FROM incident
WHERE active = true
  AND priority IN (1, 2)
```

DISTINCT and WHERE subqueries also compose:

```sql
SELECT DISTINCT caller_id.name
FROM incident
WHERE active = true
  AND assignment_group IN (
    SELECT group FROM sys_group_has_role WHERE role.name = 'itil'
  )
```

### Level 7 — CMDB relationship queries

When the **CMDB** toggle is on and the table is a `cmdb_ci*` descendant, an extra Relations column appears. Use `WITH CMDB_FIELDS(...)` to control which fields show up for related CIs in the relationship modal:

```sql
SELECT sys_id, name, sys_class_name
FROM cmdb_ci_server
WHERE operational_status = 1
WITH CMDB_FIELDS(install_status, model_id.name, location.name)
LIMIT 100
```

See [CMDB Relationship Queries](#cmdb-relationship-queries) for badge semantics and the diagram view.

### Putting it all together

A realistic "find me the active high-priority outages handled by the on-call group, with their recent tasks" query stacks most of the above:

```sql
SELECT DISTINCT caller_id.name, caller_id.department, assignment_group.name,
  (SELECT number, state, sys_updated_on
   FROM incident_task AS recent_tasks
   ORDER BY sys_updated_on DESC)
FROM incident
WHERE active = true
  AND priority IN (1, 2)
  AND (short_description LIKE '%outage%'
       OR short_description LIKE '%down%')
  AND assigned_to IN (
    SELECT user
    FROM sys_user_grmember
    WHERE group.name = 'Network Operations'
  )
ORDER BY sys_created_on DESC
LIMIT 200
```

This single query exercises: DISTINCT, three-segment dot-walking, grouped AND/OR, two `LIKE` variants, an `IN` literal list, an `IN` subquery (semi-join), an inner `SELECT` with its own `ORDER BY` and alias, plus an outer `ORDER BY` and `LIMIT`.

---

## Inner (Nested) Queries

Inner queries allow you to fetch related child records alongside the parent query results — similar to a SQL subquery or JOIN.

### Inner Query Syntax

Place a nested `SELECT` inside the outer `SELECT` clause, wrapped in parentheses:

```sql
SELECT number, short_description,
  (SELECT number, state FROM incident_task)
FROM incident
WHERE active = true
LIMIT 50
```

You can alias the nested result with `AS`:

```sql
SELECT number,
  (SELECT number, state FROM incident_task AS tasks),
  (SELECT name FROM sys_attachment AS attachments)
FROM incident
WHERE priority = 1
LIMIT 20
```

Inner queries also support their own `WHERE`, `ORDER BY`, and `LIMIT` clauses:

```sql
SELECT number,
  (SELECT number, state FROM incident_task WHERE active = true ORDER BY number DESC)
FROM incident
LIMIT 50
```

### How Inner Queries Work

1. The outer query executes first, fetching the parent records
2. For each inner query, the extension automatically resolves the relationship between the child table and the parent table (via reference fields in `sys_dictionary`)
3. One bounded query is fired per parent (`refField = <parent_id> LIMIT N+1`) at concurrency 10 — each call is small and reliable
4. The progress modal tracks each phase: "Fetching parent records..." then "Fetching inner query 'tasks'... (X/Y parents)"

### Per-Parent Row Cap

When no inner `LIMIT` is specified, the extension caps each parent's children at **200 rows** by default (matching Salesforce SOQL subquery behaviour). This prevents a single high-volume parent from dominating the response and keeps the result set manageable.

When a parent has more than 200 matching children:

- The cell is highlighted in **amber** with a **`200+ rows`** label instead of `200 rows`
- A warning toast appears after execution naming the truncated parents

To fetch more for a specific parent, either:
- Add an explicit inner `LIMIT` — e.g. `(SELECT sys_id FROM incident LIMIT 1000)` — overrides the default cap, no warning
- Use the **Counts** toggle (below) to see the real totals server-side

### Counts Toggle

After executing a query with inner SELECTs, a **`Counts`** button appears in the results action row. Click it to switch inner columns from records to **server-side aggregate counts**.

- Each inner cell becomes a plain number — the exact total per parent (e.g. `3,900`), regardless of the 200-row cap
- Counts are sortable: click the column header to find the parents with the most (or fewest) related records
- Counts are filterable: type `>= 100` in the column filter to show only parents with at least 100 children
- Re-click **Counts** to return to records mode

Useful when you want to know *how many* related records each parent has rather than seeing the records themselves — for example, "which user groups have the most assigned incidents?"

### Nested Results Modal

Inner query results appear as a clickable badge in the results grid showing the count of related records (e.g. **"3 records"**). Clicking the badge opens a full modal showing the nested records in a sortable, filterable table.

From the nested results modal, you can:
- Sort and filter columns
- Right-click to open a record in ServiceNow or view full record details

---

## WHERE Subqueries (Semi-join / Anti-join)

Use a `SELECT` subquery inside `IN` or `NOT IN` in the `WHERE` clause to filter parent records based on the existence (or absence) of matching child records.

### Semi-join — "has at least one child matching…"

Return only parent records whose ID appears in the child query results:

```sql
SELECT user.name, group.name
FROM sys_user_grmember
WHERE user.active = true
AND group IN (
    SELECT group
    FROM sys_group_has_role
    WHERE role.name = 'itil'
)
```

This returns group members where the group has the "itil" role assigned.

### Anti-join — "has no children matching…"

Return only parent records whose ID does *not* appear in the child query results:

```sql
SELECT number, short_description
FROM incident
WHERE active = true
AND assigned_to NOT IN (
    SELECT user
    FROM sys_user_grmember
    WHERE group.name = 'Network'
)
```

This returns active incidents where the assignee is not a member of the "Network" group.

### How It Works

Since ServiceNow's Table API does not support subqueries in encoded queries, the extension uses a **two-phase execution**:

1. **Phase 1 — Subquery:** The inner `SELECT` is executed first. The values of the specified field (e.g. `group` sys_ids) are collected from the results.
2. **Phase 2 — Main query:** The collected values are substituted into the outer query as a flat comma-separated list (e.g. `groupINid1,id2,id3,...`), and the main query executes normally.

A progress modal tracks the subquery resolution phase before the main query begins.

### Subquery Syntax

The inner `SELECT` supports the same clauses as regular queries:

```sql
field IN (SELECT return_field FROM child_table WHERE conditions)
field NOT IN (SELECT return_field FROM child_table WHERE conditions)
```

- The first field in the inner `SELECT` is the one whose values are collected
- `WHERE`, `ORDER BY`, and `LIMIT` are all supported in the inner query
- Multiple subqueries can be used in the same `WHERE` clause

### Combining with DISTINCT

WHERE subqueries and `DISTINCT` work together:

```sql
SELECT DISTINCT caller_id.name
FROM incident
WHERE active = true
AND assignment_group IN (
    SELECT group
    FROM sys_group_has_role
    WHERE role.name = 'itil'
)
```

The subquery resolves first, then the main query fetches records, and finally DISTINCT deduplicates the results.

---

## CMDB Relationship Queries

The extension can discover and display **CMDB relationships** (`cmdb_rel_ci`) for any `cmdb_ci*` table. This reveals how Configuration Items (CIs) are connected to each other through ServiceNow's CMDB relationship model — something not visible through normal reference fields.

### Enabling CMDB Mode

Toggle the **CMDB** checkbox in the editor toolbar. When enabled and querying a `cmdb_ci*` table, an extra "CMDB Relations" column appears in the results.

### CMDB Relations Badge

Each row gets a colour-coded badge:

- **P:N** (green) — this CI is **parent of** N other CIs
- **C:M** (amber) — this CI is **child of** M other CIs
- A dash (**—**) if no relationships exist

Hovering over the badge shows a tooltip with relationship counts and the related class types.

### CMDB Relationship Modal

Clicking the badge opens a detailed modal with two sections:

- **Parent of** (green) — CIs this record is a parent of, grouped by relationship type
- **Child of** (amber) — CIs this record is a child of, grouped by relationship type

Each section displays a sortable, filterable table with columns for Name, Class, custom fields (if specified), and Relationship Type. The class column shows both the human-readable label and the internal table name. Names link directly to the record in ServiceNow.

Right-clicking any row provides:
- **Open in ServiceNow** — direct link to the record
- **View Record Details** — full record inspection

### WITH CMDB_FIELDS Syntax

Control which fields are displayed in the CMDB Relationship Modal for related CIs:

```sql
SELECT sys_id, name FROM cmdb_ci_ip_pool
WITH CMDB_FIELDS(install_status, operational_status)
LIMIT 100
```

- `name` and `sys_class_name` are always included
- Fields in `CMDB_FIELDS(...)` are added between the defaults and `rel_type`
- If omitted, defaults to: Name, Class, Relationship Type
- Autocomplete suggests base `cmdb_ci` fields when the cursor is inside `CMDB_FIELDS(...)`

#### Authoritative Counts + Per-CI Fetch + Drill-Down

The `P:N C:M` badge counts come from ServiceNow's `/api/now/stats/cmdb_rel_ci` aggregate API, so they always reflect the **true** number of direct parent/child relationships even when the row fetch is capped.

**Per-CI fetch with adaptive cap.** Row data is fetched one query per CI per direction (concurrency 10), with the per-CI row cap adapting to outer-query size:
- 1–2 CIs in the result → cap 1000 per CI
- 3–10 CIs → cap 500
- 11+ CIs → cap 200 (fairness — every CI gets useful data even on wide pages)

Cells with truncated row data show a small amber dot on the badge; the tooltip and modal banner show `Showing X of Y (truncated)`. The modal always renders a section as long as the count for that direction is > 0, even if zero rows came back (so you never see a "missing" Parent of / Child of section for a heavily-connected CI).

**Drill-down: Fetch all.** Inside the modal (and on the diagram's focused node), each truncated section shows a `Fetch all N →` button. Clicking it fires an unbounded per-CI fetch (safety cap 25,000), mutates the relationship list in place, clears the truncation flag, and re-renders. A confirm prompt appears when `count > 5000`.

CMDB relationships are direct only — `cmdb_rel_ci` stores one-hop edges, so the counts you see reflect direct parents/children, not transitive closure.

#### Diagram Column Filter (iter-15)

Each class column in the CMDB diagram has a filter input at the top: type any substring and the visible items narrow live (case-insensitive match on name). The column shows every loaded item in a scrollable list — no "Load more" pagination. Filter state persists across "Fetch all" re-renders so you don't lose your search when expanding the dataset; it resets when you navigate to a different focused CI.

### CMDB + Inner Queries

When CMDB mode is on and an inner query also targets a `cmdb_ci*` table, the nested modal includes CMDB relationship badges for those child records too. Clicking them opens the same CMDB modal. CMDB data for nested records is lazy-loaded only when you open the nested modal.

---

## CMDB Relationship Diagram

An interactive visual graph for exploring CMDB relationships. Access it via the **"View Diagram"** button in the CMDB Relationship Modal.

### Grouped Column Layout

Related CIs are **grouped by class** (table) into vertical columns. Each column shows:

- A **header** with the class name and a badge showing the total count of CIs of that class
- Individual CI nodes listed vertically below
- **Pagination** — the first 20 items per column are displayed initially; a "Load more" button at the bottom loads the next 20, showing how many remain

This grouping ensures that all distinct classes are visible at a glance, even when one class has hundreds of relationships. Columns are ordered by count (largest first) and scroll horizontally when there are many class groups.

### Diagram Sections

The layout from top to bottom:

1. **Grandchild of** (purple) — parents of parents (second-level hierarchy). Each grandchild-of column is connected via purple SVG lines to the specific parent column it came through, making lineage immediately clear. Each node shows an italic "via ParentName" label.
2. **Child of** (amber) — direct parents of the focused CI, connected to the focus node via amber bezier curves
3. **Focused CI** (centre, blue glow) — shows name, table name, and display class
4. **Parent of** (green) — direct children of the focused CI, connected via green bezier curves

The grandchild-of level is **optional** — controlled by a toggle in Settings ("Show additional parent level in diagram"). When disabled, only the single-level parent/child layout is shown and no extra API calls are made.

### Interactive Navigation

- **Click** any connected node to navigate to it — its relationships are fetched on demand (lazy-loading)
- A **breadcrumb trail** tracks your navigation path; click any crumb to jump back
- A **Back button** returns to the previous CI

### Context Menus

**Right-click any node** for:
- **Expand relationships** — navigate to that CI
- **Create query in new tab** — generates a `SELECT sys_id, name, short_description FROM <table> WHERE sys_id = <id> LIMIT 1` query in a new tab
- **Open in ServiceNow** — opens the record directly

**Right-click a column header** for:
- **Query all N records** — opens a new tab with a `SELECT ... WHERE sys_id IN (...)` query for every CI in that class group
- **Open list in ServiceNow** — opens the class table list view directly

---

## Results Grid

### Column Sorting

Click any column header to sort the results. Click again to toggle between ascending and descending order. Sorting uses natural ordering so that numeric values sort correctly (1, 2, 10 instead of 1, 10, 2).

### Column Filtering

Each column has a filter input below the header. Type to filter rows that contain the entered text in that column. Multiple column filters are applied together (AND logic). The row count updates to show "N rows (filtered from M)" when filters are active.

### Column Resizing

Drag the edge of any column header to resize it. Column widths are preserved as you interact with the results.

### Reference Field Links

Fields that reference other tables (e.g. `assigned_to`, `cmdb_ci`) are displayed as clickable links. Clicking a reference link opens the referenced record directly in ServiceNow, using the actual target table name for the URL.

### Row Selection & Shift-Click

- Click the checkbox on any row to select it
- Use **Shift+Click** to select a range of rows between the last selected and the current one
- Use the header checkbox to select/deselect all rows
- Selected rows enable the "Delete Selected" button

---

## Record Management

### Inline Row Editing

Double-click any editable cell in the results grid to edit its value directly. Press **Enter** to save or **Escape** to cancel. The updated value is sent to ServiceNow immediately via the REST API. System fields (`sys_id`, `sys_created_on`, `sys_updated_by`, etc.) are not editable.

### View Record Details

Right-click any row and select **"View Record Details"** to open a modal that fetches and displays **all fields** for that record from ServiceNow — not just the columns in your query. Fields are presented in a structured list showing the field name, label, and value.

### Add Fields to Query from Record Details

When viewing record details, clicking on a field name adds it to your query. The behaviour is context-aware:

| Opened from | Field added to |
|---|---|
| Outer query record | Outer `SELECT` clause |
| Inner query record | Inner `SELECT` clause |
| CMDB modal record | Outer `WITH CMDB_FIELDS(...)` clause |

### Delete Records

- **Single record:** Right-click a row → "Delete Record" (with confirmation)
- **Multiple records:** Select rows using checkboxes, then click "Delete Selected". A confirmation modal lists all records to be deleted, and they are removed via the ServiceNow API

---

## Tabs

The extension supports multiple query tabs, allowing you to work on several queries simultaneously.

### Rename, Pin, Color & Duplicate

Right-click any tab to access the context menu:

- **Rename** — double-click the tab title or use the context menu
- **Pin** — pinned tabs cannot be closed or reordered; shows a pin icon
- **Set Color** — choose from blue, green, orange, purple, red, yellow, or none. The tab header reflects the selected colour
- **Duplicate** — creates a new tab with the same query content
- **Close** — closes the tab (prevented for pinned tabs or the last remaining tab)

### Drag & Drop Reordering

Drag tabs to reorder them. Pinned tabs cannot be moved.

---

## Query History

Click the **History** button in the toolbar to open the history sidebar. Every executed query is automatically saved with:

- The query text
- Timestamp
- Instance name
- Record count returned

From the history sidebar you can:
- **Rerun** a query (loads it into the editor and executes)
- **Copy** the query text to clipboard
- **Delete** individual history entries
- **Clear** all history
- **Expand** any entry to view the full query text

History stores up to 100 entries and persists across browser sessions.

---

## Saved Queries

### Save, Edit & Organize in Folders

Click the **Saved Queries** button to open the saved queries sidebar. Save the current query with a name, folder, and optional tags.

Pre-configured folders include Default, Production, Development, and Testing — and you can create custom folders. Filter saved queries by folder and/or search text.

### Run, Apply & Copy

Each saved query card provides:

- **Run** (play button) — opens the query in a new tab and executes it immediately
- **Apply** (plus icon) — loads the query into the current tab without executing
- **Copy** — copies the query text to clipboard
- **Edit** — opens the save modal to update name, folder, tags, or the query itself
- **Delete** — removes the saved query (with confirmation)
- **Click the card** — shows a full-text preview of the query

### Save / Load All Open Queries

- **Save All Open Queries** — saves every open tab's query into a single folder (you choose the folder name). Tab names and colours are preserved
- **Load All from Folder** — opens every query from a selected folder, each in its own tab, restoring tab names and colours
- **Delete Folder** — deletes a folder and all queries within it

This is useful for saving and restoring entire workspaces of related queries.

### Export & Import

- **Export** — downloads all saved queries and folders as a JSON file
- **Import** — uploads a previously exported JSON file. Duplicate queries (by name) are skipped; new folders are created automatically

---

## Copy Query

The **Copy Query** dropdown in the toolbar provides two options:

### Copy Encoded Query

Copies the ServiceNow encoded query string (the `WHERE` + `ORDER BY` portion translated to ServiceNow syntax). This is useful for:

- Pasting into `sysparm_query` URL parameters
- Using with `GlideRecord.addEncodedQuery()`
- Sharing filter conditions with colleagues

### Copy GlideRecord Script

Generates and copies a complete GlideRecord script snippet based on your current query:

```javascript
var gr = new GlideRecord('incident');
gr.addEncodedQuery('active=true^priority=1');
gr.query();
while (gr.next()) {
  gs.info(gr.number + ' ' + gr.short_description);
}
```

The script includes the table name, encoded query, and uses the first few fields from your `SELECT` for the output line.

---

## Table Information Panel

Click the **Table Info** button after executing a query to open the Table Information panel for the current table. It displays:

### Inheritance Hierarchy

Shows the full table inheritance chain (e.g. `incident` → extends `task` → extends base). Useful for understanding which fields are inherited.

### Outgoing References

Lists all reference fields on the table and their target tables (e.g. `assigned_to → sys_user`, `cmdb_ci → cmdb_ci`).

### Incoming References

Lists all tables and fields that reference the current table (e.g. `incident_task.parent → incident`). Internal ServiceNow variable tables (Integration Hub, Flow Designer) are filtered out for clarity.

### Open in ServiceNow

A direct link to the table's `sys_db_object` record in your ServiceNow instance.

### Summary

A quick count of inheritance levels, outgoing references, and incoming references.

---

## Export Results

### Export to CSV

Click **Export CSV** to download the current results as a CSV file. Nested (inner query) columns export the related records' `sys_id` values as a semicolon-separated list. The CMDB Relations column is excluded from CSV exports as it is an interactive UI feature.

### Copy Results to Clipboard

Click **Copy** to copy all results to the clipboard in CSV format, ready to paste into a spreadsheet.

---

## Themes

Open **Settings** and choose from multiple colour themes to customise the extension's appearance. The selected theme persists across sessions.

---

## Settings

Access settings via the gear icon:

- **Theme picker** — switch between visual themes
- **CMDB Diagram** — toggle "Show additional parent level (grandchild-of) in diagram" on or off. When on (default), the diagram fetches and displays a second level of parent hierarchy. When off, only the direct parent/child level is shown. This setting persists across sessions
- **Reset Table Cache** — clears cached table metadata and reloads the extension (useful if table structures have changed)
- **Fetch All Fields** — manually fetches and caches all fields for a specified table

---

## Flow Inspector

A dedicated tool for discovering and exploring flows, scripts, and UI configurations across a ServiceNow instance. Access it from the **app switcher** dropdown in the header.

### Search Modes

Find records using three lookup modes, selectable from the dropdown:

| Mode | Description |
|------|-------------|
| **Trigger table** | Finds flows triggered by a specific table (e.g. `incident`), plus Business Rules, Client Scripts, UI Actions, and UI Policies that operate on that table |
| **Scope** | Finds all records belonging to a specific application scope |
| **Name** | Searches by name across all enabled record types (minimum 2 characters) |

An **Auto-detect mode** toggle infers whether input looks like a table API name or a record name and switches the lookup mode automatically.

### Supported Record Types

The inspector searches across 10 record types spanning three groups:

| Group | Types |
|-------|-------|
| **Flow** | Flows, Subflows, Actions |
| **Script** | Business Rules, Script Includes, Scheduled Jobs, Script Actions |
| **UI** | Client Scripts, UI Actions, UI Policies |

### Pre-Search Type Toggle

A row of toggle pills below the lookup bar controls which record types are included in the search. This determines which API calls are made — selecting fewer types means faster, more focused results.

- **All** — searches every type; only the "All" pill is highlighted
- **Individual pills** — click to narrow the search to specific types; pills are colour-coded by group (green for flows, teal for scripts, indigo for UI)
- At least one type must remain selected
- The selection persists across sessions via browser storage

### Post-Search Filters

After results load, client-side filters refine the displayed items without additional API calls:

- **Type** — filter by record type (only types present in results are shown, with counts)
- **Status** — All / Active / Inactive
- **Scope** — filter by application scope
- **Trigger** / **Table** — filter by trigger type or table (when applicable)
- **Sort** — Name A–Z, Name Z–A, Recently Modified, Oldest Modified, Type
- **Search** — free-text filter across names, descriptions, scopes, and type-specific fields (applied on Enter)

### Result Cards

Each record is displayed as a card showing:

- Record name, type badge (colour-coded), and active/inactive status
- Type-specific secondary info (e.g. "When: before · Table: incident" for Business Rules, "Event: incident.assigned" for Script Actions)
- Scope, creation/update dates, and author
- Direct link to open the record in ServiceNow

### Detail Views

Clicking a card opens a detail view. Flows, subflows, and actions get a **rich detail view** with trigger information, action steps, "Used in this flow" / "Used by" cross-references, and an "Export for AI" feature. New record types (Business Rules, Script Includes, etc.) get a **lightweight detail view** showing type-specific fields, metadata, and an "Open in ServiceNow" link.

### Navigation

Detail views support breadcrumb navigation — clicking a referenced subflow or action opens its detail view, and the breadcrumb trail lets you navigate back through the chain.

---

## Log Explorer

A dashboard-style mini application for browsing ServiceNow system logs. Access it from the **app switcher** dropdown in the header.

### Log Sources

Switch between six ServiceNow log tables using the source dropdown:

| Source | Table | What It Shows |
|--------|-------|---------------|
| **System Log** | `syslog` | Platform logs — Trace, Debug, Info, Warning, Error, Fatal messages |
| **Transaction Log** | `syslog_transaction` | REST/SOAP calls, scheduler runs, response times |
| **Events** | `sysevent` | System events (flow triggers, scheduled jobs, etc.) |
| **Audit Log** | `sys_audit` | Record changes — who changed what field, old vs. new value |
| **Email Log** | `sys_email` | Inbound and outbound email records |
| **Outbound HTTP** | `sys_outbound_http_log` | Outbound REST/SOAP calls and scripted HTTP requests to external systems |

Each source has pre-configured default columns appropriate to its data.

### Time Range

Select a time window from the dropdown: **Last 15 min**, **30 min**, **1 hour** (default), **6 hours**, **24 hours**, or **Custom range…**. Narrower ranges are faster because ServiceNow log tables use rotating shards — querying "Last 1 hour" typically hits only 1–2 shards instead of all 8.

#### Custom Range

Choosing **Custom range…** reveals two `datetime-local` pickers (**From** / **To**) and an **Apply** button. Pick any window down to the second; the explorer converts the values into `javascript:gs.dateGenerate(...)` bounds on both sides of the source's time field. Bounds that are empty or inverted (From > To) are ignored so the rest of the query still runs cleanly. The custom From/To values are preserved across reloads.

### Level Filter Cards

When viewing System Log, colour-coded cards at the top show the count of records per severity level — **Trace**, **Debug**, **Info**, **Warning**, **Error**, and **Fatal** — matching ServiceNow's actual `syslog.level` dictionary values. Click any card to filter the log stream to that level. Multiple levels can be toggled independently.

The **"All" button** controls the fetch mode:

- **All ON** (default) — Refresh fetches all log types. Clicking individual level cards applies instant client-side filtering on the loaded data.
- **All OFF** — Refresh fetches only the selected log types from the server (e.g. only Info logs). This is useful on high-volume instances where mixed log types fill the fetch limit within seconds, making it hard to see specific log types over a meaningful time range.

### Search & Filter

- **Search** — type in the search bar to filter across all field values (client-side, instant)
- **Source dropdown** — filter by the log source component (e.g. `com.glide.ui.ServletErrorListener`)
- **User dropdown** — filter by the user who triggered the log entry

All filters are applied together (AND logic) and operate entirely client-side on the loaded records — no additional API calls.

### Log Stream

Records are displayed in a grid with a sticky column header. Columns are automatically sized based on content width — short fields like time and level stay compact, while longer fields like message expand to fill available space.

- **Click any row** to expand it, revealing all field values in a detail panel. Only one row is expanded at a time; opening a new one collapses the previous
- **Level badge** — System Log rows render the level as a colour-coded badge (Trace/Debug/Info in blue, Warning in amber, Error in red, Fatal in deep red). Levels are mapped client-side from `syslog.level`'s numeric values
- **Cell truncation** — long values (e.g. messages) are truncated to ~200 characters in the grid for performance; the full value is shown on hover via the cell's tooltip and in the expanded detail panel
- **Copy button** in the expanded panel copies all field labels and values to the clipboard, each on a new line. The button flashes a "Copied" confirmation
- **Column resize** — drag the right edge of any column header to adjust its width. Per-column widths persist for the lifetime of the explorer instance
- **Load More** — appears when the server returned a full batch, indicating more records may exist. Click to fetch the next page (using the same source, time range, level, and filters) and append it to the stream. Shows "Loading..." feedback during the fetch

### Live Mode

Toggle the **Live** switch in the toolbar to enable automatic refreshing. When active, the Log Explorer fetches new records every 10 seconds and prepends them to the stream. A pulsing green indicator shows that live mode is active. Live mode stops automatically when switching to another app or changing the log source.

### Configure Fields & Fetch Limit

Click the **Fields** button to open the configuration modal:

- **Fields** — enter comma-separated field names to customise which columns are displayed. Each log source remembers its own field configuration independently. `sys_id` is always included even if you omit it from the list (it's needed for row identity)
- **Records to fetch** — set how many records to load per batch (default 350, range 50–5000)
- **Reset to Default** — restores the source's default fields and the 350 record limit

### Persisted Preferences

The following settings survive across browser sessions via `chrome.storage.local`:

- The selected **log source**
- The current **time range** (including the Custom range From/To values)
- Per-source **field overrides** entered in the Fields modal
- The **records-to-fetch** limit

Stats counts, search text, level toggles, and column widths are intentionally **not** persisted — each new session starts fresh on filters but with your chosen workspace shape.

### Performance

The Log Explorer uses optimised API calls compared to the standard Query Editor:

- Skips display value resolution (`sysparm_display_value=false`) — level names like "Error" are mapped client-side
- Skips the separate count query (`sysparm_no_count=true`)
- Requests only the fields needed (`sysparm_fields` is always explicit)
- Excludes reference link metadata (`sysparm_exclude_reference_link=true`)

A toast notification after each fetch shows the number of records and time taken (e.g. "Fetched 350 records in 2140ms").

---

## Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl+Enter` / `Cmd+Enter` | Execute query |
| `Escape` | Close topmost modal / cancel edit / dismiss autocomplete |
| `Tab` or `Enter` | Accept autocomplete suggestion |
| `Up / Down` arrows | Navigate autocomplete suggestions |
| `Shift+Click` (checkbox) | Select range of rows |
| `Double-click` (cell) | Start inline editing |

---

## Modal Stack (ESC Handling)

Multiple modals can be open simultaneously (e.g. nested query modal → CMDB modal → diagram → record details). Pressing **Escape** always closes only the topmost modal, working your way back through the stack naturally.

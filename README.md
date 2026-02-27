# ServiceNow Query Editor — Chrome Extension

A powerful Chrome extension that provides a SQL-like query interface for ServiceNow. Write queries against any ServiceNow table, explore CMDB relationships visually, manage saved queries in folders, edit records inline, and much more — all without leaving your browser.

---

## Table of Contents

- [Getting Started](#getting-started)
- [Writing & Executing Queries](#writing--executing-queries)
- [Inner (Nested) Queries](#inner-nested-queries)
- [CMDB Relationship Queries](#cmdb-relationship-queries)
- [CMDB Relationship Diagram](#cmdb-relationship-diagram)
- [Results Grid](#results-grid)
- [Record Management](#record-management)
- [Tabs](#tabs)
- [Query History](#query-history)
- [Saved Queries](#saved-queries)
- [Copy Query](#copy-query)
- [Table Information Panel](#table-information-panel)
- [Condition Builder](#condition-builder)
- [Export Results](#export-results)
- [Themes](#themes)
- [Settings](#settings)
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
| `NOT IN (a, b, c)` | Value not in list | `NOT IN` | `state NOT IN (6, 7)` |

Conditions can be combined with `AND` / `OR` and grouped using parentheses.

### SELECT COUNT

Use `SELECT COUNT FROM table` (or `SELECT COUNT(*) FROM table`) to get a fast record count without fetching any rows:

```sql
SELECT COUNT FROM incident WHERE active = true
```

This uses ServiceNow's aggregate API for performance.

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
3. Child records are batch-fetched using `IN` queries on the reference field, ensuring efficient API usage
4. The progress modal tracks each phase: "Fetching parent records..." then "Fetching inner query 'tasks'..."

### Nested Results Modal

Inner query results appear as a clickable badge in the results grid showing the count of related records (e.g. **"3 records"**). Clicking the badge opens a full modal showing the nested records in a sortable, filterable table.

From the nested results modal, you can:
- Sort and filter columns
- Right-click to open a record in ServiceNow or view full record details

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

### CMDB + Inner Queries

When CMDB mode is on and an inner query also targets a `cmdb_ci*` table, the nested modal includes CMDB relationship badges for those child records too. Clicking them opens the same CMDB modal. CMDB data for nested records is lazy-loaded only when you open the nested modal.

---

## CMDB Relationship Diagram

An interactive visual graph for exploring CMDB relationships. Access it via the **"View Diagram"** button in the CMDB Relationship Modal.

The diagram displays:
- The **focused CI** in the centre
- **Parent CIs** above (amber border) with connecting bezier curves
- **Child CIs** below (green border) with connecting bezier curves
- Each node shows: CI Name, internal table name, display class name, and relationship type

### Interactive Navigation

- **Click** any connected node to navigate to it — its relationships are fetched on demand (lazy-loading)
- A **breadcrumb trail** tracks your navigation path; click any crumb to jump back
- A **Back button** returns to the previous CI
- Up to 9 nodes per row; overflow is indicated with a "+N more" badge
- The diagram adapts node widths and re-renders on window resize

### Diagram Context Menu & Actions

Right-click any node for:
- **Expand relationships** — navigate to that CI
- **Create query in new tab** — generates a `SELECT * FROM <table> WHERE sys_id = <id>` query in a new tab
- **Open in ServiceNow** — opens the record directly

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

## Condition Builder

Click the **Condition Builder** button to open a visual query builder sidebar. Construct queries without writing SQL:

1. Enter a **table name**
2. Specify **fields** to return (or `*`)
3. Add **conditions** — each row has a field, operator, and value
4. Combine conditions with **AND** / **OR**
5. Set **ORDER BY** and **LIMIT**
6. Click **Generate Query** to insert the constructed query into the editor

The available operators include `=`, `!=`, `>`, `<`, `>=`, `<=`, `LIKE`, `IN`, and `NOT IN`.

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
- **Reset Table Cache** — clears cached table metadata and reloads the extension (useful if table structures have changed)
- **Fetch All Fields** — manually fetches and caches all fields for a specified table

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

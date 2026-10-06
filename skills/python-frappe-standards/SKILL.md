---
name: python-frappe-standards
description: Python coding standards for Frappe/ERPNext Doctype controllers, DocEvent overrides, and whitelisted APIs: section order, lifecycle order, helpers, translations, SQL, tabs. Use when writing, editing or reviewing any .py file in this app.
---

# Python Frappe Doctype Coding Standards

This document defines standard conventions for writing Python Doctype classes in Frappe. The goal is to ensure consistency, readability, maintainability, and predictable execution flow across all doctypes.

## 1. General Principles

- Every Doctype class must follow Frappe's standard execution lifecycle.
- Controller (lifecycle) methods must always be written first in the class.
- Controller methods must be:
  - Grouped together
  - Ordered according to actual execution flow
  - Clearly separated using section comments
- All helper/utility methods must:
  - Be grouped under the controller that uses them
  - Follow the same top-down execution order

## 2. Mandatory Section Order Inside a Doctype Class

Every Doctype class must follow this exact structure:

```python
class <DoctypeName>(Document):
    # --- Controller Methods ---
    <lifecycle methods in order>

    # --- <Controller Name> Utils ---
    <private helper methods>

    # --- Whitelisted Methods ---
    <@frappe.whitelist() methods>
```

No utility methods should appear before controller methods.

## 3. Controller Methods — Execution Order

Controller methods must be written in the same order as Frappe executes them.

### 3.1 Save Lifecycle (Insert & Update)

**When inserting a new document (Save)**

Before Save Methods (`run_before_save_methods`):
1. `before_insert`
2. `before_validate`
3. `validate`
4. `before_save`

After Save Methods (`run_post_save_methods`):
5. `after_insert`
6. `on_update`
7. `on_change`

**When updating an existing document (Save)**

Before Save Methods (`run_before_save_methods`):
1. `before_validate`
2. `validate`
3. `before_save`

After Save Methods (`run_post_save_methods`):
4. `on_update`
5. `on_change`

### 3.2 Submit Lifecycle

Before Save Methods (`run_before_save_methods`):
1. `before_validate`
2. `validate`
3. `before_submit`

After Save Methods (`run_post_save_methods`):
4. `on_update`
5. `on_submit`
6. `on_change`

### 3.3 Cancel Lifecycle

Before Save Methods (`run_before_save_methods`):
1. `before_cancel`

After Save Methods (`run_post_save_methods`):
2. `on_cancel`
3. `on_change`

### 3.4 Update After Submit Lifecycle

Before Save Methods (`run_before_save_methods`):
1. `before_update_after_submit`

After Save Methods (`run_post_save_methods`):
2. `on_update_after_submit`
3. `on_change`

## 4. Utility (Helper) Method Rules

### 4.1 Naming

Helper methods should be **private** (prefixed with `_`) when they are:
- Only used internally within the doctype
- Not intended to be called from outside the class

Helper methods can be **public** (no `_` prefix) when they need to be:
- Called from client-side JavaScript
- Used by other modules/doctypes
- Exposed as whitelisted methods (`@frappe.whitelist()`)

### 4.2 Placement

Utility methods must:
- Appear after controller methods
- Be grouped by controller
- Follow the same execution sequence used in the controller

### 4.3 Whitelisted Methods

Whitelisted methods (decorated with `@frappe.whitelist()`) must:
- Be defined at the end of the class
- Appear after all controller methods and their utilities
- Be grouped together in a dedicated section with comment `# --- Whitelisted Methods ---`

## 5. Example — Correctly Structured Doctype

```python
class PurchaseOrderAmendment(Document):
    # --- Controller Methods ---
    def before_validate(self):
        self._set_purchase_order_doc()
        self._update_purchase_order_detail()
        self._update_total()

    def validate(self):
        self._prepare_changes()
        self._apply_changes_to_purchase_order()
        self._finalize_purchase_order()

    # --- Before Validate Utils ---
    def _set_purchase_order_doc(self):
        self.purchase_order_doc = frappe.get_doc(PURCHASE_ORDER, self.purchase_order)

    def _update_purchase_order_detail(self):
        self.is_subcontracted = self.purchase_order_doc.is_subcontracted
        self.is_old_subcontracting_flow = self.purchase_order_doc.is_old_subcontracting_flow

    def _update_total(self):
        self.total = sum(
            flt(getattr(item, "rate", 0)) * flt(getattr(item, "qty", 0))
            for item in self.items
        )

    # --- Validate Utils ---
    def _prepeare_changes(self):
        pass

    def _apply_changes_to_purchase_order(self):
        pass

    # --- Whitelisted Methods ---
    @frappe.whitelist()
    def get_purchase_order_details(self):
        return frappe.get_doc(PURCHASE_ORDER, self.purchase_order)
```

## 6. Additional Best Practices

### 6.1 Keep Functions Small and Focused

- Functions should do one thing and only one thing
- Break functions longer than 10 lines into smaller helper functions
- Multiple levels of indentation indicate the function is doing too many things
- "Paragraph breaks" (whitespace between logical groups) suggest the function should be split

### 6.2 User-Facing Messages Must Be Translated

- All user-facing strings must be wrapped in the `_("")` function in Python
- Use `__("")` function in JavaScript for translations
- Example: `frappe.throw(_("Cannot update a cancelled Material Request."))`

### 6.3 SQL Best Practices

Prefer Frappe APIs over raw SQL:
- Use `frappe.qb` (Query Builder)
- Use `frappe.get_all()` for queries
- Use `frappe.db.get_value()` for single values

**One query per row:** When you need several fields from the same row, fetch them in one `frappe.db.get_value()` call. Pass a list of fieldnames. Do not call `get_value` once per field.

- ❌ Wrong: two round trips to the same row:

```python
status = frappe.db.get_value("Sales Order", name, "status")
...
if frappe.db.get_value("Sales Order", name, "per_billed"):
```

- ✅ Correct: one call, unpacked into names:

```python
status, per_billed = frappe.db.get_value("Sales Order", name, ["status", "per_billed"])
```

- Use `as_dict=True` when you read more than two or three fields.
- This also applies when the two reads are far apart in the function. Move the read to the top and reuse the values.

**SQL Injection Protection:** Always use parameterized queries

- ✅ Correct: `frappe.db.sql('SELECT age FROM tabUser WHERE name=%s', user)`
- ❌ Wrong: `frappe.db.sql('SELECT age FROM tabUser WHERE name={}'.format(user))`

**SQL Formatting:** Use consistent indentation for readability

```python
# Good formatting
result = frappe.db.sql("""
    SELECT
        item_name, description, default_warehouse
    FROM
        tabItem
    WHERE
        disabled = 0
""")

# Query Builder formatting
result = (
    frappe.qb.from_(doctype)
    .select(doctype.name)
    .where(doctype.autoname.like(Concat(prefix, ".%")) & doctype.name != name)
).run()
```

### 6.4 Simple Code Structures

- Don't club multiple conditions in one line
- Break complex AND/OR conditions into multiple lines for readability
- Use clear variable names that explain intent

### 6.5 Function Sequence

- Calling function should be on top, called functions below
- This follows the natural reading flow: controller → helpers
- Matches our controller-first structure

```python
def validate(doc, method=None):
    check_stock_availability(doc)  # Called first
    validate_quantities(doc)       # Called second


def check_stock_availability(doc):
    # Implementation
    pass


def validate_quantities(doc):
    # Implementation
    pass
```

### 6.6 Business Logic Must Be Server-Side (Python)

- All business logic must be implemented in Python (not just JavaScript)
- Calculations and value-setting logic must work via REST API
- When documents are posted via API, only server-side code executes
- JavaScript should only handle UI/UX interactions

### 6.7 Code Comments

- Code should be self-explanatory where possible
- Comments should explain "why" the code exists, not just "what" it does
- Use docstrings for function/method documentation

Example:

```python
def _get_available_qty(item_code: str, warehouse: str) -> float:
    """Get available stock quantity from bin for given item and warehouse."""
    if not (item_code and warehouse):
        return 0.0

    bin_doc = get_bin(item_code, warehouse)
    return flt(getattr(bin_doc, "actual_qty", 0) or 0.0)
```

### 6.8 Avoid Deprecated APIs

Do not use deprecated Frappe/ERPNext APIs. Avoid globals like:
- `cur_frm` (use `frm` parameter instead)
- `$c_obj()`
- `get_query` (use `set_query`)
- `add_fetch` (use custom scripts or controller methods)

### 6.9 Code Formatting

- Use tabs, not spaces (ERPNext legacy standard)
- Maintain consistent indentation throughout the file
- Follow PEP 8 guidelines where applicable

### 6.10 General Best Practices

- Keep controller methods thin - delegate work to helper functions
- Avoid DB writes outside intended lifecycle methods
- Ensure predictable execution flow
- Write API-friendly code that works both via UI and REST API

## 7. DocEvent (Override) Standards

When writing DocEvent overrides (hooking into standard Frappe/ERPNext doctypes), follow the same structure as Doctype classes.

### 7.1 DocEvent Method Naming

- DocEvent method names must match the controller name exactly
- Use standard controller names: `validate`, `on_submit`, `before_save`, `on_cancel`, etc.
- Method signature: `def controller_name(doc, method=None):`

### 7.2 DocEvent File Structure

DocEvent files must follow this exact structure:

```python
# --- Imports ---
<import statements>

# --- Controllers ---
<controller methods in execution order>

# --- <Controller Name> Helpers ---
<private helper methods>

# --- Whitelisted Methods ---
<@frappe.whitelist() methods>
```

### 7.3 Example — Correctly Structured DocEvent

```python
import frappe
from frappe import _
from frappe.utils import flt, get_url_to_form
from erpnext.stock.utils import get_bin
from utthunga_customizations.utthunga_customizations.helpers.epc_budget.budget_validation import validate_budget_not_exceeded
from utthunga_customizations.utthunga_customizations.helpers.utils import get_project_emails as _get_project_emails


# --- Controllers ---
def validate(doc, method=None):
    validate_budget_not_exceeded(doc)


def on_submit(doc, method=None):
    notify_insufficient_stock_on_mr_submit(doc)


# --- On Submit Helpers ---
def notify_insufficient_stock_on_mr_submit(doc, method=None) -> None:
    """Check stock and notify Purchase Manager if stock is insufficient."""
    if getattr(doc, "material_request_type", None) != "Purchase":
        return
    if not getattr(doc, "items", None):
        return

    short_items: list[dict] = []

    for row in doc.items:
        item_code = row.get("item_code")
        required_qty = flt(row.get("qty"))

        if not item_code or required_qty <= 0:
            continue

        warehouse = row.get("warehouse") or doc.get("set_warehouse")
        if not warehouse:
            continue

        available_qty = _get_available_qty(item_code, warehouse)

        if required_qty > available_qty:
            short_items.append({
                "item_code": item_code,
                "required_qty": required_qty,
                "available_qty": available_qty,
                "warehouse": warehouse,
            })

    if not short_items:
        return

    recipient = _get_purchase_manager_recipient_from_project(doc)
    if not recipient:
        return

    _send_insufficient_stock_email(doc, short_items, recipient)


def _get_available_qty(item_code: str, warehouse: str) -> float:
    if not (item_code and warehouse):
        return 0.0

    bin_doc = get_bin(item_code, warehouse)
    return flt(getattr(bin_doc, "actual_qty", 0) or 0.0)


def _get_purchase_manager_recipient_from_project(doc) -> str:
    project = getattr(doc, "project", None)
    if not project:
        return ""

    email = _get_project_emails(project=project, fields=("custom_purchase_manager",))
    return email[0] if email else ""


def _send_insufficient_stock_email(doc, short_items: list[dict], recipient: str) -> None:
    if not recipient or not short_items:
        return

    frappe.sendmail(
        recipients=recipient,
        subject=_("Material Request {0}: Insufficient Stock").format(doc.name),
        template="material_request_insufficient_stock",
        args={"doc": doc, "items": short_items, "link": get_url_to_form(doc.doctype, doc.name)},
        reference_doctype=doc.doctype,
        reference_name=doc.name,
    )


# --- Whitelisted Methods ---
@frappe.whitelist()
def update_item_quantities(material_request_id: str, updated_items):
    """Update item quantities for short closing Material Request."""
    # implementation here
    pass
```

## 8. Enforcement

- Every new Doctype and DocEvent must follow this structure
- PRs not following this standard should not be approved
- This standard is mandatory for all Python Frappe Doctype and DocEvent implementations.

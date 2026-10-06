---
name: javascript-frappe-standards
description: JavaScript coding standards for Frappe Doctype client scripts: event order, frm.events helpers, translations, set_query in setup, no business logic in JS, no deprecated APIs. Use when writing, editing or reviewing any .js file in this app.
---

# JavaScript Frappe Doctype Coding Standards

This document defines standard conventions for writing JavaScript Doctype client scripts in Frappe. The goal is to ensure consistency, readability, maintainability, and predictable execution flow across all client-side code.

## 1. General Principles

- Every Doctype JavaScript file must follow Frappe's standard event lifecycle
- Standard events must always be written first in the form handler
- Events must be:
  - Grouped together by type
  - Ordered according to execution flow
  - Clearly separated using section comments
- All helper functions must:
  - Be defined in the parent form's `frm.events` object
  - Be reusable across child tables
  - Follow clear naming conventions

## 2. Mandatory Section Order Inside a Doctype JavaScript File

Every Doctype JavaScript file must follow this exact structure:

```javascript
// --- Constants (if needed) ---
const CONSTANT_NAME = [...];

frappe.ui.form.on("DoctypeName", {
    // --- Standard Events ---
    setup(frm) { },
    refresh(frm) { },
    onload(frm) { },
    onload_post_render(frm) { },

    // --- Field Change Events ---
    fieldname(frm) { },

    // --- Helper Functions ---
    helper_function(frm, ...args) { },
});

// --- Child Table Events ---
frappe.ui.form.on("ChildDoctypeName", {
    fieldname(frm, cdt, cdn) { },
});
```

## 3. Event Categories and Execution Order

### 3.1 Standard Events (Lifecycle Events)

Standard events execute in this order:

1. **`setup`** - Runs once when form is first loaded, used for:
   - Setting query filters (`set_query`)
   - Initializing form properties
   - One-time setup operations
2. **`onload`** - Runs when form loads (before rendering)
3. **`onload_post_render`** - Runs after form is rendered
4. **`refresh`** - Runs every time form refreshes, used for:
   - Updating buttons
   - Setting field properties dynamically
   - Applying conditional formatting
   - Triggering initial calculations

**Important Rules:**
- Never write business logic (calculations, validations) in standard events
- Standard events should only handle UI/UX setup
- Avoid `before_save` and `validate` events in JavaScript - use Python controllers instead

### 3.2 Field Change Events

Field events trigger when a specific field's value changes:

```javascript
fieldname(frm) {
    // Handle field change
}
```

**Best Practices:**
- Keep field change handlers focused on single responsibility
- Delegate complex logic to helper functions
- Use async/await for database calls
- Always check if required values exist before processing

### 3.3 Child Table Events

Child table events receive three parameters:

```javascript
frappe.ui.form.on("ChildDoctypeName", {
    fieldname(frm, cdt, cdn) {
        const row = locals[cdt][cdn];
        // Handle child table field change
    }
});
```

## 4. Helper Functions

### 4.1 Placement and Structure

Helper functions must be:
- Defined in the parent form's `frm.events` object
- Placed after all event handlers
- Named descriptively using snake_case or camelCase
- Reusable across parent and child table events

Example:

```javascript
frappe.ui.form.on("Delivery Handover", {
    // --- Standard Events ---
    refresh(frm) {
        frm.events.update_work_type(frm, false);
        frm.events.apply_checklist_row_colors(frm);
    },

    // --- Field Change Events ---
    po_available(frm) {
        frm.events.update_work_type(frm, true);
    },

    // --- Helper Functions ---
    update_work_type(frm, set_value) {
        const is_po_yes = frm.doc.po_available === "Yes";
        const options = is_po_yes ? ["Booked Work"] : ["Work at Risk", "Work at Opportunity"];
        frm.set_df_property("work_type", "options", options);

        if (set_value) {
            frm.set_value("work_type", is_po_yes ? "Booked Work" : "");
        }
    },

    apply_checklist_row_colors(frm) {
        // Implementation
    }
});
```

### 4.2 Calling Helper Functions

**From Parent Form:**

```javascript
frm.events.helper_function(frm, arg1, arg2);
```

**From Child Table:**

```javascript
frappe.ui.form.on("Child Table", {
    fieldname(frm, cdt, cdn) {
        frm.events.parent_helper_function(frm, cdt, cdn);
    }
});
```

## 5. Best Practices

### 5.1 Use Translations

All user-facing strings must be wrapped in the `__("")` function:

```javascript
// ✅ Correct
frappe.msgprint(__("Cannot delete Milestone No: {0}", [milestone_no]));
frappe.throw(__("Project ID is required"));

// ❌ Wrong
frappe.msgprint("Cannot delete Milestone No: " + milestone_no);
```

### 5.2 Business Logic Must Be Server-Side

- Never implement business logic in JavaScript
- Calculations, validations, and data transformations must be in Python
- JavaScript should only handle:
  - UI interactions
  - Field visibility/properties
  - Client-side user experience
  - Calling server methods

Example:

```javascript
// ❌ Wrong - Business logic in JS
po_available(frm) {
    if (frm.doc.po_available === "Yes") {
        frm.doc.custom_total = frm.doc.amount * 1.18; // Tax calculation
    }
}

// ✅ Correct - Call server method
po_available(frm) {
    frm.call({
        method: "calculate_totals",
        doc: frm.doc,
        callback: (r) => {
            frm.refresh_field("custom_total");
        }
    });
}
```

### 5.3 Query Filters

Set query filters in the `setup` event:

```javascript
setup(frm) {
    frm.set_query("customer", () => ({
        filters: { disabled: 0 }
    }));

    frm.set_query("item_code", "items", () => ({
        filters: {
            is_sales_item: 1,
            company: frm.doc.company
        }
    }));
}
```

### 5.4 Async/Await for Database Calls

Use async/await for cleaner asynchronous code:

```javascript
// ✅ Good
async customer_name(frm) {
    if (!frm.doc.customer_name) return;

    const res = await frappe.db.get_value("Customer", frm.doc.customer_name, "payment_terms");
    const template = res.message?.payment_terms || null;
    frm.set_value("payment_terms_template", template);
}

// ❌ Avoid nested callbacks
customer_name(frm) {
    frappe.db.get_value("Customer", frm.doc.customer_name, "payment_terms", (r) => {
        if (r.message) {
            frm.set_value("payment_terms_template", r.message.payment_terms);
        }
    });
}
```

### 5.5 Field Property Updates

Use `set_df_property` to update field properties:

```javascript
// Show/hide fields
frm.set_df_property("fieldname", "hidden", 1);

// Read-only fields
frm.set_df_property("fieldname", "read_only", 1);

// Required fields
frm.set_df_property("fieldname", "reqd", 1);

// Field options
frm.set_df_property("fieldname", "options", ["Option 1", "Option 2"]);
```

### 5.6 Working with Child Tables

**Adding rows:**

```javascript
frm.add_child("items", {
    item_code: "ITEM-001",
    qty: 1,
    rate: 100
});
frm.refresh_field("items");
```

**Iterating rows:**

```javascript
(frm.doc.items || []).forEach(row => {
    // Process each row
});
```

**Accessing row data:**

```javascript
frappe.ui.form.on("Child Table", {
    fieldname(frm, cdt, cdn) {
        const row = locals[cdt][cdn];
        console.log(row.item_code);
    }
});
```

### 5.7 Conditional Logic

Keep conditions simple and readable:

```javascript
// ✅ Good - Clear and readable
if (!frm.doc.customer_name) {
    frm.set_value("payment_terms_template", null);
    return;
}

// ❌ Avoid complex nested conditions
if (frm.doc.customer_name && frm.doc.company && !frm.doc.is_internal) {
    if (frm.doc.payment_terms_template) {
        // nested logic
    }
}
```

### 5.8 Error Handling

Use appropriate error handling methods:

```javascript
// Validation errors
frappe.throw(__("Field is required"));

// User messages
frappe.msgprint(__("Operation completed successfully"));

// Confirmation dialogs
frappe.confirm(
    __("Are you sure you want to proceed?"),
    () => {
        // On yes
    },
    () => {
        // On no (optional)
    }
);

// Alerts
frappe.show_alert({
    message: __("Record saved successfully"),
    indicator: "green"
});
```

### 5.9 Code Comments

- Use section comments to organize code: `// --- Section Name ---`
- Comment complex logic to explain "why", not "what"
- Use JSDoc for function documentation when needed

```javascript
// Calculate duration between two dates using server-side API
async calculate_duration(start_date, end_date) {
    const response = await frappe.call({
        method: "module.get_duration_in_days",
        args: { start_date, end_date }
    });
    return response.message || 0;
}
```

### 5.10 Avoid Deprecated APIs

Do not use deprecated Frappe JavaScript APIs:

- ❌ `cur_frm` - Use `frm` parameter instead
- ❌ `$c_obj()` - Use `frappe.call()` or `frm.call()`
- ❌ `get_query` - Use `set_query`
- ❌ `add_fetch` - Use custom scripts or field updates

## 6. Complete Example - Correctly Structured JavaScript File

```javascript
// --- Constants ---
const CHECKLIST_TABLES = [
    "check_list_contracts",
    "check_list_commercial_conditions",
    "check_list_scope_of_supplies"
];

frappe.ui.form.on("Delivery Handover", {
    // --- Standard Events ---
    setup(frm) {
        frm.set_query("customer", () => ({
            filters: { disabled: 0 }
        }));

        frm.set_query("item_code", "items", () => ({
            filters: {
                is_sales_item: 1,
                company: frm.doc.company
            }
        }));
    },

    refresh(frm) {
        frm.events.update_button_visibility(frm);
        frm.events.apply_field_permissions(frm);
        frm.events.load_checklist_rows(frm);
    },

    onload_post_render(frm) {
        frm.events.clear_company_if_new(frm);
    },

    // --- Field Change Events ---
    async customer(frm) {
        if (!frm.doc.customer) {
            frm.set_value("payment_terms_template", null);
            return;
        }

        const res = await frappe.db.get_value(
            "Customer",
            frm.doc.customer,
            "payment_terms"
        );

        frm.set_value("payment_terms_template", res.message?.payment_terms || null);
    },

    business_unit(frm) {
        frm.set_value("department", "");
    },

    project_category(frm) {
        frappe.dom.freeze(__("Updating Checklist Tables..."));
        frm.call({
            method: "update_check_list_tables",
            doc: frm.doc,
            callback: () => frappe.dom.unfreeze()
        });
    },

    // --- Helper Functions ---
    update_button_visibility(frm) {
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button(__("Create Project"), () => {
                frm.events.create_project(frm);
            });
        }
    },

    apply_field_permissions(frm) {
        const logged_in_user = frappe.session.user;
        const is_owner = frm.doc.owner === logged_in_user;

        frm.set_df_property("delivery_manager", "read_only", !is_owner ? 1 : 0);
        frm.set_df_property(
            "project_manager",
            "read_only",
            frm.doc.delivery_manager !== logged_in_user ? 1 : 0
        );
    },

    load_checklist_rows(frm) {
        const fixed_rows = ["Proposal", "Drawings", "BOQ"];

        frm.set_df_property("attachments", "cannot_add_rows", true);
        frm.set_df_property("attachments", "cannot_delete_rows", true);

        if (frm.doc.attachments?.length > 0) return;

        fixed_rows.forEach(name => {
            frm.add_child("attachments", { attachment_name: name });
        });

        frm.refresh_field("attachments");
    },

    clear_company_if_new(frm) {
        if (frm.is_new()) {
            frm.set_value("company", "");
        }
    },

    async create_project(frm) {
        frappe.confirm(
            __("Create a new project from this handover?"),
            async () => {
                try {
                    const result = await frm.call({
                        method: "create_project_document",
                        doc: frm.doc
                    });

                    if (result.message) {
                        frappe.show_alert({
                            message: __("Project {0} created successfully", [result.message]),
                            indicator: "green"
                        });
                        frm.reload_doc();
                    }
                } catch (error) {
                    frappe.msgprint({
                        title: __("Error"),
                        message: error.message,
                        indicator: "red"
                    });
                }
            }
        );
    }
});

// --- Child Table Events ---
frappe.ui.form.on("Milestone Item", {
    async start_date(frm, cdt, cdn) {
        const row = locals[cdt][cdn];
        if (!row.start_date || !row.end_date) return;

        const duration = await frm.events.calculate_duration(row.start_date, row.end_date);
        frappe.model.set_value(cdt, cdn, "duration_days", duration);
    },

    async end_date(frm, cdt, cdn) {
        const row = locals[cdt][cdn];
        if (!row.start_date || !row.end_date) return;

        const duration = await frm.events.calculate_duration(row.start_date, row.end_date);
        frappe.model.set_value(cdt, cdn, "duration_days", duration);
    }
});
```

## 7. Enforcement

- Every JavaScript file must follow this structure
- Business logic must not be implemented in JavaScript
- All user-facing strings must be translated
- PRs not following this standard should not be approved
- This standard is mandatory for all Frappe Doctype JavaScript implementations.

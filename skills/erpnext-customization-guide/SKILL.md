---
name: erpnext-customization-guide
description: Guidelines for customizing ERPNext/Frappe: configurable notifications, config-driven overrides that fall back to standard behaviour, hooks.py hygiene, no hardcoded client values. Use when touching hooks.py, doc_events, overrides, fixtures or custom fields.
---

# ERPNext/Frappe Customization — General Implementation Guidelines

These guidelines apply to all client customization work on ERPNext/Frappe. The goal is to keep customizations **configurable, reversible, and non-invasive** to standard functionality.

---

## 1. Notifications — Always Configurable (Enable/Disable)

Any custom notification logic (email, SMS, in-app alerts, webhook triggers, etc.) must have a way to be turned on or off without touching code.

**Recommended approach:**
- Prefer the standard **Notification** doctype (Frappe's built-in notification framework) wherever the use case fits — it already comes with an `enabled` checkbox, condition field, and channel selection out of the box.
- If a custom notification requires logic beyond what the Notification doctype supports (e.g. complex business rules, third-party integration), build it in a custom hook/method but **gate it behind a settings field** — e.g. a checkbox in:
  - A custom "Settings" doctype (e.g. `Client XYZ Settings`), or
  - An existing settings doctype if one already exists for the module.
- Never hardcode notification triggers as "always on." A functional/business user should be able to disable a notification from the UI without needing a developer.

**Checklist:**
- [ ] Is there an enable/disable checkbox for this notification?
- [ ] Does disabling it fully stop the trigger (not just suppress the message)?
- [ ] Is the setting discoverable by a non-technical user (proper label, in a logical doctype)?

---

## 2. Config-Driven Overrides — Never Break Standard Functionality

Whenever standard ERPNext/Frappe behavior is being overridden or customized (server scripts, hooks, overridden whitelisted methods, doc_events, client scripts, etc.), it must be done in a way that:

- **Standard functionality can still work as-is** if the customization is disabled, removed, or not applicable for a given company/client/scenario.
- The override is **config-driven**, not a permanent hard replacement of standard behavior.

**Recommended approach:**
- Add a settings flag (custom field or dedicated settings doctype) that controls whether the override logic runs.
  ```python
  def custom_validate(doc, method):
      settings = frappe.get_cached_doc("Client XYZ Settings")
      if not settings.enable_custom_validation:
          return  # fall back to standard behavior
      # custom logic here
  ```
- Avoid **monkey-patching** or fully replacing standard methods unless absolutely necessary. Prefer:
  - `doc_events` hooks (before_save, on_submit, etc.) over overriding controller methods directly.
  - Extending via inheritance (custom Controller extending the standard one) over copy-pasting and modifying standard code.
- If a standard method *must* be overridden (`override_whitelisted_methods` / `override_doctype_class`), keep the override as a thin wrapper that calls the standard method first (or falls back to it) whenever the custom condition isn't met.
- Document every override in one place (e.g. a `CUSTOMIZATIONS.md` or the knowledge base) listing:
  - What is overridden
  - Why
  - The config/flag that controls it
  - What happens when the flag is off

**Checklist:**
- [ ] Is there a way to turn this override off and get pure standard behavior back?
- [ ] Is the override scoped (per company/client) rather than global, where relevant?
- [ ] Is the override documented with its controlling config?
- [ ] Will a future ERPNext version upgrade still work if this override is disabled?

---

## 3. General Principles (Supporting the Above)

- **Config over code:** Any behavior a client might want to toggle, tune, or turn off later should live in a settings doctype/custom field — not buried in code that requires a developer to change.
- **Fail safe to standard:** If a customization errors out or its config is missing, it should fall back to standard behavior rather than breaking the transaction.
- **No hardcoding client-specific values** (company names, warehouse codes, email IDs) directly in code — use settings/config doctypes or Frappe's Custom Field / Property Setter mechanisms.
- **Use hooks.py responsibly:** Keep custom `doc_events`, `scheduler_events`, and `override_whitelisted_methods` entries scoped and documented — don't let hooks.py become a dumping ground with no traceability.
- **Version/upgrade safety:** Before overriding a standard method, check if it changes across ERPNext versions. Prefer hook-based extension so upgrades don't silently break the override.
- **Test both states:** Every customization should be tested with the config **enabled** and **disabled** to confirm standard functionality is genuinely intact when off.

---

*This document is intended as a living checklist — update it as new patterns or pitfalls are identified during implementation work.*

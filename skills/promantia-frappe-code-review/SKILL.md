---
name: promantia-frappe-code-review
description: Review Frappe/ERPNext code in this app against the team's Python, JavaScript and customization standards. Use when the user asks to review a diff, a PR, a doctype, a controller, a client script, a hook or a DocEvent override — for example "review this", "check this against our standards", "is this ready to merge".
---

# Frappe Code Review

Review code against the three standards skills in the sibling folders. Report findings. Do not edit code unless the user asks.

## 1. Load the references

Read the skills that apply to the files under review:

| File type | Skill |
|---|---|
| `*.py` (doctype controllers, DocEvents, APIs, helpers) | `python-frappe-standards` (`../python-frappe-standards/SKILL.md`) |
| `*.js` (doctype client scripts, `doctype_js`) | `javascript-frappe-standards` (`../javascript-frappe-standards/SKILL.md`) |
| Anything that changes core ERPNext/Frappe behavior (`hooks.py`, `doc_events`, `override_*`, custom fields, notifications) | `erpnext-customization-guide` (`../erpnext-customization-guide/SKILL.md`) |

Also read the app's `CLAUDE.md` and `docs/knowledge.md`. Project rules there override the generic standards. For example: Shipment 3PL, GRN 3PL and Delivery Order 3PL controllers are empty on purpose, because their validation lives in `wizard_engine/`.

## 2. Find the scope

- If the user names files or a PR, review those.
- If not, review the uncommitted diff (`git diff` and `git status`). Also read the full file around each change, because a small diff can break code near it.
- For each doctype you review, open its `.json` schema. Check field types, `fetch_from`, `reqd`, `read_only`, Select options and permissions. Many bugs come from code that disagrees with the schema.

## 3. Check, in this order

1. **Correctness.** Does the code do what it claims? Look for:
   - Conditions that compare against a value the schema does not allow.
   - Client code that overwrites a `fetch_from` or configured value.
   - `set_value` in `refresh`, which makes saved docs dirty.
   - Missing `None` checks.
   - `frappe.db.commit()` in the middle of a transaction.
2. **Security.** Look for:
   - Raw SQL built with string formatting.
   - Whitelisted methods that don't check input types or permissions.
   - `allow_guest`.
   - User input put into the DOM without escaping.
3. **Performance.** Look for:
   - DB calls inside loops.
   - Server calls on every `refresh`.
   - Fetching a whole doc to read one value.
   - Filters on columns with no index.
4. **Standards compliance.** Check against the reference docs:
   - Section order and section comments.
   - Business logic kept server-side.
   - `__()` / `_()` translations.
   - `set_query` in `setup`.
   - No deprecated APIs (`cur_frm`, `get_query`, `add_fetch`).
   - Tabs in Python.
   - Small helpers.
   - Customizations that can be turned off through config.
5. **Maintainability.** Look for:
   - Commented-out code.
   - Dead branches.
   - Duplicate logic.
   - Long history comments that belong in `docs/`.
6. **Tests.** Does the change have a test, or a clear manual check if it is JS-only?

## 4. Report

- Group findings by severity: **Critical**, **Major**, **Minor**.
- For each finding, give:
  - The file and line number, as a clickable location, for example `[file.js:42](path/file.js#L42)`.
  - The rule it violates, with the section from the reference, for example "JS standards §5.3".
  - Why it matters, with a concrete scenario of what breaks and for whom.
  - The corrected code.
- Say which files you checked and found clean.
- End with a summary table of issue counts per file, and a short summary of the fixes you recommend.

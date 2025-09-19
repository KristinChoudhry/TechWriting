# Syntax Reference Guide

This guide explains how to create a new product with specified configuration parameters. Use it when adding catalog items and setting rating methods and attributes.

It expands the original template with clearer field definitions, allowed values, defaults, dependencies, and validation notes. Keep the same sections below when copying into your docs.

---

## Syntax

- Navigation: Products → Product Management → Products → New
- Prerequisites:
  - User role with permission to create and edit products
  - If you plan to set a default rating method, ensure at least one rating method exists
  - If you plan to enforce contract rates, ensure contract sources are configured

---

## Required fields

- Product Name

Details:

| Field        | Type | Required | Description |
|--------------|------|----------|-------------|
| Product Name | Text | Yes      | Unique identifier and display name for the product |

Additional notes:
- Length: 1–200 characters
- Allowed characters: letters, numbers, spaces, `_ - .`
- Validation:
  - Must be unique across all products (case-insensitive)
  - Leading and trailing spaces are trimmed
  - Disallows control characters and duplicate consecutive spaces
- Behavior:
  - Appears in product lists, search, and lookups
  - Used as the default display name in quotes unless overridden

---

## Optional configurations

- Override Permissions
- Rating Method
- Contract Settings
- Pricing Attributes

Details:

| Field                                  | Type     | Required | Description |
|----------------------------------------|----------|----------|-------------|
| Override Permission                    | Dropdown | No       | Controls ability to modify rating/price |
| Product Status                         | Dropdown | No       | Controls availability (set after initial save) |
| Contractual Rate Source Required       | Checkbox | No       | Forces contract-level rates |
| Restrict Use of Unreferenced Contracts | Checkbox | No       | Limits to contract rates on current account |
| Specify Default Rating Method          | Checkbox | No       | Enables rating method configuration |
| Rating Method                          | Dropdown | Conditional | Default algorithm for calculating charges |
| Pricing Attributes                     | Group    | No       | Custom attributes used in rating and pricing |
| Contract Settings                      | Group    | No       | Additional constraints when contracts are enforced |

Behavior and dependencies:
- Product Status is only editable after the first save. Default is Draft.
- Specify Default Rating Method must be On to expose the Rating Method field.
- Restrict Use of Unreferenced Contracts is only available when Contractual Rate Source Required is On.
- Rating Method may require additional fields in Pricing Attributes (for example, `unit_rate` for Per Unit).

Field-level notes:
- Pricing Attributes
  - Type: Key-value pairs; data types include number, currency, boolean, text, date
  - Used as inputs for rating formulas or tier selection
  - Validation: attribute names must be unique for the product
- Contract Settings
  - Applies when Contractual Rate Source Required is On
  - Includes restrictions on allowable rate sources and contract scoping

---

## Override Permission options

| Option           | Description                                       | Default |
|------------------|---------------------------------------------------|---------|
| No Restriction   | Allow rating method and rate changes              | Yes     |
| Lock Rating      | Prevent changes to the rating method              | No      |
| Lock Rate Values | Prevent changes to rate tables and numeric values | No      |
| Fully Locked     | Prevent changes to both method and rate values    | No      |

Behavior:
- Lock states apply in the product editor and in downstream quoting flows
- Admin roles may retain override capability depending on system policy
- Changing from a more permissive to a more restrictive option can invalidate draft quotes

---

## Rating Method

Purpose:
- Defines how charges are computed for the product

Visibility and requirement:
- Visible only when Specify Default Rating Method is On
- Required before activating the product if Specify Default Rating Method is On

Common options and inputs:

| Method     | Inputs required                        | Example calculation |
|------------|-----------------------------------------|---------------------|
| Flat Rate  | `base_rate`                             | `charge = base_rate` |
| Per Unit   | `unit_rate`, `quantity`                 | `charge = unit_rate * quantity` |
| Tiered     | `tiers[]`, `quantity`                   | Sum across tiers consumed |
| Volume     | `breakpoints[]`, `quantity`             | `charge = unit_rate_at_volume(quantity) * quantity` |
| Time Based | `rate_per_hour`, `duration_hours`       | `charge = rate_per_hour * duration_hours` |
| Formula    | `formula`, variables via attributes     | `charge = eval(formula, attributes)` |

Notes:
- Selecting a method can reveal additional inputs in Pricing Attributes
- Changing an existing method may trigger recalculation and warnings in active quotes

---

## Product Status

- Set after initial save
- Affects visibility in search, quoting, and ordering

Typical values:
- Draft: Not visible to external users; default after first save
- Active: Available for selection in quotes and orders
- Inactive: Hidden for new selections; existing quotes may still reference it
- Deprecated: Visible for reference only; not selectable in new transactions

Transition rules:
- Draft → Active requires all required fields to be valid
- Active → Inactive may warn if the product is in use
- Deprecated transitions may require administrator role

---

## Contract settings

Fields:
- Contractual Rate Source Required (On/Off)
- Restrict Use of Unreferenced Contracts (On/Off)

Behavior:
- When Contractual Rate Source Required is On:
  - Rates must come from a contract-level source
  - Ad hoc rates and public rate tables are disallowed
  - Restrict Use of Unreferenced Contracts becomes available
- When Restrict Use of Unreferenced Contracts is On:
  - Only contracts linked to the current account can be used
  - Attempting to select rates from unrelated contracts results in a validation error

Validation and errors:
- If On and no valid contract rate source exists: show an error on save
- If restricted and the account has no linked contracts: block activation or selection

---

## Pricing attributes

Purpose:
- Provide inputs for rating and pricing methods

Examples:
- `unit_rate` (currency)
- `base_rate` (currency)
- `min_commitment` (number)
- `discount_eligible` (boolean)
- `rating_formula` (text or expression)

Guidelines:
- Use descriptive names; keep units clear (for example, rate per hour)
- Set default values where appropriate
- Document attribute purpose and range in product notes

Validation:
- Attribute names must be unique per product
- Numeric fields can support up to 4 decimal places unless configured otherwise
- Currency fields use the system default currency unless specified

---

## Validation rules

- Product Name must be unique and non-empty
- When Specify Default Rating Method is On, Rating Method is required
- When Contractual Rate Source Required is On:
  - At least one valid contract rate source must be resolvable
  - Restrict Use of Unreferenced Contracts requires a current account context
- When Product Status is set to Active:
  - All required and conditionally required fields must pass validation
  - Any locked override states must be consistent with the selected rating configuration

---

## Examples

Create a product with a Per Unit rating method:
1. Go to Products → Product Management → Products → New
2. Enter Product Name
3. Turn on Specify Default Rating Method
4. Set Rating Method to Per Unit
5. In Pricing Attributes, set `unit_rate = 25.00`
6. Save the product
7. Set Product Status to Active

Create a product that requires contract rates:
1. Go to Products → Product Management → Products → New
2. Enter Product Name
3. Turn on Contractual Rate Source Required
4. (Optional) Turn on Restrict Use of Unreferenced Contracts
5. Save the product
6. Set Product Status to Active when a valid contract source is available

Lock rating changes:
1. Open an existing product
2. Set Override Permission to Fully Locked
3. Save changes
4. Confirm that Rating Method and rate fields are disabled in quoting flows

---

## Error messages and troubleshooting

Common messages:
- Product Name already exists
  - Cause: Duplicate name
  - Fix: Choose a unique name or modify the existing product
- Rating Method required
  - Cause: Specify Default Rating Method is On but no method selected
  - Fix: Select a valid rating method or turn off the option
- Contract rate source required
  - Cause: Contractual Rate Source Required is On but no applicable source found
  - Fix: Link a valid contract to the account or configure a contract rate source
- Unreferenced contract not allowed
  - Cause: Restrict Use of Unreferenced Contracts is On and the selected contract is not linked
  - Fix: Use a contract associated with the current account

Diagnostic tips:
- Check field visibility rules if expected fields are missing
- Verify user permissions for editing status and locked fields
- Review product audit logs for recent changes that affect validation

---

## Notes and best practices

- Use Draft while configuring complex products and switch to Active only when validation passes
- Prefer clear, measurable Pricing Attributes and avoid ambiguous names
- Document assumptions in product notes (units, rounding, dependencies)
- Test changes in a staging environment before applying to Active products
- When tightening Override Permission, communicate impacts to quoting teams

---



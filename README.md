# Costlab: Dispensing Cost (per order) prototype

A clickable prototype of a new **Dispensing cost (per order)** section for the Master Formulation Record (formula detail page, `src/app/(app)/formulas/[id]/page.tsx`).

Open `index.html` in a browser. It is a single file with sample data and no backend; nothing saves.

## What it does

The section sits below the existing **Cost breakdown**. The batch cost breakdown and all-in cost do **not** change. Dispensing cost is its own per-order figure, meant to be carried to the quote side.

### Dispensing cost templates
- Dropdown labeled **Dispensing cost templates**. Default is **None** (all fields blank).
- Picking a template fills in labor, packaging and overhead.
- Editing any value after that sets the dropdown back to **None** (the formula now has its own values).
- Example templates in the prototype: Standard CC Dispense, Cold Chain Overnight, Ambient Mailer.

### Labor (flat $ per order)
- Filling
- PV (pharmacist verification)
- Shipping

### Packaging & material
- Library components with a quantity per order (for example Xpander Pack x1, Ice Pack x4).
- Each line is priced at the component's preferred manufacturer: `qty per order @ $unit each`.
- Add with a searchable component picker (name or NDC/SKU); quantity is editable inline; lines can be removed.

### Overhead
- One flat $ amount per order.

### Buttons
- **Save as template** (white, secondary): asks for a template name, rejects blank or duplicate names, then adds the template to the dropdown and selects it.
- **Save dispensing cost** (primary): saves the values on this formula.

### Total dispensing cost panel
- Headline total = **Product + Labor + Packaging + Overhead**.
- **Product** is the all-in cost per saleable unit from the cost breakdown ($6.50 in the sample).
- Stacked bar and line items for Product, Labor, Packaging, Overhead.

## Reference implementation

`reference/formula-dispensing-cost.patch` is a working implementation against `grantajensen/costlab` (based on `main` at `1f1264a`). It includes:

- Migration `0025_formula_dispensing_cost.sql`: four `dispense_*_cents` columns on `formulas` and a `formula_dispensing_items` table (RLS matches other formula tables).
- `src/lib/dispensing-cost.ts` calculation plus tests (`tests/dispensing-cost.test.ts`).
- Server actions: save dispensing cost, add/remove packaging lines. Duplicating a formula copies packaging lines; deleting a component used as packaging is blocked.
- UI section on the formula page.

Typecheck, lint and the full test suite (197 tests) passed on that patch.

**The patch was written before the template and Product changes.** Still to build on top of it:

1. Templates: a `dispensing_cost_templates` table (plus template packaging lines), the dropdown with **None** as default, and **Save as template** with a name prompt.
2. Product line in the total panel, and the headline total including Product.

Apply with:

```
git am reference/formula-dispensing-cost.patch
```

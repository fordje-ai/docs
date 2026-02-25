# Docs Feedback Updates — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Update Fordje Connect developer docs to broaden the value prop, include utilities as AHJ type, add value framing to quickstart, enrich AHJ access with data point discovery, and remove hierarchy endpoint references.

**Architecture:** Pure documentation changes across 4 MDX files. No code, no tests. Each task targets one file. Verify with `mint dev` at the end.

**Tech Stack:** Mintlify (MDX), docs.json config

---

### Task 1: Update `index.mdx` — Broaden value prop, add utilities, add laddering explanation

**Files:**
- Modify: `docs/index.mdx`

**Step 1: Update frontmatter description**

Replace the `description` field in the YAML frontmatter:

```yaml
description: "Programmatic access to building code, zoning, permitting, licensing, utility interconnection requirements, and other regulatory data across the 30,000+ US jurisdictions."
```

**Step 2: Replace the intro paragraph (line 6)**

Replace:
```
Building codes vary across thousands of jurisdictions. Keeping track of which rules apply where is manual, error-prone, and time-consuming. Fordje Connect gives you programmatic access to building code data organized by jurisdiction — so you can look up, compare, and export code requirements across any Authority Having Jurisdiction (AHJ) your organization subscribes to.
```

With:
```
Building codes and other regulatory requirements are different across every one of the over 30,000 US jurisdictions. Keeping track of which rules apply where is manual, error-prone, and time-consuming. Fordje Connect gives you programmatic access to building code, zoning, permitting, licensing, utility interconnection requirements, and other regulatory data organized by jurisdiction — so you can look up, compare, and export requirements across any Authority Having Jurisdiction (AHJ) your organization subscribes to.
```

**Step 3: Update the AHJ accordion (line 12)**

Replace:
```
AHJs are the cities, counties, and states that write and enforce building codes. Every piece of data in Fordje is tied to a specific AHJ. Your organization subscribes to the AHJs relevant to your work, and Fordje Connect scopes all data to that subscription.
```

With:
```
AHJs are the cities, counties, states, and utilities that write and enforce building codes, zoning rules, permitting requirements, and interconnection standards. Every piece of data in Fordje is tied to a specific AHJ. Your organization subscribes to the AHJs relevant to your work, and Fordje Connect scopes all data to that subscription.
```

**Step 4: Add a "How Fordje handles jurisdictional laddering" section**

Add a new section after the `</AccordionGroup>` tag and before the `## How access works` section:

```mdx
## How Fordje handles jurisdictional laddering

Building codes often layer across multiple levels of government. A project in Los Angeles, for example, may need to comply with requirements from the City of Los Angeles, Los Angeles County, and the State of California — each of which can adopt its own codes. Utility interconnection requirements add another layer on top of that.

Fordje resolves this laddering for you behind the scenes. When you pull data points for an AHJ, the values you receive already account for the full chain of parent jurisdictions. You get the complete, resolved answer — no need to manually trace which level of government a requirement comes from.
```

**Step 5: Commit**

```bash
git add docs/index.mdx
git commit -m "docs: broaden value prop, add utilities to AHJ definition, add laddering explanation"
```

---

### Task 2: Update `quickstart.mdx` — Add value framing above code examples

**Files:**
- Modify: `docs/quickstart.mdx`

**Step 1: Add value framing to Step 2 (Look up code requirements)**

After line 78 (`Now let's query some actual code data. Data points are the individual code requirements tracked in Fordje — things like permit requirements, insulation values, or inspection rules.`) and before `You can filter data points by AHJ IDs, state, or trade group.`, add:

```
Plug these data points into your sales, design, engineering, or permitting tool so you have accurate data points exactly where you need them. Or pull a full set into your CRM at the start of a project.
```

**Step 2: Add value framing to Step 3 (Compare across jurisdictions)**

After line 117 (`One of the most powerful features of Fordje Connect is comparing code requirements across multiple jurisdictions at once. Reports let you build a matrix — jurisdictions on one side, code requirements on the other.`), add:

```
Quickly see where you can easily expand into or provide a full list of requirements for all current projects to your teams.
```

**Step 3: Commit**

```bash
git add docs/quickstart.mdx
git commit -m "docs: add value framing to quickstart steps"
```

---

### Task 3: Update `concepts/ahj-access.mdx` — Data point discovery, support callout, remove hierarchy

**Files:**
- Modify: `docs/concepts/ahj-access.mdx`

**Step 1: Add data point discovery section**

Add a new section after the `## Full denial` section and before the `## Which endpoints enforce access` table:

```mdx
## Discovering available data points

You can see which data points exist for any AHJ in your subscription by calling the data points endpoint filtered to specific jurisdictions:

```bash
curl -X GET "https://connect.fordje.com/v1/data-points?ahj_ids=1&page_size=10" \
  -H "X-API-Key: your-api-key"
```

This returns the data points that Fordje tracks for that jurisdiction. Examples of data points you might find include:

- Minimum insulation R-value
- Whether a permit is required for re-roofing
- Utility interconnection voltage thresholds
- Fire sprinkler requirements for residential construction
- Solar panel setback distances

Different AHJs will have different sets of data points depending on what Fordje has catalogued for that jurisdiction. See the [data points API reference](/api-reference/introduction) for all available filters and response details.

<Note>
  Not seeing a data point you need? Reach out to [support@fordje.com](mailto:support@fordje.com) — we're always expanding our coverage.
</Note>
```

**Step 2: Remove hierarchy row from the access control table**

Remove this row from the table:
```
| `GET /v1/ahjs/{id}/hierarchy` | 403 if not subscribed |
```

**Step 3: Commit**

```bash
git add docs/concepts/ahj-access.mdx
git commit -m "docs: add data point discovery section, support callout, remove hierarchy from table"
```

---

### Task 4: Update `api-reference/ahjs.mdx` — Add utilities, remove hierarchy section

**Files:**
- Modify: `docs/api-reference/ahjs.mdx`

**Step 1: Update intro text**

Replace:
```
AHJ (Authority Having Jurisdiction) endpoints let you list the AHJs in your subscription, get detail for a specific AHJ, and explore jurisdiction hierarchies.
```

With:
```
AHJ (Authority Having Jurisdiction) endpoints let you list the AHJs in your subscription and get detail for a specific AHJ. AHJs include cities, counties, states, and utilities.
```

**Step 2: Update the List AHJs example response to show a utility**

Add a utility example to the `items` array in the List AHJs example response, after the Houston entry:

```json
{
  "id": 3,
  "name": "Pacific Gas & Electric",
  "type": "utility",
  "state_code": "CA"
}
```

**Step 3: Remove the entire "Get AHJ hierarchy" section**

Delete everything from the `---` separator after the Get AHJ detail section (line 108) through the end of the file (lines 108-165). This removes:
- The `## Get AHJ hierarchy` heading
- The endpoint description
- Path parameters table
- Example request
- Example response

**Step 4: Commit**

```bash
git add docs/api-reference/ahjs.mdx
git commit -m "docs: add utility AHJ type, remove hierarchy endpoint"
```

---

### Task 5: Verify with local preview

**Step 1: Run broken links check**

```bash
cd docs && mint broken-links
```

Expected: No broken links (the hierarchy endpoint was internal-only, no other pages link to it as an anchor).

**Step 2: Run local preview**

```bash
cd docs && mint dev
```

Manually check:
- Index page shows broadened value prop and laddering section
- Quickstart steps 2 and 3 have value framing text
- AHJ Access page has data point discovery section and support callout, no hierarchy row
- AHJ API reference shows utility type and no hierarchy section

**Step 3: Final commit if any fixups needed**

```bash
git add -A && git commit -m "docs: fixups from review"
```

# @affine/templates — Code Documentation

Pre-built template content for AFFiNE: edgeless canvas templates, sticker assets, and the onboarding workspace ZIP.

---

## What's included

```
templates/
  onboarding/
    onboarding.zip          ← First-run workspace content
  edgeless/
    5W2H.json               ← 16 edgeless canvas templates
    SWOT.json
    Flowchart.json
    Gantt Chart.json
    [... 12 more]
  stickers/                 ← Sticker/emoji SVG assets
  edgeless-templates.gen.ts ← Generated loader (don't edit)
  stickers-templates.gen.ts ← Generated loader (don't edit)
```

---

## Using templates in code

### Edgeless canvas templates

```ts
import edgelessTemplates from '@affine/templates/edgeless';

// Templates grouped by category
const categories = edgelessTemplates;
// {
//   'Brainstorming': [template1, template2, ...],
//   'Marketing': [template3, ...],
//   'Presentation': [...],
//   'Project Management': [...],
// }

// Each template has the AFFiNE block JSON format
const template = categories['Brainstorming'][0];
// template.name, template.preview, template.content (JSON)
```

### Stickers

```ts
import stickers from '@affine/templates/stickers';
// Array of sticker groups, each with SVG assets
```

### Onboarding workspace

```ts
import onboardingZip from '@affine/templates/onboarding.zip';
// ArrayBuffer of the ZIP file — import into the workspace engine
```

---

## Template categories and names

| Category               | Templates                                                                                                                         |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **Brainstorming**      | 5W2H, Concept Map, Fishbone Diagram                                                                                               |
| **Marketing**          | SWOT Analysis, 4P Marketing Matrix                                                                                                |
| **Presentation**       | Simple Presentation, Storyboard, Business Proposal                                                                                |
| **Project Management** | Flowchart, Gantt Chart, Project Planning, Project Tracking Kanban, Monthly Calendar, User Journey Map, SMART Goals, Data Analysis |

---

## Regenerating templates

When a template is updated in AFFiNE:

1. Export the page from AFFiNE using `ZipTransformer`.
2. Unzip and place the JSON in `edgeless/` (for canvas templates) or `onboarding/` (for the onboarding zip).
3. Run `yarn build` or `yarn postinstall` — this triggers `build-edgeless.mjs` and `build-stickers.mjs`.
4. The `.gen.ts` files are regenerated automatically.

The build scripts read the JSON files, compress with JSZip, and produce the typed loader exports.

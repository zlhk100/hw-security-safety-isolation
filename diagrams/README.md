# Diagrams

Source SVG files for all diagrams used in the article and mechanism pages.

## Naming Convention

```
[scope]-[subject]-[type].svg

scope:   article / mechanism / platform / use-case
subject: smmu / mpam / qualcomm / sdv / etc.
type:    topology / flow / comparison / taxonomy
```

Examples:
- `article-taxonomy-overview.svg` — the unified taxonomy diagram in the main article
- `mechanism-smmu-nested-walk.svg` — SMMU nested translation walk
- `use-case-sdv-isolation-topology.svg` — SDV spatial isolation topology

## Contribution Guidelines for Diagrams

1. **Source format:** SVG only — no PNG, no JPEG, no binary formats. SVGs are version-controllable and can be reviewed in PRs.

2. **Color palette:** Use CSS variables matching the article theme:
   - Initiator-side elements: `#1a6fb5` (blue)
   - Target-side elements: `#c0440e` (warm red)
   - Safety/FFI elements: `#1a7a4a` (green)
   - Lock/permanence elements: `#a05c00` (amber)
   - Neutral/bus elements: `#7a6a55` (warm grey)

3. **Accessibility:** Include `<title>` and `<desc>` elements in every SVG for screen readers.

4. **Size:** Keep viewBox dimensions reasonable — 680×400 or similar. Avoid fixed pixel widths; use `width="100%"` for responsive embedding.

5. **Font:** Use `IBM Plex Mono` for labels (matches article typography). Include a fallback: `font-family="IBM Plex Mono, monospace"`.

## Current Diagrams

| File | Used in | Description |
|------|---------|-------------|
| (embedded in article/index.html) | Main article | All current diagrams are inline SVG in the HTML |

Contributions that extract diagrams into standalone SVG files and link them from the article are welcome.

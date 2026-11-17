---
name: 'design-product-favicon'
description: 'Design distinctive, production-ready product favicons and app icon packs from a product concept, existing brand, logo, website, or repository. Use when Codex needs to create, critique, refresh, export, integrate, validate, commit, push, or publish favicon assets for a product, with special emphasis on brand recognition, unique silhouette, 16px readability, favicon.svg, favicon.ico, PNG sizes, apple-touch-icon, web manifest metadata, HTML tags, and GitHub handoff or PR preparation.'
---

# Design Product Favicon

## Overview

Design a favicon system that is distinctive before it is decorative: recognizable at 16px, hard to confuse with generic AI/tool icons, consistent with the product, and ready to ship in a web repository or standalone GitHub package.

## Core Standard

Treat the favicon as the product's smallest brand mark. Reject a concept if it only communicates a category such as "AI", "dashboard", "tool", or "sparkle" but does not create a memorable product-specific shape.

## Workflow

1. Establish the product signal before drawing.
   - Inspect provided product copy, UI, brand files, logos, screenshots, or repository styles.
   - If no brand exists, infer a simple direction from the product name, category, and target audience.
   - Write a one-line recognition claim: "Users should remember this favicon as ___".
   - Name what the favicon must not resemble: generic AI sparkle, random dashboard, default app tile, stock logo, or unrelated category mark.

2. Pass the identity gate before polishing.
   - Define one signature silhouette that still works in black and white.
   - Prefer a proprietary shape, metaphor, cutout, or composition over category decoration.
   - If the concept could fit three unrelated products, redesign it.
   - Compare the idea against common neighbors: browser extensions, SaaS dashboards, AI tools, note apps, and developer tools.

3. Sketch 2-3 low-detail concepts before finalizing.
   - Evaluate concepts by shape first, color second, effects last.
   - Keep the strongest concept and optionally keep one alternate only when it expresses a clearly different brand position.
   - Explain why rejected concepts were too generic or weak at small sizes.

4. Create the master asset as SVG first.
   - Use a square `viewBox`, transparent background by default, and paths/shapes that survive rasterization.
   - Avoid fine strokes, small text, complex gradients, and detailed illustrations.
   - Keep the visual centered with enough optical padding for browser masks and rounded app icon treatments.

5. Export a favicon pack.
   - Use `scripts/build_favicon_pack.py` when a master SVG exists.
   - Default deliverables: `favicon.svg`, `favicon.ico`, `favicon-16.png`, `favicon-32.png`, `favicon-48.png`, `apple-touch-icon.png`, `icon-192.png`, `icon-512.png`, `site.webmanifest`, and `preview.html`.
   - If the target app has existing naming conventions, match them instead of forcing the defaults.

6. Integrate into the product.
   - Place assets where the framework expects them: `public/` for many React/Vite/Next apps, app-specific static folders when present, or the existing favicon path if already configured.
   - Add or update HTML/head metadata only when the app does not already generate it elsewhere.
   - Keep unrelated SEO, branding, or layout changes out of scope.

7. Validate visually and technically.
   - Open `preview.html` or the app in a browser and inspect at 16, 32, 48, 180, 192, and 512px.
   - Verify light and dark backgrounds, pinned-tab/mask behavior if relevant, and that the smallest icon reads instantly.
   - Run the silhouette test: the mark should still be recognizable when color is removed.
   - Run the neighbor test: describe how it differs from common AI/tool favicons.
   - Check that exported files exist, are non-empty, and paths referenced by HTML/manifest are correct.

8. Publish through GitHub when requested.
   - Read `references/github-publish.md` before creating remotes, pushing, or opening PRs.
   - Use a private repo by default for new standalone skill/icon packages unless the user explicitly asks for public.
   - If publishing into an existing product repository, stage only favicon-related files and any necessary metadata updates.

## Design Guidance

Read `references/favicon-design.md` when making design decisions, evaluating options, or explaining tradeoffs. It contains the identity rubric, size matrix, quality bar, and common failure modes.

For visual generation, prefer code-native SVG for the final master. Use bitmap/image generation only for mood exploration or when the user explicitly wants an illustrative raster style, then simplify the final favicon into a clean mark.

## Script Usage

Run the pack builder from the skill directory or copy it into the target repository:

```bash
python3 scripts/build_favicon_pack.py \
  --source path/to/favicon.svg \
  --out public/favicon \
  --name "Product Name" \
  --theme-color "#0F766E" \
  --background-color "#FFFFFF"
```

If the script reports missing converters, install one supported converter (`cairosvg`, `rsvg-convert`, or ImageMagick) or use the macOS fallback when available.

## Output Expectations

Deliver the edited assets, the integration files changed, and a concise summary of the design rationale. Include the recognition claim, why the chosen mark is more identifiable than rejected alternatives, validation results, and any GitHub URL, branch, commit, or blocker from the publish step.
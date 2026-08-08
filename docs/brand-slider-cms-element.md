# Brand / Manufacturer Slider — CMS Element

*A configurable brand-logo slider that merchants place through the Shopware CMS layout builder, backed by a custom CMS data resolver.*

## Business Problem

Distributor and B2B catalogs sell on the strength of the brands they carry. Buyers frequently arrive looking for a manufacturer they already trust, and a visible wall of recognised logos is one of the strongest credibility signals a supplier site can show — particularly for a merchant who is not itself a household name.

Shopware ships manufacturer data as part of the product model, but provides no way to surface it as a curated homepage or landing-page section. The options available to a merchant were all poor:

- **A static image in a text block.** Logos become one flattened picture — no links, no alt text, no ability to add or remove a brand without re-exporting the image, and unreadable on mobile.
- **Hand-written HTML in a CMS block.** Fragile, unstyled, and requires someone comfortable writing markup every time a supplier relationship changes.
- **A third-party slider plugin.** Adds another JavaScript library to the page and another dependency to keep compatible across Shopware upgrades, for one section.

These approaches also made the brand section harder to keep aligned with the catalog and did not provide a structured path from a manufacturer logo to that manufacturer's product listing.

## Solution

A first-class CMS element and block that merchants place through the standard Shopware layout builder alongside the platform's core elements.

**What the merchant does.** Drag the block into a CMS layout, set a section heading, and choose which manufacturers to show. No markup, no image editing, no developer.

**What the shopper gets.** A responsive logo slider where every logo links through to a product listing filtered to that manufacturer. This is the piece that turns a credibility section into a browsing path — a shopper who recognises a brand is one click from that brand's products, and the destination uses Shopware's own listing and filtering, so the feature does not need a custom listing controller and retains the platform's sorting, pagination, and filtering behaviour.

**Behaviour adapts to the data.** The slider configures itself from the number of brands actually selected rather than from a fixed setting:

- If fewer brands are selected than the configured slides-per-view, the layout clamps to the real count — so three brands render as three centred logos, not three logos and three gaps.
- If there is nothing to scroll, the slider, its arrow controls, autoplay, and looping all switch off. A single brand renders as a static logo rather than an autoplaying carousel of one, which is the tell-tale sign of a section that was configured but not thought about.
- Per-breakpoint slide counts derive from that same clamped value, reducing the need for manual retuning when brands are added or removed.

## My Contribution

I designed and implemented the plugin end to end:

- The **CMS data resolver** — criteria construction, batched data collection, and the flattened storefront payload.
- The **administration extension** — CMS block and element registration, the configuration component, and the preview components shown in the layout builder's element picker.
- The **storefront template** — including the data-driven responsive slider configuration and the empty and single-item states.
- The **SCSS** for the element, built against the theme's own colour and spacing custom properties rather than hard-coded values.
- The **service wiring** and plugin scaffolding.

## Architecture

**Services.** A single CMS element resolver, registered in the container and tagged so Shopware's CMS layer discovers it automatically. It receives the manufacturer repository and a logger through constructor injection. There is no custom controller, custom entity, or plugin-owned database table. The plugin composes existing platform data, which avoids additional schema and reduces maintenance associated with custom persistence.

**Data resolution.** The resolver implements Shopware's two-phase CMS resolution contract:

- **Collect phase** — builds a criteria object restricted to the configured manufacturer IDs, applies the configured limit, and adds the media association needed for logos. The criteria is returned in a criteria collection keyed by the slot's unique identifier rather than being executed immediately.
- **Enrich phase** — reads the resolved result set and maps it to a flat array structure containing only what the template renders: identifier, translated name, logo URL, and the manufacturer-filtered listing link. That structure is attached to the slot as a data struct.

Returning criteria rather than querying directly is the important architectural decision, and the reason is covered under Performance below.

**Configuration handling.** Element configuration arrives from the administration in more than one shape depending on how the value was entered, so the resolver normalises identifiers from either a delimited string or an array, trimming and filtering as it goes. Every configuration read applies a sensible default, so a partially-configured element renders rather than erroring.

**DAL and repositories.** The manufacturer repository is injected as a service; associations are declared on the criteria rather than lazily loaded per item, so translated names and media resolve in the same round trip.

**Administration components.** A CMS block and a CMS element are registered against Shopware's CMS service. The element supplies a component for the layout-builder canvas, a configuration component bound to the element's config, and a preview component for the element picker. The block declares its default spacing and sizing and maps its content slot to the element.

**Storefront templates.** A block template resolving the content slot, and an element template rendering the slider. Both live under the plugin's own Twig namespace and are registered through the plugin's CMS configuration, so Shopware resolves them without a theme change.

## Shopware Concepts Used

- **Symfony services and dependency injection** — service definition with constructor-injected repository and logger, registered via a service tag rather than by manual registration.
- **Service tagging** — the CMS data resolver tag, which is how the platform discovers custom element resolvers without any plugin bootstrap code.
- **DAL and criteria** — criteria construction with ID restriction, limit, and declared associations; repository search executed by the CMS layer.
- **CMS elements and blocks** — the full custom-element contract across both administration registration and storefront rendering.
- **Structs** — array structs used to pass a resolved payload from resolver to template.
- **Twig** — element and block templates, slot resolution, and JSON-encoded configuration passed to the storefront slider.
- **Administration** — Vue-based CMS component registration, configuration components bound to element config, and preview components.
- **Translations** — translated manufacturer names read through the translation accessor rather than the raw field, and translated snippet keys for control labels.
- **Configuration** — per-element configuration with defaults, handled entirely through the CMS element config contract.

## Performance

**Coordinated CMS data loading.** The resolver returns criteria during the collect phase rather than executing repository searches directly. This allows Shopware's CMS data-resolution layer to coordinate loading with the other elements on the page and avoids introducing an independent repository lookup inside the element's rendering path.

**Minimal associations.** Only the media association required for logos is declared. Manufacturer records carry more relational data than this section needs, so unrelated associations are not requested for the slider.

**A flat render payload.** Entities are mapped to a small array of scalars before reaching Twig. The template therefore renders prepared values rather than traversing entity relations inside the loop.

**No additional carousel dependency.** The slider reuses the storefront's existing slider implementation and is driven by configuration in the markup, avoiding another third-party carousel library for this feature.

**Lazy-loaded images.** Logos below the fold defer loading, reducing initial image work when a section contains many brands.

**Work avoided when there is nothing to scroll.** Autoplay, looping, and controls are disabled when the brand count does not exceed the visible slots, so a static set of logos is not treated as an interactive carousel.

## Multi-Channel Support

Element configuration is stored per CMS layout, and layouts are assigned per sales channel, so each channel can run its own brand selection, heading, and slider behaviour from the same catalog — a B2B channel can foreground its industrial suppliers while a retail channel foregrounds consumer brands, with no duplicated data.

Manufacturer names resolve through the translation layer, so a multi-language channel receives correctly translated names. Logo links resolve against the requesting channel's context, keeping the destination aligned with the channel the shopper is browsing.

## Accessibility

- The slider region carries a landmark role and is labelled with the section heading, so screen-reader users encounter it as a named region rather than an unexplained group of images.
- Each logo link has an accessible name derived from the brand name, so the link is announced with meaningful brand text rather than relying on image or URL text alone.
- Logo images carry the brand name as alt text; brands without an uploaded logo fall back to a styled text tile rather than a broken image.
- Previous and next controls use translated labels rather than icon-only markup.
- The slider region is keyboard focusable.

## Screenshots

> **To be added** — sanitized or demonstration data only. Suggested: the element's configuration panel in the layout builder, and the rendered slider on desktop and mobile using placeholder brand logos.

---

[← Back to portfolio overview](../README.md)

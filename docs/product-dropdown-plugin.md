# Product Detail Dropdown Sections

*Merchant-composed accordion sections on the product page, assembled from property groups, custom fields, and media without a developer.*

## Business Problem

Technical products carry far more information than a product page can show at once: specification tables, material and compliance data, dimensional drawings, datasheets, installation guides, warranty terms. A trade buyer needs all of it, but not simultaneously — they arrive looking for one specific thing.

Shopware's product page presents a fixed structure. Extending it meant a developer editing templates, and every subsequent change — adding a compliance section, reordering, surfacing a newly created custom field — meant another ticket and another deployment. In practice this produced two bad outcomes: product pages that showed too little because adding a section wasn't worth the cost, or pages carrying hard-coded sections that had drifted out of relevance because removing them wasn't worth it either.

The underlying problem is that **which** attributes matter is a merchandising decision that changes as the catalog changes, but it had been encoded as a development decision.

## Solution

A configuration-driven accordion on the product page, composed entirely in the administration.

**A visual section builder.** Instead of a text field expecting hand-written configuration, the plugin supplies a purpose-built administration component. The merchant creates named sections, adds items to each from the store's actual property groups and custom fields, and reorders both sections and items. The store's real data is presented as pickers, so a merchant chooses from what exists rather than typing identifiers and hoping.

**Items render according to what they contain.** The plugin inspects each configured value at render time and chooses the presentation:

- **Property groups** render as a specification table of name/value pairs, with multiple options in one group collapsed into a single readable row.
- **Custom fields holding media** render as a labelled list of download links — the mechanism by which datasheets, drawings, and certificates reach the page.
- **The core product description** is available as a section item, so long-form copy can be placed alongside specifications rather than only in its default position.
- **Everything else** renders as text or rich content.

Media values are detected from the configured field value shape: identifier-like values are resolved as media and rendered as links, while other values follow the text-content path. The merchant selects a custom field without maintaining a second content-type setting.

**Empty content disappears.** Items with no value are skipped, sections whose items all resolve empty are dropped, and if nothing resolves the page renders unchanged. A configuration built for a fully-specified product does not produce a page of empty accordions on a sparsely-specified one — which is what makes one configuration usable across an entire catalog rather than requiring per-product curation.

**Two placement options.** The sections render on the product detail page, and are also exposed as a CMS element and block so they can be positioned within a CMS-managed product layout.

## My Contribution

I designed and implemented the plugin end to end:

- The **event subscriber** — criteria enrichment, configuration decoding, and the section-building logic including value-type detection and media resolution.
- The **administration section builder** — the component, its data loading from property groups and custom fields, and the ordering and mutation handling.
- The **plugin configuration schema** binding the builder to a configuration field.
- The **CMS block and element** registration and their preview components.
- The **storefront templates** for both placements.

## Architecture

**Services.** A single event subscriber, registered in the container with the system configuration service and the media repository injected, and discovered through the kernel event subscriber tag. No custom entities, tables, migrations, or controllers are required. The plugin composes data that already exists, avoiding plugin-owned persistence and reducing schema-related maintenance.

**Subscribers.** The subscriber listens to two product page events, and the pairing is the key architectural decision:

- **Product page criteria event** — adds the property and property-group associations to the page's criteria *before* the product is loaded. This is what makes the second listener possible without extra queries.
- **Product page loaded event** — reads the channel-scoped configuration, builds the sections, and attaches them to the page as a named extension.

Enriching the criteria rather than re-fetching the product in the loaded listener allows the required property data to arrive with the product page load, avoiding a separate product re-query for the feature.

**Configuration handling.** Configuration is stored as a JSON string and decoded defensively — a non-string, malformed, or non-array value yields an empty configuration and the subscriber returns without touching the page. Sections and their items are sorted by stored position on decode, and again after building, so ordering is stable regardless of how the configuration was persisted.

**Value resolution.** Each item type has its own resolution path:

- *Property groups* are resolved by filtering the product's already-loaded properties to the configured group, reading translated group and option names, de-duplicating, and joining multiple options into one value.
- *Custom fields* are read from the product, with the core description special-cased to read from the translation accessor.
- *Media detection* tests values against an identifier pattern — accepting both a single value and an array — and resolves matches through the media repository into link data. A non-matching value short-circuits to text rendering without a query.

**DAL and repositories.** The media repository is injected as a service and queried only when a value looks like media identifiers. Property data arrives through the enriched product-page criteria, so the feature does not require a separate property repository query.

**Structs.** Built sections are attached to the page as an array struct under a namespaced key, consumed from Twig.

**Administration components.** A section builder component, injected with the repository factory, loads property groups and custom fields to populate its pickers. It maintains sections and items in local state with explicit add, remove, and move operations, renumbering positions on each mutation, and emits its serialised value as the configuration changes so the custom field remains compatible with the administration configuration binding used by the plugin screen. The component is bound to a configuration field by component name, so it replaces the default text input in the plugin's configuration screen.

CMS block and element registration, with canvas and preview components, provides the second placement path.

**Storefront templates.** A product-detail template renders the accordion; a CMS element template includes the same partial, so both placements share the same rendering path and are easier to keep consistent.

## Shopware Concepts Used

- **Symfony services and dependency injection** — constructor injection, service tagging for subscriber discovery.
- **Events and subscribers** — paired criteria and page-loaded events on the product page.
- **Criteria enrichment** — adding associations to a page's criteria before load rather than re-querying afterwards.
- **DAL, criteria and associations** — nested property-group associations; identifier-restricted media queries.
- **Page extensions and structs** — array structs attached to the page under a namespaced key.
- **System configuration** — a configuration field rendered by a custom administration component, read per sales channel.
- **Custom fields and property groups** — consumed as merchant-selectable content sources.
- **Media entities** — resolved to URLs, filenames, and titles for download links.
- **Translations** — translated property group names, option names, and product description via the translation accessor.
- **CMS elements and blocks** — a second placement path for the same output.
- **Twig** — accordion rendering with per-type branching, shared partial across two placements.
- **Administration** — a custom Vue component bound to a configuration field, repository factory usage, and CMS component registration.

## Performance

**Criteria enrichment instead of re-fetching.** Property and group data are added to the product page criteria before load, so the feature avoids a separate product re-query solely to collect those associations.

**Media queried only when needed.** Media resolution runs only when a value matches the identifier pattern, and matching identifiers are resolved in a batched repository call rather than one lookup per file. Text and specification content requires no additional repository lookup beyond data already available on the loaded product page.

**Early exits.** The subscriber returns when configuration is absent or undecodable, and again if no sections resolve, preventing section-building and media work when the feature is not configured for the channel.

**Work proportional to configuration.** Only configured groups and fields are resolved. Property filtering runs over the product's already-loaded collection in memory.

**No render-time traversal.** Sections are fully built into plain arrays before reaching Twig, so template rendering performs no entity traversal or lazy loading.

**Bootstrap-native accordion.** Expand and collapse behaviour uses the storefront's existing Bootstrap components, so the feature ships no JavaScript of its own.

## Multi-Channel Support

Configuration is read **scoped to the sales channel of the current request**, so each channel defines its own sections, its own item selection, and its own ordering from the same catalog and the same plugin.

This matters for the intended use case. A trade channel can surface compliance documents, warranty terms, and full dimensional specifications, while a retail channel on the same installation shows a shorter, less technical set — with no duplicated product data, no per-channel templates, and no deployment to change either.

Property group names, option names, and product descriptions resolve through the translation layer, and media URLs resolve in the requesting channel's context, so multi-language and multi-domain channels each receive correct content.

## Accessibility

- Sections use the storefront's existing accordion pattern, inheriting its established expanded/collapsed state, control relationships, and keyboard behaviour rather than reimplementing disclosure semantics.
- Each section header is a real button with a programmatic relationship to the panel it controls.
- Section headings use real heading elements, so sections appear in the document outline and can be navigated by heading.
- Specification content renders as a real table with row headers, so name/value relationships are announced correctly rather than being conveyed by layout alone.
- Download links carry the media title or filename as their text, so assistive technology receives meaningful link text instead of relying on a bare URL.

## Screenshots

> **To be added** — sanitized or demonstration data only. Suggested: the section builder in the plugin configuration screen, and the rendered accordion on a product page showing a specification table and a downloads section. Use demonstration product data and placeholder document names.

---

[← Back to portfolio overview](../README.md)

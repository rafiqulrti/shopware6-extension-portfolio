# B2B Storefront Enhancements

*A companion plugin surfacing Shopware Commercial B2B data — multi-warehouse stock, shopping list imagery, and option label resolution — in the storefront.*

## Business Problem

Shopware Commercial provides substantial B2B capability, but data available in the platform is not automatically presented in the form a buyer needs. Three storefront gaps mattered on this store, each affecting buyer experience or support effort:

**Stock is a single number, but the business has several warehouses.** Multi-warehouse tracks stock per location, yet the product page shows one aggregate figure. A trade buyer's real question is not "is there stock" but "is there stock I can collect today, or that ships from the depot nearest me". Without per-location visibility, buyers may need to contact sales or support before completing an order.

**Shopping lists showed products without imagery.** B2B shopping lists routinely run to dozens of line items and are reviewed repeatedly before becoming an order. A list of text rows is slow to scan and easy to misread, particularly across variants of similar products. The underlying difficulty is that variants frequently have no cover image of their own, inheriting it from the parent — so a naive image lookup returns nothing for exactly the products most likely to appear on a trade list.

**Option values displayed as stored keys rather than labels.** Custom fields defined as option sets store a key and display a label. Where a value reached the storefront outside the administration's own rendering, buyers saw the raw stored key — internal-looking values on a customer-facing page, in the wrong language on a localised channel.

## Solution

A companion plugin to the storefront theme that resolves each gap at the page-load layer, so templates receive data ready to render.

**Per-warehouse stock on the product page.** Warehouse and warehouse-group stock data is resolved and attached to the product page. Buyers see stock by location instead of one opaque total, turning a phone call into a self-service decision.

Crucially, it **falls back to the parent product** when a variant carries no warehouse data of its own. Warehouse assignment is commonly maintained at parent level, and without the fallback the feature would appear broken on precisely the variant pages where it is most needed.

**Imagery on shopping lists.** Line items are enriched with cover imagery, with the same parent-fallback reasoning: variants lacking their own cover have their parent's image resolved and applied, reducing missing imagery on variant-heavy shopping lists.

**Human-readable option labels.** Stored option keys are mapped to their configured labels, chosen by locale, so buyers see intended wording in their own language.

## My Contribution

I designed and implemented the plugin: all three subscribers including their fallback strategies and batched loading, the locale-aware label resolution, the storefront template integration, and the service wiring. It was built as a companion to the storefront theme I also developed, keeping presentation-agnostic data resolution in a plugin and leaving the theme responsible for rendering.

## Architecture

**Services.** Three event subscribers, registered with autowiring and autoconfiguration and discovered through the kernel event subscriber tag, each receiving the repository it needs. No entities, tables, migrations, or controllers — the plugin resolves and shapes data that already exists in the platform and its commercial extensions.

The separation of concerns is deliberate: each subscriber owns one page-level concern and can be enabled, disabled, or replaced independently. Registering three narrow subscribers rather than one broad one reduces coupling between shopping-list behaviour, product-page behaviour, and label resolution.

**Subscribers.**

*Warehouse information* listens to the product page-loaded event and:

- Loads the product with warehouse and warehouse-group associations declared on the criteria.
- Extracts warehouse rows — identifier, name, and stock — and warehouse group data from the loaded entity extensions.
- Falls back to loading the **parent product** when the variant yields no warehouse data.
- Returns without attaching anything when no data resolves, so pages for products outside multi-warehouse handling are untouched.
- Attaches the result to both the product and the page, so templates can reach it from whichever context they render in.

*Shopping list imagery* listens to the commercial shopping-list line-items-loaded event and:

- Collects the product identifiers across the whole set of line items.
- Loads them in **one batched query** with the cover media association.
- Identifies items still lacking cover media, collects their parent identifiers, de-duplicates, and issues a **second batched query** for those parents.
- Applies resolved covers back onto the line items.

The loading strategy batches product identifiers and performs a second parent lookup only for items that still need inherited cover media, avoiding a repository lookup for each line item.

*Option label resolution* listens to the product page-loaded event and:

- Reads the configured option value from the product's custom fields, returning immediately when absent.
- Loads the custom field definition and extracts its option configuration.
- Builds a key-to-label map and resolves the stored value or values through it, falling back to the raw value when no label exists — so an unmapped key degrades to something readable rather than to blank.
- Selects labels by the channel's locale, then by English fallbacks, then by any available label.
- Attaches the resolved labels as an extension and also writes a display value back onto the product's custom fields, so simple templates can read a formatted string directly.

**DAL, criteria and repositories.** Product, custom field, and commercial warehouse data is accessed through injected repositories with explicitly declared associations. Every multi-record lookup is identifier-batched rather than iterated.

**Entities.** Commercial multi-warehouse entities are consumed through their own collections and entities, and validated with instance checks before use — so a store without the commercial extension, or with it disabled, degrades rather than fatals.

**Structs.** Resolved data is attached as array structs under namespaced keys, on the product and the page as appropriate.

**Storefront templates.** A product detail template extends the core template and overrides a single block to render resolved values after the buy area, calling the parent so core content continues to render.

## Shopware Concepts Used

- **Symfony services and dependency injection** — autowiring and autoconfiguration with per-subscriber repository injection.
- **Events and subscribers** — storefront product page events and Shopware Commercial B2B shopping list events.
- **Shopware Commercial integration** — multi-warehouse product, warehouse, and warehouse-group entities and collections; B2B shopping list events.
- **DAL, criteria and associations** — nested associations for warehouse and cover media; identifier-batched queries.
- **Entity extensions** — reading commercial data attached to products as extensions, and attaching the plugin's own resolved data the same way.
- **Page and entity extensions** — data attached to both the page and the entity so templates can reach it from either context.
- **Structs** — array structs carrying resolved payloads to Twig.
- **Custom fields** — option-set definitions read from the custom field entity to resolve stored keys to labels.
- **Translations and locale resolution** — labels selected by the channel's language locale with a defined fallback chain.
- **Twig** — targeted block override with parent preservation.
- **Variant and parent product handling** — explicit parent fallback where variant-level data is absent.

## Performance

**Batched loading by design.** Shopping-list enrichment loads products as a batch and performs a second batched parent lookup only for items that still need inherited covers. This avoids a repository lookup for each individual line item.

**Fallback queries are conditional.** The parent lookup runs only when items actually lack cover media, and only for the de-duplicated set of parent identifiers. When every item already has cover media, the parent lookup is skipped.

**Early exit before work.** Every subscriber returns before querying when its precondition fails — no configured option value, no warehouse data, no line items. Pages outside each feature's scope return before repository work is performed.

**Associations declared, not traversed.** Warehouse and cover associations are declared on criteria so data arrives in the same query, avoiding lazy loading inside loops.

**Resolution at page load, not render.** All mapping happens in the subscriber, so templates render prepared arrays and strings with no entity traversal or per-row queries.

**A pre-formatted display value** is written alongside structured label data, so straightforward templates can render a string without iterating.

## Multi-Channel Support

**Locale-aware label resolution.** Option labels are selected using the language locale of the current channel's context, with a defined fallback chain — configured locale, then English variants, then any available label. A buyer on a localised channel sees labels in their language, and a channel whose locale has no configured label still shows readable text rather than a raw key or a blank.

**Channel-scoped data access.** Product and media resolution runs in the request's context, so products, imagery, and stock reflect what is visible in the requesting channel.

**Per-channel warehouse relevance.** Warehouse and warehouse-group data is resolved per product page in channel context, so channels serving different regions surface the stock picture relevant to them.

## Accessibility

- Resolved values render as real lists and labelled groups rather than as delimited strings, so screen readers announce discrete values rather than a run-on line.
- Shopping list imagery makes list rows scannable visually while leaving the existing text content intact, so the enhancement adds a channel of information without removing one.
- Labels replace internal keys in customer-facing output, which is as much an accessibility improvement as a cosmetic one — an option announced as a stored key conveys nothing to a screen reader user.

## Screenshots

> **To be added** — sanitized or demonstration data only. Suggested: the product page showing per-warehouse stock, a shopping list with resolved product imagery, and resolved option labels on a product page. Use demonstration catalog data with placeholder warehouse names and stock figures.

---

[← Back to portfolio overview](../README.md)

# Product Group Page — Specification Table Listings

*Category pages rendered as sortable, searchable specification tables with merchant-defined columns, progressive loading, and inline product detail.*

## Business Problem

For distributors selling by specification rather than by style — fittings, sensors, fasteners, cable assemblies, filters — the standard product grid is actively counterproductive.

Buyers in these categories already know the rating, dimension, or material they need. What they are doing is *comparing variants*, and a grid of near-identical thumbnails gives them nothing to compare against. The differentiating value sits on the product detail page, so finding the right variant means opening and closing a dozen pages, holding numbers in your head. Merchants in this space routinely lose orders either to a competitor whose site shows a table, or to their own PDF catalog — which, tellingly, buyers often prefer to the website.

The requirement was to present a category as buyers actually shop it: rows of products, columns of the specifications that matter, sortable and searchable, without leaving the page. Three constraints made this non-trivial:

- **Which specifications matter varies by category.** A cable category compares on gauge and length; a sensor category compares on output type and voltage. A fixed column set would serve neither.
- **Specification data lives in several places.** Some values are core product fields, some are property group options, some are custom fields. A useful table has to draw from all three in the same row.
- **These categories are large.** Rendering every row of every child category up front would produce unusable page weight.

## Solution

A category-level feature that converts a category page into a specification table, configured per category by the merchant.

**Merchant-defined columns.** An administrator enables the table on any category and builds its column set from a picker offering core product fields, the store's property groups, and its custom fields — mixed freely in one table and reordered as needed. A category with no column configuration falls back to a sensible default set, so the feature produces a working table the moment it is switched on.

**Child categories become sections.** A parent category renders each of its child categories as its own table, so a buyer sees the whole product family organised on one page rather than navigating a level at a time.

**Interaction happens without page loads.** Each table supports:

- **Column sorting**, handled client-side for already-loaded rows and delegated to the server where a column maps to a server-side sorting key — so sorting is instant when it can be and correct when it must be.
- **Search within the table**, debounced so typing doesn't fire a request per keystroke.
- **Inline row detail** — clicking a row expands product detail, including imagery, beneath it. The buyer compares, drills into one row, and returns to the comparison without losing their place or their scroll position.
- **Progressive loading.** Tables render an initial, configurable number of rows with a merchant-labelled "view all" control for the remainder.

**Tables load as they are needed.** Sections load when scrolled into view rather than all at page load, so a page containing many child categories stays fast regardless of how many it holds.

## My Contribution

I designed and implemented the plugin end to end:

- The **backend service layer** — the page loader, column resolution and row mapping service, and the product loading service with its fallback strategy.
- The **AJAX controller** and its routes for table reloads and inline product detail.
- The **event subscriber** that activates the feature per category, and the page structs that carry data to the templates.
- The **migration** that provisions the category configuration fields and their administration field set.
- The **administration extension** — the column builder component and the category detail page extension that hosts it.
- The **storefront JavaScript plugins** — table interaction and the section carousel.
- The **storefront templates and SCSS**, including the listing override that swaps in the table presentation.

## Architecture

**Services.** All services are registered with autowiring and autoconfiguration, keeping the container definition small and letting service tags drive discovery:

- A **page loader** that assembles the table data for a category page.
- A **column resolver** that turns stored configuration into normalised column definitions and maps product entities to rows.
- A **product loader** that fetches listings for each child category.
- An **AJAX controller** exposing two storefront routes.
- A **navigation subscriber** that activates the feature.

The container also declares explicit aliases for the abstract navigation, category, and product listing routes, so the plugin depends on the platform's abstract route contracts rather than concrete implementations — meaning other plugins decorating those routes remain in the chain.

**Subscribers.** A subscriber on the navigation page-loaded event reads the category's configuration field and returns immediately unless the feature is enabled for that category. When it is, it delegates to the page loader and attaches the result to the page as a named extension. The check is a single array read on an already-loaded entity, so categories not using the feature pay effectively nothing.

**Controllers.** A storefront controller with two AJAX routes, both scoped to the storefront and marked as XML HTTP requests:

- **Table reload** (POST) — re-resolves columns and rows for a category and returns a rendered table body fragment. Returning rendered markup rather than JSON means sorting, searching, and pagination all reuse exactly the same Twig used for the initial render, so there is no second rendering path to keep in sync — a common source of drift in table features.
- **Product detail** (GET) — returns a rendered detail fragment for one product, or a no-content response when the product isn't available in the channel.

Both responses set explicit no-store cache headers, since their content is context- and configuration-dependent.

**DAL and criteria.** Product loading goes through the platform's abstract product listing route, so listings inherit the store's own listing behaviour — availability rules, channel visibility, filtering and sorting — rather than reimplementing it. Criteria are built dynamically: the column resolver reports which associations the configured columns require, and only those are added. A table with no property columns never loads the property association.

The product loader also implements a **fallback path**. If the listing route returns nothing, it retries with a direct repository search filtered by channel availability and category membership, with exact total counting and search term applied. Listing routes can legitimately return empty for configuration reasons that have nothing to do with whether products exist in the category; the fallback means a misconfiguration degrades to a working table rather than an empty page.

**Entities and custom fields.** No custom entity or table. A migration provisions a category custom field set containing an enable flag and a JSON column configuration, plus its administration relation. Using custom fields rather than a bespoke entity means the configuration is part of the category record — it exports, imports, syncs, and versions with the category through the platform's own mechanisms, and there is no schema of the plugin's own to maintain across upgrades.

**Repositories.** Sales-channel-scoped product repositories are injected by service ID through autowiring attributes, so all product access is channel-aware by construction rather than by convention.

**Structs.** Two structs carry data from the loader to templates — a page struct holding the category, its child sections, and the view-all settings; and a child struct holding one section's category, columns, rows, load state, and total.

**Administration components.** A column builder component loads the store's property groups and custom fields through the repository factory and presents them alongside core field options as one combined picker. Columns can be added, removed, and reordered, with positions renumbered on every change. The category detail page is extended to host the builder, and the raw custom field set is filtered out of the generic custom fields area — so the merchant sees a purpose-built column builder instead of a raw JSON field, which is the difference between a feature that gets used and one that gets a support ticket.

**Storefront templates.** A listing element override swaps in the table presentation when the page carries the feature's extension, keeping the original listing in the DOM but hidden and marked aria-hidden so core listing behaviour that depends on it continues to function. Table markup is split into a wrapper, a separately-renderable body, and a row detail fragment — the split that lets the AJAX routes re-render exactly one piece.

**Storefront JavaScript.** Two plugins built on the platform's plugin base class:

- **Table plugin** — debounced search, dual-mode sorting, expandable row detail, view-all handling, intersection-observer lazy loading, and synchronisation with the core listing so filter changes reload the table.
- **Carousel plugin** — horizontal scrolling of section navigation with smooth scroll-into-view.

## Shopware Concepts Used

- **Symfony services and dependency injection** — autowiring and autoconfiguration, constructor injection, autowire attributes for service-ID-specific repositories, and abstract route aliasing.
- **Events and subscribers** — the navigation page-loaded event, with per-category activation.
- **Storefront controllers and routes** — attribute-defined routes with storefront scope, route imports by attribute, and rendered-fragment responses.
- **Page loaders, page extensions and structs** — a dedicated loader producing typed structs attached to the page as an extension.
- **DAL, criteria and associations** — dynamically composed criteria, availability and category filters, exact total count mode, search terms, and conditionally added associations.
- **Abstract routes** — navigation, category, and product listing routes consumed through their abstract contracts so decoration still works.
- **Migrations** — a migration step provisioning a custom field set, its entity relation, and its fields idempotently.
- **Custom fields** — category-level configuration stored as custom fields rather than a bespoke entity.
- **Property groups and custom fields as data sources** — resolved per product into table cells.
- **Twig** — template overrides, separately-renderable fragments, and namespaced includes.
- **Administration** — a custom Vue component, component override of a core detail page, and repository factory usage.
- **Configuration** — plugin configuration read per sales channel.
- **Property accessor** — safe, readability-checked traversal of nested entity paths for core column values.

## Performance

Performance was the main design pressure, since the feature exists to render a great deal more data per page than a standard listing.

**Lazy loading by intersection.** Sections load their tables when scrolled into view rather than at page load. A page with many child categories therefore performs work proportional to what the buyer actually looks at, not to how the catalog is structured.

**Associations resolved from configuration.** The column resolver reports which associations the configured columns actually need, and only those are added to the criteria. A table of core fields and custom fields never pays for the property association — a meaningful saving, since property loading is among the more expensive associations on a large product set.

**Bounded result sets at every level.** The product loader applies a default limit when none is supplied, the page loader applies the configured initial limit per section, and the view-all control raises it on demand. No code path can issue an unbounded product query.

**Client-side sorting where it is safe.** Sorting already-loaded rows happens in the browser against pre-computed sort values, with no request at all. Sort values are normalised server-side during row mapping — lower-cased for strings, cast to float for numbers, with empty numerics sorted deterministically — so the browser compares prepared scalars rather than parsing formatted display strings. Only columns backed by a server-side sorting key round-trip.

**Debounced search.** Search input is debounced before triggering a reload, so a typed query produces one request rather than one per character.

**Fragment rendering, not full pages.** AJAX responses render only the table body or a single row's detail, so an interaction transfers a fragment rather than a page.

**Flat row payloads.** Products are mapped to plain arrays of scalars before reaching Twig, so no entity traversal or lazy loading happens inside a render loop over hundreds of rows.

**Explicit cache headers.** AJAX responses are marked no-store, preventing intermediate caches from serving one context's table to another.

## Multi-Channel Support

The feature is channel-aware at every layer.

- **Product loading** runs through the sales-channel product listing route and sales-channel-scoped repositories, so tables show only products visible and available in the requesting channel, at that channel's prices, under that channel's rules.
- **Availability filtering** in the fallback path and the row detail route is explicitly scoped to the current channel's identifier, so a product hidden in one channel cannot surface through a direct AJAX request in another.
- **Plugin configuration** — the initial row limit and the view-all label — is read scoped to the current sales channel, so each channel sets its own density and wording.
- **Column configuration is per category**, and because categories are assigned to channels through the navigation tree, a channel-specific category branch carries its own table layout.
- **Property and custom field values** resolve through the translation layer, so multi-language channels receive translated specification values.

## Accessibility

- Sortable column headers are real buttons rather than click handlers on header cells, so sorting is keyboard reachable and announced as an interactive control.
- The search input carries an explicit accessible label in addition to its placeholder.
- The hidden core listing is marked both `hidden` and aria-hidden, so it is removed from the accessibility tree rather than merely visually hidden — assistive technology sees one product list, not two.
- Sort direction indicators are marked decorative, leaving the button's accessible name clean.
- Wide tables scroll within their own container rather than forcing the page to scroll horizontally.

## Screenshots

> **To be added** — sanitized or demonstration data only. Suggested: the column builder on the category detail page, a rendered specification table with sorting and search, an expanded inline row detail, and the mobile presentation. Use demonstration catalog data with placeholder product numbers and specifications.

---

[← Back to portfolio overview](../README.md)

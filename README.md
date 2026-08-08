# Shopware 6 Development Portfolio — Case Studies

A collection of sanitized case studies covering Shopware 6 website delivery,
custom plugins, administration extensions, CMS development, storefront
customization, data workflows, B2B features, and theme development completed
as part of my work at Harris Digital.

> **Portfolio and confidentiality notice:** Production source code and
> client-specific implementations are private and remain the property of
> Harris Digital and/or its clients. This repository contains no proprietary
> source code, credentials, private repository addresses, customer data,
> production configuration, or confidential client information.

## Website / Project Case Studies

| Case study | What it covers |
|---|---|
| [Laticrete — Shopware 6 Full-Stack Ecommerce Development](docs/laticrete-shopware6-project.md) | End-to-end Shopware delivery across storefront, CMS, custom plugins, theme work, catalog data, deployment, and production support |
| [Basetheme — Shopware 6 Full-Stack Ecommerce Development](docs/basetheme-shopware6-project.md) | Full-stack Shopware implementation covering storefront development, reusable theme work, CMS, plugins, data, configuration, and infrastructure support |

## Extension, CMS & Theme Case Studies

| Case study | What it covers |
|---|---|
| [Product Group Page — Specification Table Listings](docs/product-group-page-plugin.md) | Storefront controllers and routes, page loader, service layer, migration and custom fields, administration column builder, and lazy-loaded interactive tables |
| [Product Detail Dropdown Sections](docs/product-dropdown-plugin.md) | Criteria enrichment, configuration-driven content assembly, media resolution, custom administration component, and CMS element |
| [B2B Storefront Enhancements](docs/b2b-storefront-enhancements.md) | Shopware Commercial integration, multi-warehouse stock, shopping-list imagery, locale-aware label resolution, and batched loading |
| [Brand / Manufacturer Slider — CMS Element](docs/brand-slider-cms-element.md) | Custom CMS element and block, CMS data resolver, DAL criteria, administration configuration, and responsive storefront slider |
| [Featured Category Sections — CMS Elements and Page Extension](docs/featured-category-cms-elements.md) | CMS elements plus a configuration-driven page extension via event subscriber, with sales-channel-scoped configuration |
| [Custom Storefront Theme](docs/custom-shopware-theme.md) | Theme configuration, Twig inheritance, SCSS architecture, storefront JavaScript plugins, CMS styling, responsive behaviour, and upgrade-conscious design |

## My Shopware Experience

I have delivered two Shopware 6 ecommerce websites and developed multiple
custom plugins and a custom storefront theme. My Shopware work includes CMS
elements and blocks, storefront controllers and routes, page loaders and event
subscribers, database migrations, custom administration components, catalog
work, and integrations with Shopware Commercial B2B features.

For Shopware projects I work across the full application stack, including:

- Technical planning and Shopware architecture
- Symfony and PHP backend development
- Shopware DAL entities, repositories, criteria, and associations
- Custom plugins and services
- Administration extensions using Vue-based Shopware components
- Custom product, category, and entity fields
- Storefront controllers, subscribers, routes, and page extensions
- Twig storefront templates and CMS rendering
- Shopware CMS elements, blocks, layouts, and Shopping Experiences
- Theme development and storefront customization
- Data import, migration, and synchronization
- API and third-party integrations
- Sales-channel-scoped configuration and multi-channel functionality
- Performance investigation, debugging, and production troubleshooting
- Deployment, Linux, Nginx, PHP-FPM, Redis, OpenSearch, and server maintenance

## My Responsibilities

I work across the complete Shopware 6 delivery lifecycle, from technical
planning through development, deployment, and production support.

Depending on the project, my responsibilities include:

- Shopware solution architecture and technical planning
- PHP and Symfony backend development
- Custom plugin architecture and implementation
- DAL repositories, criteria, entities, associations, and migrations
- Administration extensions and CMS components
- Twig storefront development and custom theme implementation
- JavaScript and SCSS storefront functionality
- Sales-channel-scoped configuration and multi-channel behaviour
- Product and catalog data migration
- API and third-party integrations
- Performance investigation and production troubleshooting
- Code review, debugging, deployment, and ongoing maintenance
- Linux, Nginx, PHP-FPM, Redis, and OpenSearch support

## Engineering Approach

Across the case studies in this repository, I focus on a few recurring
principles:

- Extend Shopware through supported services, events, DAL, CMS, and theme
  inheritance instead of modifying core files.
- Keep merchant-facing configuration inside the administration wherever it
  reduces repeated development work.
- Scope data and configuration to the active sales channel when channel
  behaviour differs.
- Load only the associations and records required by the current feature and
  avoid per-item repository lookups inside render loops.
- Keep storefront markup responsive and accessible, while reusing platform
  components where practical.
- Treat maintainability and upgrade effort as design constraints rather than
  post-launch cleanup tasks.

## Technologies

`Shopware 6` `PHP` `Symfony` `Twig` `Vue.js`
`DAL` `MySQL` `JavaScript` `SCSS` `REST APIs`
`Linux` `Nginx` `PHP-FPM` `Redis` `OpenSearch`

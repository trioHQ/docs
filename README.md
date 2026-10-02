<p align="center">
  <a href="https://docs.trio.com.br">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="images/LOGO_TRIO_Prancheta1.png">
      <img alt="Trio" src="images/LOGO_TRIO-02.png" width="220">
    </picture>
  </a>
</p>

# Trio Documentation

[![Live docs](https://img.shields.io/badge/docs-docs.trio.com.br-16A34A.svg?style=flat)](https://docs.trio.com.br)
[![Built with Mintlify](https://img.shields.io/badge/built%20with-Mintlify-0D9373.svg?style=flat)](https://mintlify.com)
[![Contributions welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg?style=flat)](#contributing)

Source for the public developer documentation at **[docs.trio.com.br](https://docs.trio.com.br)**.

[Trio](https://www.trio.com.br) is a complete banking solution for companies, designed to automate financial operations, move money at scale, and manage corporate treasury. Trio is authorized by the Central Bank of Brazil (No. 619, ISPB 49931906) and exposes its infrastructure through a REST API so you can embed financial services directly into your product.

The documentation covers:

- **Banking API** — bank accounts, virtual accounts, counterparties, cards, and documents
- **Pix** — static and dynamic QR codes, Pix keys, Pix Automático (recurrences), biometrics, and refunds
- **Boleto** — issuing, updating, cancelling, and paying boletos
- **Payments** — Pix transfers, utility bills, taxes (ISS, DAS, DARF), and internal transfers
- **Webhooks** — authentication, payload structure, and every event type
- **Trio SDK (Trio Checkout)** — embeddable payment initiation, PayIn, and PayOut flows
- **Guides and FAQ** — sandbox setup, first credential, first webhook, onboarding, and integration best practices

## 🚀 Quick Start

The docs are built with [Mintlify](https://mintlify.com). You need [Node.js](https://nodejs.org) v20.17.0 or later.

```bash
# Install the Mintlify CLI
npm install -g mint

# Start the development server from the repository root
mint dev
```

Open `http://localhost:3000` to preview the documentation.

## 📚 Documentation Structure

```
.
├── v1/                   # V1 documentation (published, default version)
│   ├── index.mdx         #   Welcome page
│   ├── getting_started/  #   Planning your integration, support channels
│   ├── developers/       #   Concepts, environments, authentication, errors, pagination, rate limits
│   ├── webhooks/         #   Webhook authentication, structure, and event reference
│   ├── guides/           #   Step-by-step integration guides (Pix, Boleto, cash-in, cash-out, onboarding)
│   ├── trio-sdk/         #   Trio Checkout SDK guides, flows, and a local example
│   ├── operations/       #   Best practices and reconciliation (closed loop)
│   ├── api-reference/    #   API reference generated from the OpenAPI specifications
│   └── faq/              #   Frequently asked questions (EN and PT-BR)
├── v2/                   # V2 documentation, same layout as v1
├── snippets/             # Reusable MDX snippets, shared by every version
├── images/               # Screenshots and diagrams, shared by every version
├── logo/                 # Brand logos for light and dark themes
├── llms.txt              # Curated index for AI agents
└── docs.json             # Mintlify configuration: navigation, versions, redirects, theme
```

Navigation is defined in `docs.json`. Every page must be listed there to appear in the sidebar.

## 🔧 Local Development

### Editing content

1. Create a branch from `main`.
2. Add or edit `.mdx` files. New pages need a `title` in the frontmatter.
3. Register new pages under the right version in the `navigation` section of `docs.json`.
4. Run `mint dev` and check the result in the browser.

### Working with the API reference

Endpoint pages under `api-reference/` are generated from the OpenAPI files in the same folder. Update the specification first, then regenerate or adjust the affected pages:

```bash
npx @mintlify/scraping@latest openapi-file api-reference/banking-api/openapi.json -o api-reference/banking-api
```

### Useful commands

```bash
mint dev            # Local preview with hot reload
mint broken-links   # Find broken internal links before opening a PR
mint update         # Upgrade the CLI to the latest version
```

### Troubleshooting

- **`mint dev` does not start** — run `mint update` and try again.
- **404 on every page** — make sure you are running the command from the directory that contains `docs.json`.
- **Page missing from the sidebar** — the page path is probably not listed in `docs.json`.
- **Unexpected rendering** — check for unclosed MDX components or invalid frontmatter.

## 🔀 Versions

The site is versioned with Mintlify [versions](https://mintlify.com/docs/organize/navigation#versions). Each version has its own navigation tree in `docs.json` and its own pages. Readers switch versions from a dropdown in the navbar.

| Version | Content location | Status |
| --- | --- | --- |
| `v1` | `v1/` | Published, default |
| `v2` | `v2/` | Selectable, being written |

`v2/` mirrors the `v1/` folder layout, so a page's V2 counterpart lives at the same relative path. Stub pages carry an MDX comment pointing at the `v1` page to start from.

Page URLs carry the version prefix, for example `/v1/guides/cash-in` and `/v2/guides/pix/overview`. The `redirects` section of `docs.json` forwards the pre-versioning URLs such as `/guides/cash-in` to their `/v1/` equivalents, so links published before the split keep working.

### Writing V2 content

1. Edit the stub under `v2/`, or add a new `.mdx` file next to it.
2. Register new pages in the `v2` version inside `docs.json`.
3. Link between V2 pages with root-relative paths that include the prefix, for example `/v2/developers/authentication`.
4. Preview with `mint dev` and pick the version from the dropdown in the navbar.

### V2 API reference

Add the V2 specification as `v2/api-reference/openapi.json`. Mintlify scans every file named `openapi*.json` and refuses one with no endpoints, so the file does not exist yet. A minimal header for the spec:

```json
{
  "openapi": "3.1.0",
  "info": { "title": "Banking API", "version": "2.0.0" },
  "servers": [
    { "url": "https://api.sandbox.trio.com.br", "description": "Sandbox" },
    { "url": "https://api.trio.com.br", "description": "Production" }
  ],
  "security": [{ "basic": [] }],
  "components": { "securitySchemes": { "basic": { "type": "http", "scheme": "basic" } } },
  "paths": {}
}
```

Then switch the `Banking API` group of the `v2` version in `docs.json` from the stub page to the `openapi` field. Mintlify generates every endpoint page and the sidebar groups from the spec's tags, and no endpoint pages need to be listed:

```json
{
  "group": "Banking API",
  "openapi": {
    "source": "v2/api-reference/openapi.json",
    "directory": "v2/api-reference"
  }
}
```

Hide an endpoint from the generated reference by adding `x-hidden: true` to its operation. If you prefer hand-curated endpoint pages, as `v1` does, generate them into `v2/api-reference/` and list them under the `v2` version instead:

```bash
npx @mintlify/scraping@latest openapi-file v2/api-reference/openapi.json -o v2/api-reference
```

### Publishing V2

1. To make V2 the default, move `"default": true` from `v1` to `v2` in `docs.json`.
2. Add V2 pages to `llms.txt`, or delete the file to fall back to the generated index.
3. To take a version offline without deleting it, set `"hidden": true` on it. Hidden versions are still rendered by `mint dev`.

## 🚢 Deployment

Deployment is handled by the Mintlify GitHub integration. Changes merged into `main` are published automatically to [docs.trio.com.br](https://docs.trio.com.br).

See the [Mintlify GitHub App documentation](https://mintlify.com/docs/settings/github) for details.

## 🤝 Contributing

Found a typo, a broken link, or an outdated example? Contributions are welcome.

1. Fork the repository and create a branch.
2. Make your changes and verify them with `mint dev` and `mint broken-links`.
3. Open a pull request describing what changed and why.

For questions about the API itself rather than the docs, use the support channels below.

## 💪 Support

- **Support portal:** [suporte.trio.com.br](https://suporte.trio.com.br)
- **Email:** [suporte@trio.com.br](mailto:suporte@trio.com.br)
- **Sandbox:** [app.sandbox.trio.com.br](https://app.sandbox.trio.com.br)
- **Documentation issues:** [GitHub Issues](https://github.com/trioHQ/docs/issues)

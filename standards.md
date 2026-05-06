# Standards & API Reference

> Project: Graphic Design Tool · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO 32000-2:2020 — PDF 2.0** — ISO standard for the Portable Document Format version 2.0; the universal document export format for graphic design; defines page geometry, colour spaces, embedded fonts, transparency, and layers; graphic design tools must produce ISO 32000-2 compliant PDFs for print production. URL: https://www.iso.org/standard/75839.html

- **ISO 15076-1:2010 — ICC Colour Profiles** — ISO standard for the ICC (International Color Consortium) colour profile format; governs device-independent colour management (sRGB, AdobeRGB, CMYK profiles); design tools must embed ICC profiles in exported images and PDFs for consistent colour reproduction across screens, print, and web. URL: https://www.iso.org/standard/54754.html

- **ISO/IEC 10918-1 — JPEG** — ISO/IEC standard for the JPEG image compression format; the primary raster image format for web-optimised design exports (photos, complex graphics); supported by all graphic design tools as an export format. URL: https://www.iso.org/standard/18902.html

- **ISO/IEC 15948 — PNG** — ISO/IEC standard for the Portable Network Graphics format; lossless raster format with transparency (alpha channel); essential for UI/UX design exports, icon design, and web graphics requiring transparency. URL: https://www.iso.org/standard/29581.html

- **ISO/IEC 27001:2022** — Information security management; governs access controls and data handling for cloud-based design platforms (Figma, Canva) that store confidential design assets, unreleased brand materials, and proprietary product designs. URL: https://www.iso.org/standard/82875.html

### W3C & IETF Standards

- **W3C SVG 2 — Scalable Vector Graphics** — W3C standard for two-dimensional vector graphics in XML; the native format for web-based vector design, icon creation, and UI component design; Figma, Sketch, and Affinity Designer export SVG 2 for web delivery; CSS animations and JavaScript interactivity can be embedded in SVG. URL: https://www.w3.org/TR/SVG2/

- **W3C WCAG 2.2 — Web Content Accessibility Guidelines** — Governs colour contrast requirements in graphic design: minimum 4.5:1 contrast ratio for normal text (AA), 3:1 for large text (AA), 7:1 for normal text (AAA); design tools (Figma, Canva) include accessibility checkers; ADA Title II compliance deadline for state/local government digital content is 2026. URL: https://www.w3.org/TR/WCAG22/

- **W3C CSS Color Level 5** — W3C specification for colour in CSS including LCH, LAB, and Display P3 colour spaces; design tools increasingly support wide-gamut colours (P3, Rec. 2020) for HDR display design. URL: https://www.w3.org/TR/css-color-5/

- **W3C WebP / AVIF** — Next-generation image formats for web-optimised graphic exports; Figma (WebP), Canva (WebP/AVIF), and Adobe Express support these formats for smaller file sizes with equivalent quality. URL: https://www.w3.org/TR/webp-format/

- **RFC 6749 — OAuth 2.0** — Authorization framework used by Figma, Canva, and Adobe Creative Cloud for third-party plugin/app authorisation and API access. URL: https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 7519 — JSON Web Token (JWT)** — Used in design platform API authentication tokens for REST API access and plugin authorization flows. URL: https://datatracker.ietf.org/doc/html/rfc7519

### Data Model & API Specifications

- **OpenAPI 3.1** — Figma publishes its REST API specification as OpenAPI (github.com/figma/rest-api-spec); enables SDK generation and programmatic design file access. URL: https://spec.openapis.org/oas/latest.html

- **Figma REST API** — REST/JSON API for programmatic access to Figma design files, components, variables, prototypes, and developer handoff data; base URL `https://api.figma.com/v1`; supports file content reading, image rendering, webhook events, and design token export. URL: https://developers.figma.com/docs/rest-api/

- **Figma Plugin API** — JavaScript/TypeScript API for building interactive plugin experiences inside the Figma editor; provides access to document nodes, styles, components, variables, and the selection/viewport. URL: https://www.figma.com/plugin-docs/

- **W3C Design Tokens Community Group (DTCG) Format** — Emerging standard for design token interchange (colours, typography, spacing, radius) in JSON format; Figma Variables supports DTCG-compatible export; enables design-to-code token pipelines (Style Dictionary, Theo). URL: https://www.w3.org/community/design-tokens/

- **OpenType / WOFF2** — Font format standards; OpenType (ISO/IEC 14496-22) is the universal desktop font format; WOFF2 (W3C, RFC 8081) is the optimised web font delivery format; design tools must handle both for accurate typography rendering and export. URL: https://www.w3.org/TR/WOFF2/

- **ICC Profile v4** — International Color Consortium profile format v4 (ICC.1:2022); embedded in JPEG, PNG, PDF, and TIFF exports for device-independent colour accuracy across print and digital media. URL: https://www.color.org/specification/ICC.1-2022-05.pdf

### Security & Authentication Standards

- **GDPR Article 32 — Security of Processing** — Cloud design platforms storing proprietary brand assets, unreleased product designs, and confidential marketing materials must implement appropriate access controls, encryption, and audit logging. URL: https://gdpr-info.eu/art-32-gdpr/

- **SOC 2 Type II** — Required enterprise compliance for cloud-based design platforms; Figma and Canva maintain SOC 2 Type II reports for enterprise procurement. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **SAML 2.0 / OIDC** — Required for enterprise SSO integration in design platforms; Figma and Canva support SAML 2.0 for corporate identity provider integration in enterprise/education tiers. URL: https://docs.oasis-open.org/security/saml/v2.0/

- **OWASP API Security Top 10 (2023)** — Governs REST API security for design platform APIs; API1 (Broken Object Level Authorization) is critical for ensuring users can only access design files they are authorised to view. URL: https://owasp.org/API-Security/

- **EU EAA — European Accessibility Act (June 2025 enforcement)** — Requires digital services marketed in the EU to meet WCAG 2.1 AA accessibility standards; graphic design tools used for creating digital products/services must support accessibility checking against EAA requirements from June 2025. URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32019L0882

### MCP Server Specifications

Adobe Creative Cloud has moved most aggressively into the MCP ecosystem among design platforms:

- **Adobe Creative Cloud MCP (Nine Connectors, 2026)** — Adobe opened Creative Cloud to developers with nine MCP connectors covering Photoshop, Premiere Pro, Express, and 50+ Adobe apps; enables AI orchestration of creative workflows; Anthropic (maker of Claude) became an Adobe integration partner. URL: https://developer.adobe.com/

- **Adobe Firefly API** — Adobe's AI image generation API; REST API for text-to-image, image editing (inpainting, outpainting), text effects, and generative fill; accessible as standalone API and integrated into Adobe Express; MCP-compatible via Adobe Creative Cloud connectors. URL: https://developer.adobe.com/firefly-services/docs/firefly-api/

- **Figma MCP Pattern** — Figma's REST API and Plugin API enable MCP server construction for design file access; community MCP servers allow AI agents to query design files, extract design tokens, generate code from components, and update properties programmatically.

---

## Similar Products — Developer Documentation & APIs

### Figma

- **Description:** Leading UI/UX and product design platform with real-time collaboration; REST API and Plugin API; FigJam whiteboard; Figma Sites, Make, and Buzz (announced Config 2025); OpenAPI spec published; updated rate limits and granular scopes (November 2025).
- **API Documentation:** https://developers.figma.com/docs/rest-api/
- **Plugin API Reference:** https://www.figma.com/plugin-docs/
- **OpenAPI Spec:** https://github.com/figma/rest-api-spec
- **SDKs/Libraries:** figma-api (TypeScript, community); @figma/rest-api-spec (OpenAPI); Figma Plugin SDK (TypeScript); figma-js (Node.js)
- **Developer Guide:** https://developers.figma.com/
- **Standards:** REST/JSON (OpenAPI 3.1), OAuth 2.0 (granular scopes), Plugin API (JavaScript/TypeScript), SVG export, PDF export, PNG/WebP/AVIF export, Design Tokens (DTCG)
- **Authentication:** OAuth 2.0 (authorization code flow with granular scopes); personal access tokens; plugin API key

### Canva

- **Description:** Mass-market graphic design platform with 240M+ users; Canva Apps SDK for building in-editor applications; Connect APIs for template, brand, and asset management; Canva AI 2.0 (April 2026) with fully editable AI outputs.
- **API Documentation:** https://www.canva.dev/docs/
- **Apps SDK:** https://github.com/canva-sdks/canva-apps-sdk-starter-kit
- **SDKs/Libraries:** canva-apps-sdk-starter-kit (TypeScript); Canva Connect API (REST/JSON)
- **Developer Guide:** https://www.canva.com/developers/
- **Standards:** REST/JSON (Connect API), OAuth 2.0, TypeScript SDK (Apps), SVG/PNG/PDF export, WCAG colour contrast checker
- **Authentication:** OAuth 2.0 (Connect API); App API key (in-editor apps)

### Adobe Creative Cloud (Firefly + Express + Photoshop API)

- **Description:** Adobe's suite including Photoshop, Illustrator, Premiere Pro, and Express; Firefly AI API for generative image creation; Creative Cloud APIs and nine MCP connectors enabling AI orchestration across 50+ Adobe apps; Anthropic integration partner (2026).
- **API Documentation:** https://developer.adobe.com/
- **Firefly API:** https://developer.adobe.com/firefly-services/docs/firefly-api/
- **Photoshop API:** https://developer.adobe.com/photoshop/photoshop-api-docs/
- **SDKs/Libraries:** Adobe I/O SDK; UXP (Unified Extensibility Platform) for Photoshop/Illustrator plugins; Firefly SDK; Creative SDK (legacy)
- **Developer Guide:** https://developer.adobe.com/creative-cloud/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0 (Adobe IMS), UXP plugins (JavaScript), ICC profiles, PDF/SVG export, CMYK colour management, MCP connectors
- **Authentication:** Adobe IMS OAuth 2.0; Service credentials for server-to-server

### Affinity Designer / Publisher / Photo (Serif)

- **Description:** One-time-purchase desktop creative suite (vector, raster, layout); relaunched as free all-in-one Affinity app by Canva (2026 announcement); no public REST API; file format is proprietary (.afdesign) with SVG/PDF/PNG export.
- **API Documentation:** No public REST API
- **SDKs/Libraries:** No official SDK; Affinity Publisher supports IDML (InDesign) import; SVG/PDF/EPS export
- **Developer Guide:** Not applicable
- **Standards:** SVG, PDF/X (print), CMYK ICC profiles, TIFF, JPEG, PNG, EPS export
- **Authentication:** Not applicable (desktop app; Canva account for cloud features)

### Inkscape (Open Source)

- **Description:** Open-source (GPL v2+) SVG vector graphics editor; the reference open-source design tool; command-line interface (inkscape CLI) for batch processing; Python extension API; SVG 1.1/2 native format; used in automated design generation pipelines.
- **API Documentation:** https://inkscape.org/develop/extensions/
- **CLI Reference:** https://inkscape.org/doc/inkscape-man.html
- **SDKs/Libraries:** Python extensions API; inkscape CLI (batch SVG processing); inkscape-python; lxml for SVG manipulation
- **Developer Guide:** https://inkscape.org/develop/
- **Standards:** W3C SVG 1.1/2 (native format), PDF/EPS export, PNG rasterisation, WCAG colour tools via extensions, GPL v2+ licence
- **Authentication:** Not applicable (local application)

### GIMP (Open Source)

- **Description:** Open-source (GPL v3+) raster image editor; Script-Fu (Scheme) and Python-Fu scripting APIs for batch processing automation; widely used in automated image processing pipelines; Flatpak and CLI available.
- **API Documentation:** https://www.gimp.org/docs/
- **Script-Fu/Python-Fu Reference:** https://www.gimp.org/tutorials/
- **SDKs/Libraries:** Python-Fu API; Script-Fu (Scheme); GIMP CLI (batch mode); gimp-python; libgimp (C API for plugins)
- **Developer Guide:** https://www.gimp.org/develop/
- **Standards:** XCF (native), PNG, JPEG, TIFF, WebP, PSD (Photoshop), ICC profiles, GPL v3+ licence
- **Authentication:** Not applicable (local application)

---

## Notes

- **Figma OpenAPI spec (github.com/figma/rest-api-spec)**: Figma publishes its full REST API specification as OpenAPI 3.1; this is notable among design platforms and enables type-safe SDK generation; updated rate limits and granular scopes (November 2025) are breaking changes requiring app re-publication.

- **Adobe Creative Cloud MCP (2026)**: Adobe's nine MCP connectors covering 50+ apps represent the most comprehensive design platform MCP integration; enables AI agents to orchestrate multi-app creative workflows (e.g., generate image with Firefly → composite in Photoshop → add to video in Premiere Pro).

- **Design Tokens DTCG format as the design-to-code standard**: The W3C Design Tokens Community Group (DTCG) JSON format is becoming the standard for exporting design tokens (colours, typography, spacing) from Figma Variables to code; Style Dictionary (MIT, Amazon) is the leading open-source transform pipeline.

- **EU EAA enforcement (June 2025)**: The European Accessibility Act requires WCAG 2.1 AA compliance for digital services in the EU from June 2025; graphic design tools creating UI/UX for European markets must include WCAG accessibility checking; ADA Title II deadline for US state/local government is 2026.

- **ICC colour profiles for print workflows**: Any graphic design tool targeting print production must support ICC colour profile embedding in PDF exports; the transition from ICC v2 to ICC v4 profiles (ICC.1:2022) is ongoing in professional print workflows.

- **Open-source landscape**: Inkscape (GPL v2+, SVG native) and GIMP (GPL v3+, raster) are the leading open-source design tools; Penpot (MPL 2.0) is the leading open-source Figma alternative for UI/UX design with collaboration features; all require self-hosting for team features.

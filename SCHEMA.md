# Wiki Schema

## Domain
Homelab, networking, WireGuard, OpenWrt, and self-hosted infrastructure.

## Conventions
- File names: lowercase, hyphens, no spaces (e.g., `wifi-6e.md`, `wireguard-guide.md`)
- Every wiki page starts with YAML frontmatter
- Use `[[wikilinks]]` to link between pages (minimum 2 outbound links per page)
- When updating a page, always bump the `updated` date
- Every new page must be added to `index.md` under the correct section
- Every action must be appended to `log.md`
- **Provenance markers:** On pages that synthesize 3+ sources, append `^[raw/articles/source-file.md]`
  at the end of paragraphs whose claims come from a specific source.

## Frontmatter
```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | guide | query | summary
tags: [from taxonomy below]
sources: [raw/articles/source-name.md]
# Optional quality signals:
confidence: high | medium | low
contested: true
contradictions: [other-page-slug]
---
```

### raw/ Frontmatter
```yaml
---
source_url: https://example.com/article
ingested: YYYY-MM-DD
sha256: <hex digest of the body>
---
```

## Tag Taxonomy
- **Standard:** wifi4, wifi5, wifi6, wifi6e, wifi7, 802.11, ieee
- **Protocol:** mac, phy, ofdm, ofdma, mu-mimo, beamforming, mlo, qam
- **Hardware:** ap, router, chipset, antenna, radio, client
- **Security:** wpa2, wpa3, encryption, auth, owf
- **Deployment:** mesh, enterprise, home, planning, survey, monitoring
- **Networking:** vpn, wireguard, openwrt, firewall, routing, nat
- **Homelab:** self-hosted, nas, server, docker, dns, dhcp
- **Technology:** iot, lora, bluetooth, zigbee, 5g, tri-band, embedded
- **Meta:** comparison, guide, glossary, faq, troubleshooting, reference, protocol, performance

## Page Thresholds
- **Create a page** when an entity/concept appears in 2+ sources OR is central to one source
- **DON'T create a page** for passing mentions
- **Split a page** when it exceeds ~200 lines
- **Archive a page** when fully superseded — move to `_archive/`, remove from index

## Entity Pages
One page per notable entity. Include: overview, key facts, relationships, sources.

## Concept Pages
One page per concept. Include: definition, current state, open questions, related concepts.

## Comparison Pages
Side-by-side analyses with table format preferred.

## Update Policy
When new information conflicts with existing content:
1. Check dates — newer sources generally supersede older ones
2. If contradictory, note both positions with dates and sources
3. Mark contradiction in frontmatter: `contradictions: [page-name]`
4. Flag for user review in lint report

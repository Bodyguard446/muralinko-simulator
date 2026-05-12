# muralinko-simulator — Session Log

Newest at top. Each entry: what was discussed, what shipped, what is next.

---

## 2026-05-11 (afternoon)
**Discussed:** Production deployment of muralinko-simulator + Wix CTA placement.
**Shipped:** Cloudflare Workers deploy at `simulateur.muralinko.workers.dev` (Git auto-deploy from `Bodyguard446/muralinko-simulator`, COOP/COEP headers for fast WASM bg removal); renamed Cloudflare workers subdomain `chokri-kefi` → `muralinko`; renamed worker `muralinko-simulator` → `simulateur`; empty default inputs + step gating in simulator; CTA button "Simulez votre projet en 30s →" added to muralinko.be hero (replaced "Découvrez comment ça marche"); 49/49 automated tests passing (printable calc, homography, recommendations, DOM pipeline, edge cases).
**Next:** Optional — custom subdomain `simulateur.muralinko.be` (requires DNS migration to Cloudflare); secondary CTAs on services + devis pages; click-tracking on simulator CTA.

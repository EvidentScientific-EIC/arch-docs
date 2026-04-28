# Atlas — FluoView / HPF / VPA Architecture & Documentation

Architecture diagrams, onboarding material, references, how-to guides, ADRs, and the canonical tech-debt register for the Olympus FluoView platform (`fv` + `h-pf` + `vpa`).

**Maintained by:** Bangalore Team
**Audience:** Engineers picking up FluoView from the Japan teams, including new Bangalore hires.

---

## Start Here

**Read [INDEX.md](INDEX.md) first.** It is the single navigation hub — every document in this repo is linked from it with a one-line description.

If you're new to the platform, follow this order:

1. [`onboarding/01_developer_guide.md`](onboarding/01_developer_guide.md) — product, architecture, key concepts
2. [`onboarding/02_environment_setup.md`](onboarding/02_environment_setup.md) — Windows / JDK 21 / Eclipse / network checklist
3. [`onboarding/03_build_runbook.md`](onboarding/03_build_runbook.md) — `h-pf → fv → vpa` build order, common errors
4. [`onboarding/04_faq.md`](onboarding/04_faq.md) — general / build / architecture / coding / SWT FAQs
5. [`diagrams/01_architecture.md`](diagrams/01_architecture.md) — 6-layer system overview
6. [`reference/01_domain_glossary.md`](reference/01_domain_glossary.md) — every term you'll encounter

For day-to-day work:

- **Adding code:** start in [`how-to/`](how-to/) — six task-oriented guides
- **Looking up an API:** [`reference/03_interface_service_catalogue.md`](reference/03_interface_service_catalogue.md)
- **Wondering about a known issue:** [`architecture/02_technical_debt_register.md`](architecture/02_technical_debt_register.md) — the canonical single source of tech-debt truth

---

## Repository Layout

```
atlas/
├── INDEX.md            ← navigation hub — start here
├── README.md           ← this file
├── diagrams/           ← 11 Mermaid diagrams
├── onboarding/         ← 4 onboarding files (read in order)
├── reference/          ← 5 deep-reference documents
├── how-to/             ← 6 task-oriented guides
├── architecture/       ← ADRs + canonical tech debt register
├── testing/            ← coverage map + test plan
└── quality/            ← static analysis + broader quality
```

**Total:** 34 documents across 8 folders.

---

## Conventions

- **Mermaid diagrams** — render in VS Code (Mermaid extension), GitHub, or [mermaid.live](https://mermaid.live)
- **ADRs are append-only** — never edit an accepted ADR; add a new superseding one instead
- **Tech-debt register** is the single canonical source — items have type, risk, and priority assigned
- **Update INDEX.md** whenever a new document is added to any folder
- **Preserve Japanese comments** in source code — add English alongside, don't overwrite

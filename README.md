# fintech-systems
A structured knowledge library on building systems in fintech, covering architectures, failure modes, trade-offs, and deep explorations of how financial systems actually work.
> For operational and leadership strategies, see [EM-Playbook](https://github.com/kratika-jain-em/em-playbook)

---

## About This Repository

Financial systems carry a different kind of weight. A bug isn't just a bug, it's a duplicate charge, a missed settlement, a compliance breach. The engineering decisions made in fintech have consequences that most software domains simply don't face.

This repository is a curated collection of technical writing on how these systems are designed, where they fail, and what the right design decisions look like under real constraints. 
I will try to cover most of the technical aspects of fintech engineering, from distributed ledger systems to payment processing internals like KYC, FX, Reconciliation etc, to common failures and trade-off strategies. 
The topics and the content is intended to grow over time

---

## About me

**Kratika Jain**
I am an Engineering leader with 15+ years of experience designing and building systems across global fintech ecosystems.

[LinkedIn](https://www.linkedin.com/in/kratika-jain-em/) | [Medium](https://medium.com/@kratikasays)

---

## Structure

```
fintech-systems/
├── architectures/       # System design patterns specific to fintech
├── failures/            # Failure modes, root causes, and lessons
├── trade-offs/          # Design decisions with no universal answer
└── deep-dives/          # End-to-end explorations of financial systems
```

---

## Topics

### Architectures
Core system design patterns where fintech diverges from general software engineering. Covers payment processing, ledger design, event streaming, reconciliation, fraud and risk systems, FX engines, KYC pipelines, card issuance, open banking integrations, and financial data infrastructure.

### Failures
The failure modes that define fintech engineering: duplicate payments, race conditions in balance updates, ledger drift, cascade failures across payment rails, silent data corruption, compliance gaps, and operational mistakes at scale. Understanding these is table stakes for senior engineers in the domain.

### Trade-offs
Design decisions with real financial and operational consequences: consistency vs availability in payment systems, synchronous vs async flows, build vs buy, event sourcing vs CRUD, data residency, and operating in regulated environments where the cost of being wrong is asymmetric.

### Deep Dives
End-to-end explorations of how financial systems actually work: card network mechanics, SWIFT and correspondent banking, Faster Payments, SEPA, ACH, ISO 20022, FX markets, and engineering case studies from Stripe, Wise, Monzo, and others.

---

## Status

This is a living repository. Documents are added incrementally. Contributions, corrections, and discussion are welcome via issues.

---

## Licence

All content is original and © Kratika Jain. You are welcome to reference and link to any document here. Please do not reproduce content wholesale without attribution.

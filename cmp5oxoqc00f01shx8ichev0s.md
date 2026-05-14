---
title: "What I look for while evaluating a payment system!"
seoTitle: "Evaluate technical health of a payment system"
datePublished: 2026-05-14T16:16:21.898Z
cuid: cmp5oxoqc00f01shx8ichev0s
slug: tdd-payment-system
cover: https://cdn.hashnode.com/uploads/covers/69e52c6413e74eec5844b8cb/f1aba593-cf38-448b-ae6c-7a2b4db22732.png
ogImage: https://cdn.hashnode.com/uploads/og-images/69e52c6413e74eec5844b8cb/5713a494-89fb-43e2-ac41-d7a7a951c405.png
tags: payment-systems, fintech-startups, fintech-software-development, technical-due-diligence

---

For me, evaluating a payment system rarely starts with the architecture design. Not because architecture is unimportant, but most systems look reasonable in a diagram and the interesting information is hardly ever there.

The systems that are worrisome can appear stable externally, but internally are dependent on undocumented workarounds, where the team is afraid of deployments, and there exists tribal knowledge that never made it to runbooks.

None of this shows up in technical presentation, but there is way to find them out.

![](https://cdn.hashnode.com/uploads/covers/69e52c6413e74eec5844b8cb/5f31c56e-54be-4b23-829a-5b2fbaf8b9d7.png align="center")

* * *

### Start with incidents, not architecture

Every payment system that works during a happy-path demo tells very little of its health. The real signal is how it behaves during degraded states - outages, reconciliation breaks, settlement delays, failed deployments, etc.

One of the first thing I'd ask is,

> "Walk me through the last serious production incident."

Not to assign blame, but to understand how quickly the issue was detected, ownership defined, whether people trusted the monitoring , how did the communication worked internally, and what changed afterwards.

Strong team answers operational questions specifically, because incidents in fintech leave marks - reconciliation exercises, settlement exposure, regulatory talks or merchant impacted. Weak teams answer with abstractions - *system is resilient*, *we have kubernates*.

> **If you are doing TDD on a payment system:** ask to see the incident log before you ask to see the architecture. What you hear in the first few minutes of that conversation will tell you more than any diagram or presentation.

* * *

### Deployment strategy reveals operational resiliency

Another thing I would pay close attention to during a TDD is *deployment behaviour*. This metrics is not about deployment frequency, rather whether the team had strong discipline around observability, rollback, reconciliation safety, change management, and migration management.

Database migrations are especially revealing. Payment systems cannot tolerate long write locks, inconsistent state transitions, reconciliation drift, or partial financial updates during settlement windows. So I ask: *how are schema changes handled? Does rollback exist? How is migration risk evaluated? Who approves the migration before it runs?*

> **If you are doing TDD:** ask to walk you through the last database migration on a financial data-critical table. The answer will tell you about their risk awareness, their tooling, and whether they have ever felt the consequences of getting it wrong.

* * *

### Dependency concentration that accumulates quietly

Modern payment stacks heavily depend on external integrations, like PSPs, sponsor banks, KYC /AML vendors, card processors, and enrichment tools like fraud, reporting, analytics etc. With increasing use and integration of AI-tools, add that to the list as well.

Two questions to get the answer as early as possible -

*   *Which dependency could hurt the organisation or system most if it failed tomorrow?*
    
*   *How difficult would it be to replace that integration under operational pressure?*
    

Mature organisations can answer both clearly.

> **If you are doing TDD:** draw the dependency map yourself based on what they tell you, and then walk through it with them collaboratively to see if something is missing.

* * *

### Financial truth is harder to maintain than we care to admit

This is one of the fastest ways to understand whether a system was designed with operational reality in mind. While evaluating a ledger or transaction system, it is crucial to understand how balances are derived, whether historical reconstruction is possible, how reconciliation works, whether transaction history is immutable, and how corrections are audited.

Two questions to almost always ask -

*   *Can the account-balance be reconstructed at any given point of time, without touching the implementations?*
    
*   *What is operational-strategy in case reconciliation breaks?*
    

During outages, multiple systems can disagree simultaneously, PSP acknowledgements, internal ledger state, settlement files, bank confirmations etc. At that point the real question is: *which system currently owns truth?* Mature organisations have a clear answer to that before incidents happen.

> **If you are doing TDD:** ask the reconciliation break question directly. Ask them to show you a break that happened in production, how it was detected, what the resolution looked like, whether the audit trail survived.

* * *

### Organisational fragility equals operational fragility

A system where only one or two engineers hold the most of the operational knowledge is not just an engineering problem, but a business continuity risk.

Operational maturity is how knowledge is distributed through the team. Look at who people depend on operationally, whether the systems are documented realistically, how onboarding works for new engineers, and how much engineering time is spent into unplanned work.

Pay attention to following phrases -

*   "t*hat service always behaves strangely after deploys"*
    
*   *"finance usually fixes that manually"*
    
*   *"we rerun settlements if it happens"*
    
*   *"only one person fully understands that flow"*
    

Each statement sounds harmless in isolation, but together they describe an organisation that is managing fragility rather than resolving it. Sprint composition tells the same story differently. If engineering capacity is continuously consumed by incidents, manual corrections, operational support, deployment recovery, and reconciliation work, it is instability rather than scaling.

> **If you are doing TDD:** ask what percentage of last quarter's engineering work was unplanned. Who you would call at 2am if the settlement job failed.

* * *

### Questions to drive the conversation

There are certain questions that move a TDD discussion away from intended architecture to operational reality easily.

![](https://cdn.hashnode.com/uploads/covers/69e52c6413e74eec5844b8cb/fc7cb3c4-bdf4-44ee-aa0e-48eed10afe92.png align="center")

* * *

### My final thoughts

TDD in fintech is rarely about if the system is modern, or use cutting edge tech stack; but to understand whether the organisation can safely operate or recover under stress.

This usually requires looking beyond the architecture slides, technology choices, deployment conduct, etc.

* * *
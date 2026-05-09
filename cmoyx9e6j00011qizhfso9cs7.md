---
title: "Do you need a PSD2 licence?"
datePublished: 2026-05-09T22:35:01.779Z
cuid: cmoyx9e6j00011qizhfso9cs7
slug: do-you-need-a-psd2-licence
cover: https://cdn.hashnode.com/uploads/covers/69e52c6413e74eec5844b8cb/4ec3f8cd-1345-4f3b-9b8f-e242a5c448e3.png
ogImage: https://cdn.hashnode.com/uploads/og-images/69e52c6413e74eec5844b8cb/d9c3713f-69a4-452e-acaa-951def980b32.png

---

One of the easiest mistake by a payments/fintech startup is to start with the product flow and leave the regulatory model for later. This may sound practical at the beginning to build the app/platform, integrate with bank, & sort the compliance when traction appears. In reality, the order can become expensive very quickly.

> PSD2 is not just a regulatory document, it shapes what the system is allowed to do, who can touch funds, who can access account data, who authenticates the customer, who owns consent, who keeps evidence, & and who is liable if (or when) something goes wrong.

* * *

> **NOTE: This article is a practical engineering overview, not legal advice. PSD2 remains the current EU framework. PSD3 and PSR reached political agreement in November 2025 but are not yet in force. For actual authorisation decisions, firms should work with counsel and the relevant national competent authority.**

## Core distinction

***"We help users pay merchants." "We connect to bank accounts." "We aggregate balances." "We route payments." "We help businesses collect money." "We provide payment infrastructure."***

All these are product-language, and not regulatory activities. The main question is whether the product is doing, or causing another party to do any of the following:

*   Accessing payment account information
    
*   Initiating payments from a user's account
    
*   Executing credit transfers or direct debits
    
*   Issuing payment instruments
    
*   Acquiring payment transactions
    
*   Transmitting money
    
*   Issuing or managing electronic money
    
*   Holding or safeguarding customer funds
    

The closer your product gets to account access, payment initiation, fund movement, or stored value, the more likely you are entering regulated territory.

* * *

## Three common models of startup

### Technical service provider

In this model, the system provides the technology to a regulated entity, but do not provide the regulated payment service itself. Examples like **fraud tooling, payment analytics, workflow automation, merchant reporting tools**, etc.

To avoid this system to cross into regulated activities, following points are to consider -

*   *Do we ever enter into possession or control of funds?*
    
*   *Do we initiate the payment instruction ourselves?*
    
*   *Do we access user's account data?*
    
*   *Are we customer-facing as the provider of the payment service?*
    
*   *Are we makign decisions effecting the payment execution?*
    

From engineering view, such systems are designed as a support layer, not regulated execution layer.

### Operating under a licensed partner

It's a common model for startups to launch fast. Instead of becoming authorized directly, the startup/organisation partner with a licensed payment institution, e-money institution, bank, or, a regulated provider. The product operates on top of their licence.

It can reduce the 'time-to-market', but does not remove engineering responsibility, only splits them between partner and product.

Licensed partner usually cares about -

*   how the onboarding works
    
*   how customers are authenticated
    
*   what data the product collects
    
*   what funds flow through the system
    
*   how transactions are monitored
    
*   how incidents are reported
    
*   what audit evidence you can produce
    
*   how quickly they can shut down or control risky activity
    

Even if you don't own the licence, **engineering implication** is building the system to satisfy the licensed partner's compliance and operational requirements, which typically includes -

*   partner approval workflows
    
*   role-based access controls
    
*   event logs and audit trails
    
*   operational dashboards
    
*   transaction monitoring feeds
    
*   reconciliation support
    
*   incident escalation hooks
    
*   customer-data controls
    

### Becoming directly authorised or registered

This is the heavier route, you become the regulated entity yourself: a payment institution, e-money institution, account information service provider, or payment initiation service provider.

This model gives more control, but also more responsibilities. You should be able to demonstrate that business can safely provide the payment service, not just as a legal exercise but an engineering evidence exercise as well.

This includes following -

*   secure system architecture
    
*   customer authentication controls
    
*   incident management process
    
*   operational resilience
    
*   data protection controls
    
*   outsourcing controls
    
*   safeguarding arrangements where relevant
    
*   financial crime and fraud controls
    
*   governance and auditability
    

* * *

## Registering as PI, EMI, AISP or PISP: The practical difference

You have decided to go with model 3 and register yourself directly as a regulated entity. The exact authorisation or registration model depends on what your product does.

### Payment institution (PI)

A Payment Institution provides payment services such as executing payment transactions, money remittance, payment initiation, or acquiring, depending on the permissions granted.

This may be relevant if your product is directly involved in moving money but does not issue e-money or take deposits like a bank.

Engineering focus areas typically include:

*   transaction lifecycle management
    
*   reconciliation
    
*   safeguarding support where applicable
    
*   fraud monitoring
    
*   customer authentication
    
*   incident evidence
    
*   operational controls
    

### Electronic money institution (EMI)

An Electronic Money Institution issues electronic money: stored monetary value representing a claim on the issuer. This may be relevant if your product involves customer balances, wallets, stored value, prepaid-style accounts, or funds held for future payment use.

Engineering focus areas typically include:

*   wallet ledger design
    
*   balance integrity
    
*   safeguarding evidence
    
*   redemption flows
    
*   transaction history
    
*   reconciliation
    
*   access controls
    
*   customer-money reporting
    

If the product has anything that looks like a customer balance, stored value, or wallet, you should examine the EMI question very early.

### Account information service provider (AISP)

An Account Information Service Provider accesses account information from payment accounts, with the customer's permission.

This may be relevant if the product reads balances, transaction history, income data, or account details from banks to provide insights, dashboards, affordability checks, or aggregation.

Engineering focus areas typically include:

*   consent lifecycle
    
*   account-data retrieval
    
*   data minimisation
    
*   refresh logic
    
*   secure storage
    
*   access audit trails
    
*   API error handling
    
*   customer revocation flows
    

AISP doesn't move the money, but the data is sensitive, and the consent and security model matters.

### Payment initiation service provider (PISP)

A Payment Initiation Service Provider initiates a payment from the customer's account held at an ASPSP.

This may be relevant if your product allows a user to pay a merchant, move money, or trigger a bank transfer without using card rails.

Engineering focus areas typically include:

*   payment initiation flow
    
*   customer authentication handoff
    
*   payment status tracking
    
*   failure handling
    
*   redirect or decoupled flow support
    
*   reconciliation touchpoints
    
*   fraud and risk controls
    
*   customer messaging
    

PIS is operationally more sensitive than AIS because a payment instruction is being initiated. That makes status handling, traceability, and failure handling much more important.

Many early-stage products start as PISP-style flows and later evolve into PI-style systems as they take on more control over money movement.

* * *

## Build vs Partner decision

Before you decide which licence to get for, the question to ask yourself is:

***"Under which regulatory model can we build, launch, learn, and control the right things at the right time?"***

A direct licence gives control, but takes time and operational maturity. A partner model gives speed, but reduces control and introduces dependency. A technical-service-provider model can work well, but only if the regulated boundary is genuinely outside your product.

The right answer depends on the product, the funding stage, the market, and how close the product is to regulated activity.

* * *

## Common mistakes to avoid

1.  **Assuming "we do not hold funds" means PSD2 does not matter:** Not holding funds may reduce some obligations, but PSD2 also covers account information and payment initiation services. A product can be in scope without holding customer money.
    
2.  **Treating the licence as a legal wrapper:** The licence model is not just a document to attach to the product, it changes what the system must prove, log, secure, and control.
    
3.  **Choosing a partner without understanding technical dependency:** A licensed partner can accelerate launch, but it can also define your product constraints. Their APIs, controls, approval process, incident requirements, and reporting expectations become part of your architecture.
    
4.  **Designing the MVP before defining the regulated boundary:** A payment MVP is not just a thin product experiment. If it touches payment initiation, funds, account data, or stored value, the MVP already contains regulated-system decisions, without boundary-settings.
    
5.  **Assuming PSD3/PSR is a reason to wait:** PSD2 is still the current framework. PSD3/PSR matters for future readiness, but startups building now still need to understand PSD2 first.
    

* * *

## Before you lock the architecture

These five things should happen before serious architecture decisions are made.

1.  **Draw the money flow:** Show where and how funds move, who controls them, where they settle, and whether your system ever has possession or control of them.
    
2.  **Draw the data flow:** Show which account data your system access, where it comes from, where and how is it stored, who can see/access it, and how the consent is captured and revoked.
    
3.  **Draw the control flow:** Show who initiates the payments, who authenticates the customer, who can block the transactions, who monitors fraud, and who handles failure or refunds.
    
4.  **Map your regulatory role:** Identify whether you are acting as a technical service provider, partner of a licensed firm, AISP/PISP/EMI. If you're not sure, this lack of clarity is itself a finding.
    
5.  **Decide what you own now vs later:** Be explicit about which regulated capabilities are owned internally and/or by a partner, what are deferred for now, and what capabilities are not applicable. A partner model to permanently outsource the ledger, SCA etc is a different long-term position than the one that defers them temporarily.
    

* * *
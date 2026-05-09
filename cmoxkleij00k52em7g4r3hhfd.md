---
title: "PSD2, PSD3 & PSR: Starting point"
seoTitle: "psd2-psd3-psr-base"
datePublished: 2026-05-08T23:52:40.893Z
cuid: cmoxkleij00k52em7g4r3hhfd
slug: primer
cover: https://cdn.hashnode.com/uploads/covers/69e52c6413e74eec5844b8cb/76d75baf-ea75-4d3e-8bb0-15f85bb5584d.png
ogImage: https://cdn.hashnode.com/uploads/og-images/69e52c6413e74eec5844b8cb/6326c054-c6a8-429a-9217-4293891fc015.png

---

Working in fintech for more than a decade made me realize, system architecture is not the hardest part for building a payments infrastructure, but understanding regulatory changes, messy terminology and turning a legal change into engineering work are the real challenges.

This series is focusing on PSD2, PSD3 and PSR; and this primer is a starting point for engineers working around open banking, payment initiations, regulated APIs, and payment systems in EU/UK.

* * *

## The short version

The simplest way to think about the landscape is this:

*   PSD2: Current regulatory framework (in force since 2018)
    
*   PSD3: Next directive, mainly focused on licensing, supervision, and the institutional structure (agreed Nov 2025)
    
*   PSR: Sits alongside PSD3 for day-to-day conduct around access, authentication, & user-protection rules.
    

This split is one of the first things engineers need to understand.

* * *

## The detailed landscape: what exists and what’s coming

### PSD2: The second payment services directive (2018)

It is the current law, a revised and extended verion for PSD1 with following two major additions:

1.  **Open Banking:** PSD2 enabled licensed third parties to access payment-related information or initiate a payment, subject to customer’s permissions and regulatory controls.
    
2.  **Strong Customer Authentication (SCA):** PSD2 introduced mandatory multi-factor authentication for electronic payments above certain thresholds, aiming to reduce fraud in card-not-present and online payment contexts.
    

PSD2 is a `directive`, meaning each member state transposed it into national law separately, some strictly and some loosely. Post Brexit, UK now operates under its own version (UK PSD2), diverging from EU framework overtime. This inconsistency is one of the primary problems PSD3 + PSR aims to fix.

### PSD3 & PSR: A combined package

PSD3 in itself is not a replacement of PSD2, it is narrower in space. It primarily covers licensing and prudential framework for payments and e-money institutions, reauthorisation obligations for existing licensed entities, supervisory arrangements between member states, & integration of EMIs into PI category.

PSR is more significant in a way that it is a `regulation`, and not a directive; which means it is applied directly and uniformly across all member states. It governs transparency obligations around fees, exchange rates etc, strong customer authentication, standardized API requirements, consent dashboards under open banking, & operational and security requirements for PSPs.

* * *

## The participation map that matters

Understanding who each participant is, determines which regulatory obligations apply to a given system; because not every player is ‘just a payments company’.

*   **ASPSP (Account Servicing Payment Service Provider):** the institution holding the customer’s payment account. Banks, building societies, EMIs. Must provide dedicated APIs to licensed TPPs with customer consent.
    
*   **AISP (Account Information Service Provider):** licensed entity that reads account data from one or more ASPSPs with customer consent. Read-only, and cannot initiate payments.
    
*   **PISP (Payment Initiation Service Provider):** licensed entity that initiates a payment from a customer’s account at an ASPSP. Does not hold funds, only instructs the ASPSP to execute.
    
*   **CBPII (Card-Based Payment Instrument Issuer):** issues a payment card linked to an account at an ASPSP. Can check available funds before authorising a transaction.
    
*   **TPP (Third Party Provider):** umbrella term for AISP, PISP, and CBPII. Engineering obligations include NCA registration, SCA implementation, eIDAS certificate management, and consent lifecycle management.
    
*   **PI (Payment Institution):** non-bank entity licensed to provide payment services. Can hold customer funds for payment purposes.
    
*   **EMI (Electronic Money Institution):** licensed to issue electronic money. Under PSD3, becomes a sub-category of PI, so existing EMI licences might require re-application during the transition period.
    

* * *

## The other frameworks that cannot be ignored

PSD2/3/PSR are only a part of a ‘payment ecosystem’, which usually sits at the intersection of following frameworks -

*   PSD2/3/PSR: Payment service rules
    
*   DORA: Digital operational resilience act
    
*   GDPR: General data protection regulation
    
*   eIDAS: Electronic identification, authentication and trust service regulation
    

* * *

## Why this series

A technical series for people trying to understand what these frameworks means, in their current and expected legislature. This article is a starting points, that aims to extend into -

*   Understanding PSD2 before building
    
*   Building PSD2-compliant systems
    
*   Prepare for PSD3/PSR migration
    

* * *
# Acquiring Bank Integration - Authorisation, Clearing & Settlement Flow

## Context
 
When a customer pays by card, the visible transaction, a tap, a click, a confirmation, is the surface of a multi-party financial flow that runs across acquirers, card schemes, and issuers over hours or days. Most payment systems interact with a PSP abstraction and never need to understand what is underneath. But when the business itself is the PSP, where the system holds a direct acquiring relationship, or while debugging why funds have not arrived, we need to understand the full lifecycle.


> The acquiring bank is the financial institution that processes card payments on behalf of a merchant. It sits between the merchant (or payment facilitator) and the card schemes, Visa, Mastercard, Amex, and ultimately the issuing bank that holds the cardholder's account. The acquiring relationship defines the commercial terms, settlement currency, the chargeback liability, and operational constraints.
 
---

## The Three-Phase Flow

Card payment processing is not a single operation. It is three distinct phases - authorisation, clearing, and settlement, each with different timing, participants, and failure modes.

```
Cardholder        Merchant/Gateway       Acquirer          Card Scheme         Issuer
    |                    |                   |                   |                 |
    |--- card tap ------>|                   |                   |                 |
    |                    |--- auth request ->|                   |                 |
    |                    |                   |--- ISO 8583 msg ->|                 |
    |                    |                   |                   |--- auth req --->|
    |                    |                   |                   |<-- approved ----|
    |                    |                   |<-- approved ------|                 |
    |                    |<-- approved ------|                   |                 |
    |<-- receipt --------|                   |                   |                 |
    |                    |                   |                   |                 |
    |             [ hours to days later — clearing ]             |                 |
    |                    |                   |                   |                 |
    |                    |--- capture msg -->|                   |                 |
    |                    |                   |---clearing file ->|                 |
    |                    |                   |                   |-- presentment ->|
    |                    |                   |                   |                 |
    |               [ settlement — T+1 or T+2 ]                  |                 |
    |                    |                   |                   |                 |
    |                    |                   |<-- net funds -----|                 |
    |                    |<-- settlement ----|                   |                 |
```

### Phase 1 — Authorisation
Authorisation is a real-time approval request. In this phase, merchants talks to the issuer via acquirer - "Is the card valid?", "does the cradholder have sufficient balance?, and/or "will this transaction be settled"?. 

> ***NOTE*** Authorisation doesn't mean "actual movement" of money, the system only places a hold on the required funds. 

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

#### ISO 8583: Card authorisations requests (MTI x1xx)

ISO 8583 is a globally accepted standard for card-originated financial transaction interchange messages between systems. These messages can be request authorisation, cash withdrawl, fund clearing/settlements etc. Typical data elements (DE) included in the authorisation requests are primary account number, processing code (purchase, withdrawl, refund etc), transaction amount, merchant id, response code, currency code etc. 

---

### Phase 2 - Clearing
This phase comes after successful authorisation, and a process of submitting the transaction data to the card scheme for final settlement. It has two approaches - 

**Single-message system** uses a single message to authorise funds and instantly settle them. 
**Dual-message system** separates authorisation and actual clearing/settlement (money movement). 

Clearing data is usually batched into files and submitted to the acquirer, typically once or twice per day within scheme-defined cut-off windows. Missing a clearing cut-off delays settlement by a full business day. 
 
```java
// Clearing record structure (simplified)
public record ClearingRecord(
    String retrievalReferenceNumber,   // must match auth RRN (DE37)
    String authorisationCode,          // from auth response (DE38)
    String merchantId,
    String terminalId,
    BigDecimal transactionAmount,
    BigDecimal captureAmount,          // may differ from auth amount (within tolerance)
    Currency currency,
    LocalDate transactionDate,
    LocalDate settlementDate,
    ProcessingCode processingCode,
    String systemsTraceAuditNumber
) {}
```

### Phase 3 - Settlement
Settlement is the actual movement of funds. The card scheme nets all transactions across issuers and acquirers and transfers the net position. The acquirer then disburses to the merchant, minus interchange fees and scheme fees.
 
Settlement timing is defined in the acquiring agreement, commonly T+1 or T+2 business days from the clearing date. Some schemes operate same-day settlement for qualifying merchants.
 
The net settlement amount differs from the gross transaction amount:
 
```
Gross transaction amount
  - Interchange fee          (paid to the issuer, defined by scheme)
  - Scheme fee               (paid to Visa/Mastercard)
  - Acquirer processing fee  (per your commercial agreement)
  ─────────────────────────
  = Net settlement amount received by merchant
```
 
Interchange rates vary by card type (consumer debit, commercial credit, premium rewards), entry mode (chip, contactless, CNP), and merchant category code (MCC). This is why accepting a corporate Amex card costs significantly more than a standard Visa debit.
 
---

## Implementation

### Authorisation request handler

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class AuthorisationService {
 
    private final AcquirerGatewayClient acquirerClient;
    private final PaymentRepository paymentRepository;
    private final IdempotencyKeyRepository idempotencyKeyRepository;
 
    @Transactional
    public AuthorisationResponse authorise(AuthorisationRequest request) {
 
        // Generate acquirer-facing references
        String stan = generateStan();           // 6-digit, unique per session
        String rrn  = generateRrn();            // 12-digit, unique per transaction
 
        Iso8583Message authMessage = Iso8583Message.builder()
                .mti("0100")                                    // authorisation request
                .pan(request.getEncryptedPan())
                .processingCode(ProcessingCode.PURCHASE)
                .amount(request.getAmount())
                .transmissionDateTime(Instant.now())
                .stan(stan)
                .posEntryMode(request.getEntryMode())
                .retrievalReferenceNumber(rrn)
                .terminalId(request.getTerminalId())
                .merchantId(request.getMerchantId())
                .currencyCode(request.getCurrency())
                .build();
 
        log.info("Sending authorisation request: stan={}, rrn={}, amount={}",
                stan, rrn, request.getAmount());
 
        AcquirerResponse response = acquirerClient.send(authMessage);
 
        AuthorisationResult result = AuthorisationResult.builder()
                .rrn(rrn)
                .stan(stan)
                .approvalCode(response.getApprovalCode())       // DE38
                .responseCode(response.getResponseCode())       // DE39
                .authorised(ResponseCode.APPROVED.equals(response.getResponseCode()))
                .authorisedAmount(request.getAmount())
                .timestamp(Instant.now())
                .build();
 
        // Persist before returning — idempotency and audit
        paymentRepository.saveAuthorisation(request, result);
 
        log.info("Authorisation result: rrn={}, responseCode={}, approved={}",
                rrn, response.getResponseCode(), result.isAuthorised());
 
        return AuthorisationResponse.from(result);
    }
 
    private String generateStan() {
        // STAN must be numeric, max 6 digits, unique within a processing day per terminal
        return String.format("%06d",
            ThreadLocalRandom.current().nextInt(1, 999_999));
    }
 
    private String generateRrn() {
        // RRN: 12 alphanumeric characters, must be unique and echoed in settlement
        return Instant.now().toString().replaceAll("[^0-9]", "").substring(0, 12);
    }
}
```

### Response Code Handling
 
Not all declines are handled the same way. Scheme rules define which response codes may be retried, which must be surfaced to the cardholder, and which carry retry restrictions.
 
```java
public enum ResponseCodeBehaviour {
    APPROVED,
    SOFT_DECLINE_RETRYABLE,     // retry permitted after delay (e.g. 51 insufficient funds)
    SOFT_DECLINE_NON_RETRYABLE, // retry permitted only after cardholder action (e.g. 59 suspected fraud)
    HARD_DECLINE,               // must not retry (e.g. 04 pick up card, 41 lost, 43 stolen)
    REFERRAL                    // call issuer for voice authorisation (rare in modern flows)
}
 
@Component
public class ResponseCodeClassifier {
 
    private static final Map<String, ResponseCodeBehaviour> BEHAVIOURS = Map.ofEntries(
        Map.entry("00", ResponseCodeBehaviour.APPROVED),
        Map.entry("51", ResponseCodeBehaviour.SOFT_DECLINE_RETRYABLE),     // insufficient funds
        Map.entry("61", ResponseCodeBehaviour.SOFT_DECLINE_RETRYABLE),     // exceeds withdrawal limit
        Map.entry("65", ResponseCodeBehaviour.SOFT_DECLINE_RETRYABLE),     // exceeds frequency limit
        Map.entry("59", ResponseCodeBehaviour.SOFT_DECLINE_NON_RETRYABLE), // suspected fraud
        Map.entry("41", ResponseCodeBehaviour.HARD_DECLINE),               // lost card
        Map.entry("43", ResponseCodeBehaviour.HARD_DECLINE),               // stolen card
        Map.entry("04", ResponseCodeBehaviour.HARD_DECLINE),               // pick up card
        Map.entry("14", ResponseCodeBehaviour.HARD_DECLINE),               // invalid card number
        Map.entry("54", ResponseCodeBehaviour.HARD_DECLINE),               // expired card
        Map.entry("01", ResponseCodeBehaviour.REFERRAL),
        Map.entry("02", ResponseCodeBehaviour.REFERRAL)
    );
 
    public ResponseCodeBehaviour classify(String responseCode) {
        return BEHAVIOURS.getOrDefault(responseCode, ResponseCodeBehaviour.HARD_DECLINE);
    }
 
    public boolean isSafeToRetry(String responseCode) {
        return classify(responseCode) == ResponseCodeBehaviour.SOFT_DECLINE_RETRYABLE;
    }
}
```

> **Scheme retry rules matter here.** Visa and Mastercard both enforce limits on how many times a declined transaction may be retried within a given window. Excessive retries against hard-decline response codes generate scheme fines, up to USD 25 per violation above threshold.

### Capture (Clearing Submission)
 
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class CaptureService {
 
    private final AcquirerGatewayClient acquirerClient;
    private final PaymentRepository paymentRepository;
 
    @Transactional
    public CaptureResult capture(CaptureRequest request) {
        AuthorisationRecord auth = paymentRepository
                .findAuthorisationByRrn(request.getRrn())
                .orElseThrow(() -> new AuthorisationNotFoundException(request.getRrn()));
 
        validateCaptureAmount(auth, request.getCaptureAmount());
 
        Iso8583Message captureMessage = Iso8583Message.builder()
                .mti("0200")                                       // financial transaction request
                .processingCode(ProcessingCode.PURCHASE)
                .amount(request.getCaptureAmount())
                .stan(generateStan())
                .retrievalReferenceNumber(auth.getRrn())           // must match original auth RRN
                .authorisationCode(auth.getApprovalCode())         // must match original approval code
                .terminalId(auth.getTerminalId())
                .merchantId(auth.getMerchantId())
                .currencyCode(auth.getCurrency())
                .build();
 
        AcquirerResponse response = acquirerClient.send(captureMessage);
 
        CaptureResult result = CaptureResult.builder()
                .rrn(auth.getRrn())
                .captureAmount(request.getCaptureAmount())
                .responseCode(response.getResponseCode())
                .captured(ResponseCode.APPROVED.equals(response.getResponseCode()))
                .capturedAt(Instant.now())
                .build();
 
        paymentRepository.saveCapture(auth, result);
 
        log.info("Capture result: rrn={}, captureAmount={}, captured={}",
                auth.getRrn(), request.getCaptureAmount(), result.isCaptured());
 
        return result;
    }
 
    private void validateCaptureAmount(AuthorisationRecord auth, BigDecimal captureAmount) {
        // Scheme tolerance: capture may not exceed auth amount by more than the allowed percentage
        BigDecimal maxCapture = auth.getAuthorisedAmount()
                .multiply(BigDecimal.valueOf(1.15)); // 15% tolerance for hospitality
 
        if (captureAmount.compareTo(maxCapture) > 0) {
            throw new CaptureLimitExceededException(
                "Capture amount %s exceeds tolerance for authorised amount %s"
                    .formatted(captureAmount, auth.getAuthorisedAmount())
            );
        }
    }
}
```

### Settlement Reconciliation
 
Settlement files from the acquirer must be reconciled against your internal capture records. Discrepancies indicate either a processing error or a fee calculation issue.
 
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class SettlementReconciliationService {
 
    private final PaymentRepository paymentRepository;
    private final ReconciliationBreakRepository breakRepository;
 
    public ReconciliationResult reconcile(List<SettlementRecord> settlementRecords,
                                          LocalDate settlementDate) {
 
        List<ReconciliationBreak> breaks = new ArrayList<>();
        BigDecimal totalSettled = BigDecimal.ZERO;
        int matchedCount = 0;
 
        for (SettlementRecord settled : settlementRecords) {
            Optional<CaptureRecord> captured = paymentRepository
                    .findCaptureByRrn(settled.getRrn());
 
            if (captured.isEmpty()) {
                // Settled by acquirer — no capture record found internally
                breaks.add(ReconciliationBreak.missingCapture(settled));
                log.warn("Settlement break — no capture found: rrn={}, amount={}",
                        settled.getRrn(), settled.getNetAmount());
                continue;
            }
 
            CaptureRecord capture = captured.get();
 
            if (!settled.getNetAmount().equals(computeExpectedNet(capture))) {
                // Amount mismatch — fee calculation discrepancy or scheme error
                breaks.add(ReconciliationBreak.amountMismatch(settled, capture));
                log.warn("Settlement break — amount mismatch: rrn={}, settled={}, expected={}",
                        settled.getRrn(), settled.getNetAmount(),
                        computeExpectedNet(capture));
                continue;
            }
 
            totalSettled = totalSettled.add(settled.getNetAmount());
            matchedCount++;
        }
 
        // Flag captures with no corresponding settlement record
        List<CaptureRecord> unsettledCaptures = paymentRepository
                .findCapturesWithoutSettlement(settlementDate);
 
        for (CaptureRecord unsettled : unsettledCaptures) {
            breaks.add(ReconciliationBreak.missingSettlement(unsettled));
            log.warn("Settlement break — capture not settled: rrn={}", unsettled.getRrn());
        }
 
        if (!breaks.isEmpty()) {
            breakRepository.saveAll(breaks);
        }
 
        return ReconciliationResult.builder()
                .settlementDate(settlementDate)
                .totalRecords(settlementRecords.size())
                .matchedCount(matchedCount)
                .breakCount(breaks.size())
                .totalSettledAmount(totalSettled)
                .breaks(breaks)
                .build();
    }
 
    private BigDecimal computeExpectedNet(CaptureRecord capture) {
        // Net = gross - interchange - scheme fee - acquirer fee
        // Interchange rates are loaded from a fee schedule per card type and MCC
        return capture.getCaptureAmount()
                .subtract(capture.getInterchangeFee())
                .subtract(capture.getSchemeFee())
                .subtract(capture.getAcquirerFee());
    }
}
```

---
 
## Approaches
 
### Direct acquirer integration via ISO 8583
 
Connect directly to the acquirer using the ISO 8583 message standard over a dedicated TCP/IP connection (or HTTPS for modern acquirers). You own the full message construction, response handling, and file-based clearing submission.
 
**Fits:** Payment facilitators, large merchants with direct acquiring relationships, and fintechs building their own payment processing stack.
 
**Advantages:** Full control over message content, retry logic, and clearing timing. No PSP markup on interchange & direct access to raw response codes and decline reasons.
 
**Limitations:** High integration complexity. Requires PCI DSS Level 1 compliance, ongoing scheme certification cycles for each major Visa/Mastercard release & scheme rule compliance is your responsibility entirely.
 
---
 
### Acquirer gateway API (REST/SOAP abstraction)
 
Most acquirers provide a modern API layer over ISO 8583, you submit JSON or XML; they handle the scheme messaging. Clearing is still typically file-based but the format is proprietary to the acquirer.
 
**Fits:** Fintechs building on top of an acquiring relationship without needing message-level control.
 
**Advantages:** Significantly lower integration complexity. Acquirer handles scheme certification and it is faster to market.
 
**Limitations:** Less control over message content and retry behaviour. Some acquirers abstract away response codes in ways that lose useful signal. Clearing file formats vary considerably between acquirers and require custom parsing.
 
---
 
### 3. PSP-as-acquirer (aggregated merchant model)
 
Use a PSP that holds the acquiring relationship and aggregates merchants underneath. Stripe, Adyen, Braintree, and Worldpay all operate this model.
 
**Fits:** Platforms and marketplaces where speed of onboarding matters more than commercial optimisation, or where individual merchant volume does not justify a direct acquiring relationship.
 
**Advantages:** No scheme certification, no direct acquirer integration, fast onboarding and PSP handles all clearing and settlement complexity.
 
**Limitations:** PSP sits between you and your funds. Settlement timing and chargeback handling are subject to PSP policy, not just scheme rules & costs are significantly higher than direct interchange rates at volume.
 
---

## Recommendations
 
**For most fintechs building their own payment stack:** start with an acquirer gateway API rather than raw ISO 8583. The integration complexity of direct scheme messaging, including STAN management, session-level connection handling, and clearing file construction, is significant, and the commercial benefit of going lower-level only materialises at meaningful transaction volumes.
 
**Take response code handling seriously from day one.** It is the area most commonly under-engineered in early integration. Implement the retry classification correctly, track retry counts against scheme limits, and build alerting around hard-decline response codes that should never appear in normal operation.
 
**Build settlement reconciliation before you need it.** The gap between what you think settled and what actually settled only becomes apparent when you reconcile. Running daily reconciliation from the first day of production is far less painful than reconstructing months of settlement history after the fact. 
 
**Understand clearing cut-off windows and build operational monitoring around them.** A missed clearing cut-off is a cash flow event, not just a technical issue. If your clearing submission fails silently at 23:58 because of a file format error, you will not know until the next business day, by which point your settlement date has slipped. Treat clearing submission as a critical operational process with explicit success confirmation and alerting.
 
---

## Further Reading
 
- [Visa Product and Service Rules](https://usa.visa.com/support/consumer/visa-rules.html): The authoritative source on scheme rules, retry limits, and chargeback reason codes. 
- [Mastercard Transaction Processing Rules](https://www.mastercard.com/content/dam/public/mastercardcom/na/global-site/documents/transaction-processing-rules.pdf): Mastercard equivalent for scheme rules and retry mechanisms. 
- [ISO 8583 — Wikipedia overview](https://en.wikipedia.org/wiki/ISO_8583): A readable introduction to the message format before diving into the formal specification.
- [Adyen — Authorisation, Capture, and Settlement](https://docs.adyen.com/online-payments/capture/): Adyen's documentation is one of the clearest practical explanations of the dual-message flow available publicly.
- [Stripe Radar — Decline Codes](https://stripe.com/docs/declines/codes): Useful reference for mapping response codes to cardholder-facing messaging, even if you are not using Stripe as your acquirer.

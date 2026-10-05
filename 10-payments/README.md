# Payments Domain — Senior Interview Guide

This chapter covers the payment-domain knowledge expected from a senior engineer or architect on an acquiring or payment-switch platform: card-scheme ecosystems, ISO 8583 messaging, the Auth–Clear–Settle lifecycle, switch and host design, reliability under timeouts, and the Java/Kafka engineering needed for low-latency, high-TPS flows. The goal is to show that you have reasoned about real money movement, not only about generic microservices.

> **How to answer:** state the business rule or message flow first, then the mechanism (fields, state, timers), then the failure mode (timeout, duplicate, mismatch), then the engineering trade-off. Distinguish what ISO 8583 says from what Visa, Mastercard, or a given issuer actually does: scheme specifications are proprietary and differ by region and release, so say when you would verify against the scheme manuals.

## Contents

1. [Ecosystem and business model](#1-ecosystem-and-business-model)
2. [ISO 8583 messaging](#2-iso-8583-messaging)
3. [Authorization, clearing, and settlement](#3-authorization-clearing-and-settlement)
4. [Cards, EMV, PIN, and security](#4-cards-emv-pin-and-security)
5. [Switch and acquiring host architecture](#5-switch-and-acquiring-host-architecture)
6. [Reliability, timeouts, and consistency](#6-reliability-timeouts-and-consistency)
7. [Java, Spring, and Kafka for high TPS](#7-java-spring-and-kafka-for-high-tps)
8. [Reconciliation, ledger, and disputes](#8-reconciliation-ledger-and-disputes)
9. [Operations, certification, and scenarios](#9-operations-certification-and-scenarios)
10. [Rapid revision](#10-rapid-revision)

---

## 1. Ecosystem and business model

### 1. Who are the parties in a card payment?

The four-party model: the **cardholder**, the **merchant**, the **issuer** (the cardholder's bank, which issued the card and bears credit/fraud risk), and the **acquirer** (the merchant's bank or processor, which accepts transactions on the merchant's behalf). The **card scheme** (Visa, Mastercard) sits between issuer and acquirer, owning the rules, the network, and the clearing and settlement services.

Money flows from the issuer to the acquirer through the scheme, and from the acquirer to the merchant minus fees. A senior answer also names the processors and gateways that often operate these roles on behalf of banks.

### 2. What is the difference between an acquirer, a processor, a payment gateway, a PSP, and a payment facilitator?

- **Acquirer:** the licensed scheme member that holds the merchant relationship and settlement liability.
- **Processor:** runs the technology (authorization, clearing, settlement files) for an acquirer or issuer; may or may not be the licensed member.
- **Gateway:** merchant-facing API and security layer that routes transactions to a processor/acquirer.
- **PSP:** a broader provider bundling gateway, acquiring, and often alternative payment methods.
- **Payment facilitator (PayFac):** a registered entity that onboards sub-merchants under its own merchant account and takes on their risk and settlement.

Boundaries blur in practice. In an interview, say which role the platform you are designing plays, because it determines liability, scheme connectivity, and PCI scope.

### 3. How does a three-party scheme differ from the four-party model?

In a three-party scheme (for example American Express historically), the scheme is also the issuer and acquirer, so there is no interchange between separate banks and the scheme controls both sides of the relationship. Many such schemes now also license issuing or acquiring to third parties.

For an engineer the difference shows in connectivity and message dialects: each scheme has its own specification, certification process, and settlement mechanism, and a switch usually carries several scheme adapters behind one canonical model.

### 4. What are interchange, scheme fees, and the merchant discount rate?

**Interchange** is the fee the acquirer pays the issuer, set by the scheme (or regulated, as in the EU/UK consumer caps) per card type, region, and transaction category. **Scheme fees** are paid to Visa/Mastercard for network use. The **merchant discount rate (MDR)** is what the merchant pays the acquirer: interchange + scheme fees + the acquirer's margin.

Pricing is either blended (a single rate) or **interchange++** (pass-through plus a markup). Engineering consequence: the platform must compute interchange per transaction at clearing time, which requires card product, region, MCC, entry mode, and data-quality fields to be captured correctly at authorization.

### 5. Where does an acquiring platform sit, and what are its main subsystems?

It sits between merchant channels (POS terminals, e-commerce gateways, ATMs) and the card schemes. Typical subsystems: channel/terminal gateways, a **payment switch** (routing, translation, authorization flow), an **acquiring host** (merchant, terminal, and limit management, risk rules), scheme connectivity adapters, a **clearing and settlement engine**, a **ledger**, reconciliation, dispute management, and reporting/payout.

A useful diagram has a real-time path (milliseconds, availability-critical) and a deferred path (clearing, settlement, reconciliation: throughput- and correctness-critical). Keep them decoupled.

### 6. How do card-present and card-not-present transactions differ for the platform?

Card-present (chip, contactless, magstripe, PIN) carries cryptographic evidence from the card (EMV data in DE55) and a POS entry mode, giving lower fraud rates and lower interchange. Card-not-present (e-commerce, MOTO, recurring) has no physical proof, so it depends on CVV2, AVS, 3-D Secure, tokens, and risk scoring, and carries higher fraud and chargeback exposure.

Liability shifts accordingly: a 3-D Secure-authenticated e-commerce transaction typically moves fraud liability to the issuer, while a non-authenticated one stays with the merchant/acquirer.

### 7. What is single-message versus dual-message processing?

**Dual-message** (typical for credit): an authorization message (0100) reserves funds in real time, and a separate clearing message later carries final amounts. **Single-message** (typical for debit/ATM/PIN): a financial message (0200) carries authorization and clearing together, and settlement follows from the same record.

Dual-message needs matching between authorization and clearing records (and handles amount changes); single-message makes the real-time message the financial record, which raises the cost of a lost or duplicated message.

### 8. What is a BIN/IIN and how is it used?

The Bank/Issuer Identification Number is the leading digits of the PAN identifying the issuer and card range. Schemes moved from 6-digit to 8-digit BINs (effective 2022), so lookups must not assume six digits.

The switch uses BIN tables for routing (which scheme/issuer), for product and region (interchange, domestic versus international), and for fraud checks. BIN tables are downloaded from the schemes, updated frequently, and must be hot-reloadable with atomic swap and a longest-prefix-match structure.

---

## 2. ISO 8583 messaging

### 9. What is ISO 8583 and why does it still dominate card payments?

ISO 8583 is a message standard for financial transaction card originated messages: a compact, binary-friendly format with a message type indicator, a bitmap, and numbered data elements. It persists because every scheme, issuer host, and acquirer host speaks it, it is efficient over long-lived TCP links, and replacing it requires coordinated change across the whole industry.

Newer ISO 20022 and REST/JSON APIs are used at the edges (merchant APIs, account-to-account), but scheme authorization traffic is still overwhelmingly ISO 8583-derived.

### 10. How is an ISO 8583 message structured?

Three parts after any network header: the **MTI** (four digits), one or more **bitmaps** indicating which data elements (DEs) are present, and the **data elements** themselves in field order. Each DE has a fixed definition: type (numeric, alphanumeric, binary), length (fixed, LLVAR, or LLLVAR), and encoding.

```text
[length prefix][MTI 0200][primary bitmap 8 bytes][secondary bitmap 8 bytes?][DE2 LL+PAN][DE3 ...][DE4 ...] ...
```

The parser is driven by a field definition table (a "packager"), not by hardcoded offsets.

### 11. How do you decode the MTI?

Each digit has a meaning: version (0 = 1987, 1 = 1993, 2 = 2003), message class (1 authorization, 2 financial, 3 file action, 4 reversal/chargeback, 5 reconciliation, 6 administrative, 7 fee collection, 8 network management), function (0 request, 1 request response, 2 advice, 3 advice response, 4 notification), and origin (0 acquirer, 2 issuer, and so on).

| MTI | Meaning |
|---|---|
| 0100 / 0110 | Authorization request / response |
| 0200 / 0210 | Financial request / response |
| 0220 / 0230 | Financial advice / response |
| 0400 / 0410 | Reversal request / response |
| 0420 / 0430 | Reversal advice / response |
| 0800 / 0810 | Network management request / response |

Schemes may use only a subset and differ in details (for example which MTIs they use for advices).

### 12. How does the bitmap work?

The primary bitmap is 64 bits; bit *n* set means DE *n* is present. Bit 1 signals a secondary bitmap covering DEs 65–128 (and, in some variants, bit 65 signals a tertiary one). The bitmap may be binary (8 bytes) or ASCII-hex (16 characters), depending on the link specification.

```java
static BitSet readBitmap(ByteBuffer buf) {
    BitSet present = new BitSet(129);
    long primary = buf.getLong();
    for (int de = 1; de <= 64; de++) {
        if ((primary & (1L << (64 - de))) != 0) present.set(de);
    }
    if (present.get(1)) {
        long secondary = buf.getLong();
        for (int de = 65; de <= 128; de++) {
            if ((secondary & (1L << (128 - de))) != 0) present.set(de);
        }
    }
    return present;
}
```

Bit 1 is the most significant bit: a classic off-by-one source.

### 13. What field encodings must a parser handle?

Fixed and variable lengths (LLVAR has a two-digit length prefix, LLLVAR three), numeric fields in ASCII or **BCD** (packed, two digits per byte), alphanumeric in ASCII or **EBCDIC** (common on mainframe links), raw binary fields (PIN block, MAC, bitmaps), and nested structures: DE55 carries EMV data as **BER-TLV**, and DE48/DE62/DE63 and similar private fields carry scheme-specific subfields.

Pitfalls: length counted in digits versus bytes, wrong charset, padding rules, and leading zeros in numeric fields. Parse into typed fields and keep the raw bytes for audit and forwarding.

### 14. Which data elements should you know?

| DE | Meaning | Notes |
|---|---|---|
| 2 | PAN | LLVAR; never log in clear |
| 3 | Processing code | Transaction type + from/to account |
| 4 | Transaction amount | 12 digits, minor units, implied decimal |
| 7 | Transmission date/time | MMDDhhmmss, GMT |
| 11 | STAN | System trace audit number, 6 digits |
| 12/13 | Local time/date | Terminal local |
| 14 | Expiry date | YYMM |
| 18 | Merchant category code | Drives interchange and risk |
| 22 | POS entry mode | Chip, contactless, magstripe, manual, e-commerce |
| 25 | POS condition code | Attended, unattended, MOTO |
| 32 | Acquiring institution ID | |
| 37 | Retrieval reference number | RRN, 12 characters |
| 38 | Authorization ID | Approval code |
| 39 | Response code | |
| 41/42 | Terminal ID / merchant ID | |
| 49 | Currency code | ISO 4217 numeric |
| 52 | PIN block | Encrypted |
| 55 | EMV chip data | TLV |
| 90 | Original data elements | For reversals |
| 128 | MAC | |

Layouts for DE48, 60–63, and others are scheme-specific.

### 15. Is ISO 8583 one standard? How do you handle scheme dialects?

No. The standard defines a framework; each scheme, and often each issuer or processor link, defines its own field usage, lengths, encodings, private fields, and mandatory elements. Two links that both say "ISO 8583" are rarely wire-compatible.

Design for it: a **canonical internal model**, one **packager/codec per link** loaded from configuration or code-generated from the spec, translation at the adapter boundary only, and conformance tests per dialect using recorded sample messages.

### 16. How are amounts and currencies represented?

DE4 is an integer count of **minor units** (cents) with an implied decimal point, and DE49 gives the ISO 4217 numeric currency. The number of minor-unit digits depends on the currency: JPY has 0, USD 2, KWD and BHD 3.

Never use floating point. In Java, carry a `long` of minor units plus a currency object that owns the exponent, and convert to `BigDecimal` only for display or rate arithmetic. Other amount fields (DE5 settlement amount, DE6 cardholder billing amount, DE54 additional amounts) carry conversion results and cashback.

### 17. What do response codes in DE39 tell you, and how should they be handled?

DE39 conveys the issuer or switch decision: `00` approved, `05` do not honor, `51` insufficient funds, `54` expired card, `55` incorrect PIN, `57` transaction not permitted, `91` issuer or switch inoperative, `96` system malfunction, among many others. Schemes add their own codes and advice fields (for example Mastercard's Merchant Advice Code).

Classify them: approvals, **soft declines** (retry later may succeed: `51`, `91`, `96`), **hard declines** (do not retry: lost/stolen/invalid card, closed account), and **technical** (timeouts, format errors). Retry policy, merchant guidance, and scheme "excessive retry" fees all depend on this classification.

### 18. What are network management messages?

0800/0810 messages manage the link rather than money: **sign-on/sign-off**, **echo test** (keep-alive), and **key exchange**, identified by the network management information code in DE70 (for example 001 sign-on, 002 sign-off, 301 echo test).

A scheme or issuer link is only usable after sign-on succeeds, and a missed echo response should mark the link degraded and stop routing to it. Connection management is part of the product, not a detail.

### 19. How are reversals represented?

A 0400 reversal request (and 0420 advice) carries enough data to identify the original, usually DE90 (original MTI, STAN, transmission date/time, acquirer and forwarding institution IDs), plus the reversal amount for partial reversals. It is used when the acquirer cannot confirm the result of an authorization or the transaction was cancelled before completion.

Reversals must be **idempotent** and delivered with store-and-forward until acknowledged, because losing one leaves a cardholder's funds held.

### 20. What is the difference between a request and an advice?

A **request** (0100/0200) asks the receiver for a decision and expects a response. An **advice** (0120/0220/0420) informs the receiver of something that already happened (for example the issuer was unavailable and a stand-in system approved, or a terminal completed an offline transaction) and the receiver must accept it, not decline it.

Advices must be persisted and retried until acknowledged. Treating an advice as a request that can fail breaks the money trail.

### 21. How would you implement an ISO 8583 parser in Java, and what are the pitfalls?

Use a proven library (jPOS is the de facto Java implementation, with packagers and a channel/server framework) or a Netty codec built around a field-definition table. Decode on the I/O thread into a typed message without reflection or regex in the hot path, and keep the raw frame.

Pitfalls: logging PAN/track/PIN fields, charsets, signed vs unsigned bytes, length prefix handling and partial TCP frames, unknown DEs (reject versus pass-through), and allocation per message that causes GC pressure at high TPS. Always fuzz and property-test the codec.

---

## 3. Authorization, clearing, and settlement

### 22. Walk through the Auth–Clear–Settle lifecycle.

1. **Authorization:** in real time the terminal or gateway sends a request; the acquirer's switch routes it through the scheme to the issuer, who approves or declines and places a hold on the cardholder's available funds. No money moves.
2. **Clearing:** later (minutes to days) the merchant's final transactions are submitted. The acquirer sends **presentments** to the scheme, which routes them to the issuer and computes interchange and fees. The issuer posts the amount to the cardholder's account.
3. **Settlement:** the scheme nets obligations between members and funds move between issuer and acquirer settlement accounts, typically the next business day. The acquirer then pays the merchant, net of fees.

The key point: authorization is a promise, clearing is the financial claim, settlement is the transfer.

### 23. What does an authorization actually do?

It asks the issuer whether the card is valid and the account can support the amount, runs the issuer's fraud and rule checks, verifies cryptograms/PIN/CVV, and returns an approval code. The issuer places a hold that reduces available balance, and may expire it if no clearing arrives (usually around seven days, longer for some categories, depending on scheme and region).

It does not move money and does not guarantee payment to the merchant by itself: the merchant is paid when the transaction is cleared and settled.

### 24. What checks does the acquirer perform before sending an authorization to the scheme?

Message validity, terminal and merchant status, MCC and product permissions, limits and velocity rules, duplicate detection, BIN-based routing and eligibility, currency and amount checks, and acquirer-side fraud/risk scoring. HSM operations such as PIN translation and MAC generation also happen here.

Each check is a latency cost and a possible false decline. Order them cheapest-first, make them in-memory where possible, and have an explicit fail-open/fail-closed policy per check.

### 25. How does clearing work, and how is it matched back to the authorization?

The acquirer builds clearing records (for Visa, historically BASE II transaction-code records such as TC 05 sales and TC 06 returns; for Mastercard, IPM files into GCMS) and submits them in files or streams. The records reference the original authorization via scheme trace data, for example Visa's Transaction Identifier or Mastercard's Banknet reference and date, together with the approval code.

Matching quality affects interchange qualification and dispute outcomes, so authorization data must be stored durably and joined to the clearing record. Exact record layouts and the current clearing platforms change by release; verify them against the scheme documentation.

### 26. How does settlement work?

Schemes compute net positions per member and settlement currency, and the amounts are moved between settlement banks or scheme settlement accounts (Visa Settlement Service, Mastercard Settlement Account Management are examples). Cut-off times, currencies, and holidays determine the value date.

The acquirer's own **merchant settlement** is separate: it aggregates cleared transactions per merchant, deducts MDR, refunds, chargebacks, and reserves, and pays out on the merchant's schedule. The acquirer carries credit risk between paying the merchant and receiving funds.

### 27. What are pre-authorization, final authorization, incremental, and partial authorization?

A **pre-authorization** reserves an estimated amount (hotels, car rental, fuel) and may later be adjusted; a **final authorization** is for a known amount the merchant intends to capture (Mastercard requires an indicator in some regions). **Incremental** authorizations raise an existing hold; **partial** authorization lets the issuer approve less than requested (for example prepaid balance), returning the approved amount so the merchant collects the remainder otherwise.

The switch must carry the right indicator fields, link subsequent messages to the original, and handle expiry and partial approval in its state machine.

### 28. What is the difference between an authorization reversal, a void, and a refund?

An **authorization reversal** releases a hold before clearing (the sale never completed). A **void** cancels a captured-but-not-yet-settled transaction. A **refund** (credit) is a new transaction after clearing that returns funds, flowing the opposite direction.

Choose the right one: a reversal costs nothing and instantly restores available balance, whereas a refund appears later and may incur fees; cardholders see the difference.

### 29. Why can the cleared amount differ from the authorized amount?

Tips and gratuity, partial shipment, final fuel amount, merchant adjustments, incremental authorizations, and foreign exchange differences. Schemes allow variance within tolerances for some categories.

The clearing engine must not assume equality: it must match on trace identifiers, tolerate permitted differences, release remaining holds with reversals, and flag out-of-tolerance cases.

### 30. How does the chargeback lifecycle work?

The cardholder disputes a transaction with the issuer, who raises a **chargeback** with a scheme reason code (fraud, goods not received, duplicate, processing error). The acquirer debits the merchant, who may submit evidence for a **representment**; unresolved cases may go to **pre-arbitration** and scheme **arbitration**. Time limits (often around 120 days from the processing date, varying by reason code) and evidence rules are strict.

The platform needs dispute case management, automatic debits and credits to the merchant ledger, document handling, and deadline tracking.

### 31. How is interchange determined at clearing?

By the card product and region, MCC, entry mode, authorization characteristics, data completeness (for example Level 2/3 data), and timeliness of clearing relative to authorization. Failure to meet the qualification criteria produces a **downgrade** to a costlier rate.

Engineers influence cost directly: sending correct POS entry mode and indicators, clearing within time limits, and preserving authorization data into clearing.

### 32. How are multi-currency transactions and DCC handled?

The transaction currency (DE49) may differ from the merchant's settlement currency and the cardholder's billing currency. The scheme converts at its rates, and DE5/DE6 plus conversion-rate fields carry the results. **Dynamic currency conversion (DCC)** lets the merchant offer the cardholder's currency at the point of sale, with strict disclosure rules.

The platform must store each amount with its currency and rate source, and settlement and ledger entries must be exact in every currency involved.

### 33. How do Visa and Mastercard differ in ways an engineer cares about?

Both use ISO 8583-derived authorization messages, but with different private fields, trace identifiers, response and advice codes, clearing formats and platforms, settlement services, mandates, and certification programs. For example, Visa carries its Transaction Identifier in a private field, while Mastercard carries Banknet data in DE63.

Do not claim detailed knowledge you lack: scheme specifications are distributed under licence to members and processors. Say what you have implemented (adapters, certification, dispute handling) and that you would verify field-level details in the current manuals.

---

## 4. Cards, EMV, PIN, and security

### 34. How does an EMV chip transaction work, and what is in DE55?

The terminal reads the card application, runs risk checks, and asks the chip to generate a **cryptogram**. For online authorization the card creates an **ARQC** over transaction data (amount, currency, date, terminal verification results, application transaction counter, unpredictable number). The terminal sends these as TLV tags in DE55 (for example 9F26 cryptogram, 9F27 cryptogram information, 9F10 issuer application data, 9F36 ATC, 95 TVR).

The issuer validates the ARQC with its HSM, and returns an **ARPC** (tag 91, issuer authentication data) in the response. The acquirer mostly relays DE55 but must parse, validate presence, and preserve it for clearing.

### 35. What is liability shift?

When a transaction is processed with a more secure method than the counterparty supports, fraud liability moves to the party that did not adopt it: a counterfeit chip-card fraud on a magstripe-only terminal falls to the merchant/acquirer; an authenticated 3-D Secure e-commerce fraud falls to the issuer.

This is why entry mode, authentication indicators (such as the ECI), and DE55 integrity are stored and forwarded accurately: they are evidence in disputes.

### 36. How is a PIN protected end to end?

The PIN is encrypted at the PIN pad into a **PIN block** (ISO formats 0, 1, 3, and 4; format 0 combines the PIN field with the PAN) using a key such as a DUKPT derived key, so a separate key encrypts every transaction. The acquirer's **HSM** translates the PIN block from the terminal key to the key shared with the scheme or issuer without exposing the clear PIN.

Rules: the PIN never exists in clear outside an HSM or secure device, is never logged, and is never stored after authorization. PCI PIN Security requirements govern key management.

### 37. What is PCI DSS and what may you store?

PCI DSS governs systems that store, process, or transmit cardholder data. The PAN may be stored only if rendered unreadable (strong cryptography, truncation, tokenization), and **sensitive authentication data** (full track data, CVV/CVC, PIN/PIN block) must not be stored after authorization, even encrypted.

Architecture goal: **minimize scope**. Isolate a card data environment, tokenize early, use P2PE at terminals, mask PAN in logs and Kafka events, restrict access, and keep audit trails. Requirements are versioned (v4.0.x is current), so cite the official standard.

### 38. What is tokenization, and how do PCI tokens differ from network tokens?

A **PCI token** is an acquirer- or gateway-issued surrogate for a PAN, usable only within that provider's vault. A **network token** (EMV payment token issued by Visa or Mastercard) replaces the PAN across the ecosystem, is bound to a device or merchant, uses a token cryptogram per transaction, and is updated automatically when the underlying card changes.

Network tokens typically improve authorization rates and security for recurring and card-on-file use. The platform must route, detokenize through the scheme token service, and keep the PAN/token mapping for clearing and disputes.

### 39. What are 3-D Secure and SCA, and where do they touch the switch?

3-D Secure (3DS2) authenticates a cardholder for e-commerce through the issuer's access control server, producing an authentication value (CAVV/AAV) and an ECI. **Strong Customer Authentication** under PSD2 in Europe requires two independent factors for many electronic payments, with exemptions (low value, low risk, merchant-initiated).

The authorization message must carry the authentication data and exemption indicators; the acquirer's risk engine decides whether to request an exemption, and a soft decline demanding SCA must be handled by re-authenticating, not by blind retry.

### 40. How do HSMs fit into the transaction path, and what do they cost in latency?

HSMs perform PIN translation, MAC generation/verification, key wrapping, and cryptogram verification (on the issuer side). Calls are network round trips of well under a few milliseconds when sized properly, but throughput is limited per device.

Design: pool and pipeline HSM connections, avoid per-message key loading, keep keys referenced by name or index, cluster for availability, and treat HSM saturation as a first-class capacity limit and failure mode (the transaction cannot proceed without it).

### 41. How is fraud and risk screening done within a tight latency budget?

Run **cheap deterministic rules** in the hot path from in-memory state (velocity counters, blocklists, merchant limits, geo/BIN checks) and call a model-scoring service only when its latency and availability fit the budget, with a strict timeout and a defined fallback. Heavier analysis runs asynchronously on the event stream and feeds back as updated rules, limits, or blocklists.

Each rule needs a measured false-decline cost. Always log the decision features for audit and model retraining.

### 42. How do you keep card data out of logs, events, and debugging tools?

Mask on the output path by construction (a message type whose `toString` returns masked values, with the raw PAN available only through an explicit method), use structured logging with field allowlists, and tokenize or encrypt PANs before they enter Kafka, databases, or tracing systems. Never put raw frames in application logs; use a protected, access-controlled audit store with its own retention.

Scan for leaks automatically (regex plus Luhn check on log samples), because accidental PAN exposure in logs is among the most common PCI findings.

---

## 5. Switch and acquiring host architecture

### 43. Describe a payment switch's architecture.

A switch accepts messages from many channels, **normalizes** them to a canonical model, validates, **routes** by BIN/merchant/rules to a scheme or issuer link, **translates** to that link's dialect, tracks the transaction state, correlates the response, and returns it to the originator. Around the flow: a risk/limit step, HSM access, a journal for audit, store-and-forward queues, and event publication to downstream systems.

```text
channels -> [gateway/adapters] -> [processing core] -> [scheme adapters] -> Visa / Mastercard / issuers
                                   |  routing, risk, state, journal
                                   +--> outbox -> Kafka -> clearing, ledger, notifications, analytics
```

### 44. Why separate connection gateways from stateless processing?

Scheme links are long-lived, stateful TCP sessions with sign-on, sequence, and a limited number of connections, so they do not scale horizontally like HTTP. Keep a small **connection tier** that owns sockets and framing, and make the **processing tier** stateless (state in a store or in the message), scaled out behind a queue or lightweight RPC.

This separation lets you deploy and scale processing without dropping scheme sessions, fail over connection owners independently, and enforce per-link flow control.

### 45. How does routing work?

Rules map a transaction to a destination by BIN/IIN range, card scheme, transaction type, currency, merchant, time, and link health. Use a longest-prefix BIN structure loaded in memory, reload atomically from scheme-supplied tables, and make rule evaluation deterministic and testable.

Add least-cost routing where permitted (for example co-badged debit), failover to alternate links, and an audit record of the selected route and reason. Never route on mutable state that could change mid-transaction without recording it.

### 46. What is a canonical message model and why use one?

An internal, scheme-independent representation of a transaction (typed fields for amount, currency, PAN token, entry mode, merchant, EMV, authentication data) that adapters map to and from. It stops scheme-specific field numbers leaking through the business logic and lets you add a link without changing the core.

Keep the original raw message alongside the canonical one for audit and for fields your model does not yet understand, and test mappings with golden message files.

### 47. How do you correlate asynchronous responses to requests?

The switch writes a request on a shared socket and later receives the response on the same socket in any order. It stores each in-flight request under a correlation key (for example STAN + RRN + terminal ID + transmission date/time, plus link ID) with a timeout, and completes it when the matching response arrives.

```java
final class PendingRequests {
    private final ConcurrentHashMap<String, CompletableFuture<IsoMessage>> inFlight = new ConcurrentHashMap<>();
    private final HashedWheelTimer timer = new HashedWheelTimer();

    CompletableFuture<IsoMessage> send(Channel ch, IsoMessage req, Duration timeout) {
        String key = correlationKey(req);
        var future = new CompletableFuture<IsoMessage>();
        if (inFlight.putIfAbsent(key, future) != null) {
            throw new IllegalStateException("duplicate in-flight key " + key);
        }
        Timeout t = timer.newTimeout(x -> {
            if (inFlight.remove(key, future)) future.completeExceptionally(new TimeoutException(key));
        }, timeout.toMillis(), TimeUnit.MILLISECONDS);
        future.whenComplete((r, e) -> t.cancel());
        ch.writeAndFlush(req);
        return future;
    }

    void onResponse(IsoMessage resp) {
        var future = inFlight.remove(correlationKey(resp));
        if (future == null) { lateResponses.increment(); return; }   // late: reversal path
        future.complete(resp);
    }
}
```

A wheel timer avoids one scheduled task per message. A response with no pending entry is a **late response** and must trigger reversal logic, not be dropped.

### 48. How do you manage scheme and issuer links?

Each link has a state machine: connecting, connected, signed-on, degraded (missed echoes), and down. Maintain several connections per scheme endpoint across data centres, send echo tests, rebalance on failure, limit in-flight messages per link, and stop routing to unhealthy links quickly.

Respect scheme-imposed windows and sequence rules, and test failover with the scheme's certification environment. A "healthy TCP connection" is not equivalent to a "link able to authorize": track sign-on status and response-time health.

### 49. What does the acquiring host manage besides routing?

Merchant and terminal onboarding data, MCC and product permissions, per-merchant limits and velocity, terminal keys, fee plans, settlement schedules, and risk status. The authorization path reads this reference data at very high rate and rarely writes it.

Treat it as **read-mostly reference data**: cache locally with versioned snapshots, propagate changes by events with explicit staleness bounds, and keep the system of record separate from the hot path.

### 50. What states does a transaction pass through?

A typical set: `RECEIVED`, `VALIDATED`, `ROUTED`, `AWAITING_RESPONSE`, `APPROVED` or `DECLINED`, `REVERSAL_PENDING`, `REVERSED`, `CAPTURED`/`CLEARED`, `SETTLED`, `REFUNDED`, `CHARGEBACK`. Late responses and timeouts add edge cases (for example `TIMED_OUT_APPROVED`).

Model it as an explicit state machine with allowed transitions, enforced in code and by a unique key plus version check in storage. Illegal transitions (an approval after a reversal) become alerts, not silent overwrites.

### 51. What must be written synchronously on the authorization path?

The minimum needed to make the outcome recoverable: a durable journal of the request and response (or at least the decision), the state transition, and the data needed for later reversal and clearing. Everything else (ledger postings, notifications, analytics, enrichment) can be emitted afterwards from the persisted record via an **outbox**.

The trade-off is durability versus latency: use fast local or replicated storage for the journal, group commits where safe, and be explicit about what is lost on a crash.

### 52. How would you design the clearing and settlement engine?

A deferred pipeline: collect captured transactions, validate and enrich them, compute interchange and fees, produce scheme-format clearing files or streams, submit, process scheme acknowledgements and rejects, then compute net settlement and merchant payouts. Run it as partitioned, **idempotent batch or micro-batch jobs** with checkpoints.

Control totals (counts and sums) at every stage detect loss and duplication. Rejected records need repair workflows, and a resubmission must not double-clear. Throughput matters, but correctness and auditability matter more.

---

## 6. Reliability, timeouts, and consistency

### 53. What do you do when an authorization request times out?

The outcome is **unknown**: the issuer may have approved and placed a hold. Respond to the merchant with a decline or timeout according to the channel rules, and immediately **send a reversal** (0400) for the original, retried with store-and-forward until acknowledged. Never silently retry the original as a new authorization, which could double-hold.

Also handle the **late response**: if an approval arrives after the timeout, match it to the original and ensure a reversal exists. Metrics on timeouts per link and the reversal backlog are key production indicators.

### 54. How do you detect duplicate messages and keep processing idempotent?

Build a duplicate key from stable message identifiers (for example link, STAN, RRN, transmission date/time, terminal/merchant, amount) and record it atomically with the transaction under a unique constraint. A duplicate request returns the **stored original response** instead of reprocessing.

Terminals and gateways retry on timeouts, and the scheme may resend advices, so every consumer, including Kafka consumers, must be idempotent. STAN wraps at six digits per day, so include the date/time in keys and expire them correctly.

### 55. What if a reversal arrives before the original transaction?

Networks and queues reorder messages, so a reversal can reach the platform first. Record the reversal as a **tombstone** keyed by the original's identifiers; when the original arrives, detect the tombstone and decline or immediately reverse it without sending it to the issuer.

This out-of-order case is where a simple "find the original then reverse it" implementation silently loses reversals. Test it explicitly.

### 56. What is stand-in processing, and what does it mean for the acquirer?

When the issuer cannot be reached, the scheme (or issuer processor) may approve or decline on the issuer's behalf using pre-agreed parameters, then send an **advice** to the issuer later. The acquirer sees an approval with a stand-in indicator and may not distinguish it from a normal approval.

If the **acquirer's** connection to the scheme is down, the acquirer must follow its own policy: decline, or for low-risk categories apply limited local rules and store-and-forward advice if the scheme rules permit. Both are business decisions with risk limits, not technical defaults.

### 57. How do you get "exactly-once" money movement?

You cannot rely on exactly-once delivery across networks and brokers. Use **at-least-once delivery plus idempotent processing**: unique business keys with constraints, deduplication tables, idempotent producers, and ledger entries keyed by the originating transaction event ID so that replay creates no second posting.

Kafka transactions give atomic read-process-write within Kafka, but cannot make an external database or scheme call atomic with it, so the dedupe key is still essential.

### 58. How do you handle concurrent transactions on the same card or merchant?

Partition by the entity that needs ordering (card token or account for limits, merchant for balances) so a single owner serializes updates. Use optimistic locking or compare-and-set for counters and limits, and avoid single hot rows (for example one merchant balance row updated by every transaction).

For hot merchants, record **append-only entries** and aggregate (or use sharded counters) rather than update one row in place.

### 59. How should retries, store-and-forward, and dead-letter handling work for advices and reversals?

Persist the message before sending, retry with capped exponential backoff and jitter, keep ordering per original transaction, and stop only when acknowledged by the receiver or manually resolved. Messages that cannot be parsed or are rejected go to a repair queue with an owner and an alert, not to a silent dead-letter topic.

Count the age of the oldest unacknowledged reversal: it is an operational and customer-impact metric.

### 60. What happens when the database, Kafka, or the HSM is unavailable?

Define behavior per dependency before the incident. **Database/journal down:** the switch usually must stop approving (it cannot guarantee recoverability), or fall back to a replicated store. **Kafka down:** authorization continues because events are in the outbox and published later; backlog grows, so size and alert on it. **HSM down:** transactions requiring crypto fail (typically decline with a technical code), so cluster HSMs.

Prefer fail-closed for money-affecting writes and fail-open only for derived or advisory functions, and test each failure in a game day.

### 61. How do you design multi-data-centre resilience for a payment switch?

Run active-active or active-passive sites each with their own scheme connections, with the processing tier stateless and the journal replicated synchronously (or within a defined RPO) to the other site. Scheme endpoints, IP allow-lists, and session rules often make failover a coordinated operation, so rehearse it.

Define RTO/RPO per component: authorization journal and ledger usually need near-zero RPO, while analytics tolerate more. Avoid split-brain by giving each in-flight transaction a single owner.

### 62. How do you keep authorization state and the ledger consistent?

Write the authorization record and an **outbox row** in the same local transaction, then publish to Kafka from the outbox (polling or CDC). Ledger consumers apply entries idempotently keyed by the event ID. Reconciliation catches residual differences.

```java
@Transactional
void on(AuthorizationApproved e) {
    authorizations.save(e.toRecord());
    outbox.save(OutboxEvent.of("auth.approved", e.id(), e.toPayload()));   // same transaction
}
```

A dual write (database then Kafka without an outbox) can lose or duplicate events on a crash.

---

## 7. Java, Spring, and Kafka for high TPS

### 63. What is the latency budget for an authorization, and where does the time go?

The end-to-end time is dominated by the scheme and issuer round trip, so the acquirer's own share must be small: typically tens of milliseconds for parsing, validation, routing, risk, HSM, journaling, and translation, within an overall scheme-imposed timeout of several seconds. Budget each step and track percentiles (p99, p99.9), not averages.

Typical costs: network hops, serialization, the synchronous journal write, HSM calls, and GC pauses. Remove avoidable hops and keep the hot path in a small number of processes.

### 64. What threading model suits a high-TPS switch in Java?

Network I/O on a **non-blocking event loop** (Netty), with business logic either run on the same loop when it is non-blocking and short, or handed to a bounded executor. With Java 21+, **virtual threads** allow simple blocking-style code for calls to storage or the HSM without a thread per request cost, but pinned monitors and native blocking calls need care.

Spring Boot is appropriate for configuration, dependency injection, metrics, health, and the management plane, but the hot path should not depend on heavy framework features (reflection-driven proxies, per-request object graphs, servlet filters). Measure before choosing.

### 65. How do you manage GC and memory for low latency?

Reduce allocation per transaction (reuse buffers, pooled direct `ByteBuf`s, avoid string conversions), keep heap sizes appropriate, and select a low-pause collector (ZGC or Shenandoah, or tuned G1) validated against p99.9 under load. Keep large reference data in compact structures.

Diagnose with GC logs, JFR, allocation profiling, and safepoint analysis. Tail latency often comes from GC, lock contention, or thread pool queueing, not from average CPU use.

### 66. Where does Kafka fit in a payment platform, and where does it not?

Kafka is for **asynchronous, durable, replayable event flow**: transaction events to the ledger, clearing capture, fraud analytics, notifications, merchant reporting, and audit. It generally should **not** sit inside the synchronous authorization request/response path, since it adds latency and a failure dependency to a path with a hard scheme timeout.

Key by the entity needing ordering (for example card token or merchant), use `acks=all` with idempotent producers, size partitions for peak throughput, and publish through the outbox.

### 67. How do you build reliable Kafka consumers for financial events?

Make processing **idempotent** by a unique event ID, commit offsets after the effect is durably recorded, bound retries, and route poison messages to a repair path with alerts. Monitor consumer lag per partition and its age, since lag in a ledger consumer delays balances and reports.

```java
@KafkaListener(topics = "auth-events", groupId = "ledger")
@Transactional
void on(AuthEvent e) {
    if (!processed.tryInsert(e.eventId())) return;   // unique constraint: replay-safe
    ledger.post(e);
}
```

Schema evolution (Avro/Protobuf with a registry and compatibility rules) matters because many consumers replay old events.

### 68. How do you choose and shape the database for the hot path?

Start from the access pattern: write-heavy journal keyed by transaction identity, point reads by correlation key, and reference-data reads. Options include a relational database with partitioning/sharding by a stable key, or a distributed store when you need horizontal scaling; the choice must support strict uniqueness and durable commits.

Avoid cross-shard transactions on the hot path, hot rows, and large unbounded tables (partition by date and archive). Keep reporting queries off the primary.

### 69. How do you apply backpressure and load shedding in payments?

Bound every queue and pool, limit in-flight requests per link, and reject early with a clear technical decline when saturated, rather than let latency grow until every request times out. Prioritize (for example reversals and advices ahead of new authorizations) and shed by policy.

Never drop silently or shed financial advices. Coordinate with merchants through rate limits and with the scheme according to its volume agreements.

### 70. How do you performance-test a switch?

Use a **scheme/issuer simulator** that reproduces latency distributions, declines, timeouts, and out-of-order responses, and generate realistic message mixes at target and peak TPS using an open-loop load model, which avoids coordinated omission. Measure p50/p99/p99.9, error and timeout rates, GC, CPU, and downstream lag.

Run soak tests (memory leaks, file descriptors), failover tests, and replay of recorded traffic with PAN masked. A capacity statement should be backed by tested numbers with headroom.

### 71. What should you observe in a payment platform?

Business and technical signals together: approval rate and response-code distribution (by link, BIN, merchant, and acquirer), authorization latency percentiles per hop, timeouts, late responses, reversal backlog age, link state, duplicate rate, HSM latency, outbox lag, Kafka consumer lag, and reconciliation breaks.

Propagate a **correlation ID** (derived from STAN/RRN and an internal ID) through logs, traces, and events, with PAN masked. A sudden drop in approval rate or spike in `91`/`96` often shows an issuer or link problem before any technical alarm fires.

---

## 8. Reconciliation, ledger, and disputes

### 72. How would you design the ledger for an acquirer?

An **immutable, double-entry** ledger: every posting has balanced debit and credit entries in accounts such as scheme receivable, merchant payable, fee income, interchange expense, reserves, and chargeback receivable. Corrections are new entries, never updates.

Use unique keys derived from the source event for idempotency, store amounts as minor units with currency, and keep running balances as derived views or snapshots. The ledger is the financial source of truth; transaction status tables are not.

### 73. What is reconciliation and what are its types?

Reconciliation compares independent records of the same money movement and explains differences. For an acquirer, typical three-way matching: **switch records** versus **scheme clearing/settlement reports** versus **bank statements** and the **merchant ledger**.

Differences (breaks) include missing, duplicate, amount, currency, timing, and fee differences. Automate matching on trace identifiers and amounts, classify the exceptions, and give operations a workflow with ageing and ownership.

### 74. How should money be represented in Java?

As an integer count of minor units with an explicit currency (a `Money` value type), never `double`. Use `BigDecimal` with explicit scale and rounding mode for rates, fees, and FX, and round only at defined points as the scheme or contract specifies.

```java
record Money(long minorUnits, Currency currency) {
    Money plus(Money o) {
        requireSameCurrency(o);
        return new Money(Math.addExact(minorUnits, o.minorUnits), currency);   // overflow fails loudly
    }
}
```

Use the ISO 4217 exponent per currency and test against zero- and three-decimal currencies.

### 75. How are merchant fees, reserves, and payouts calculated?

Per transaction: gross amount, interchange, scheme fees, acquirer margin, and tax, per the merchant's pricing plan. Per payout: sum of cleared sales minus refunds, chargebacks, fees, and any **rolling reserve** or hold held against risk.

The calculation must be deterministic and reproducible from stored inputs (rate plan version, rate table version), because merchants and auditors will challenge individual amounts.

### 76. How do you process very large clearing and settlement files safely?

Stream and partition the file rather than loading it, validate header and trailer **control totals**, process records idempotently (keyed by file ID and record sequence), and checkpoint progress so a restart resumes without double-posting. Quarantine bad records and report them.

Keep file integrity (checksums, signatures where specified), archive originals immutably for audit, and make reprocessing a controlled, audited operation.

### 77. What regulatory and audit concerns affect the design?

PCI DSS (card data), PSD2/SCA in Europe, data residency and privacy laws (for example GDPR), anti-money-laundering and sanctions screening, and scheme rules. Auditors expect immutable audit trails, access control with separation of duties, change management, retention policies, and evidence of testing.

Design for it: append-only journals, who-did-what logging for operational actions (manual reversals, repairs), encryption at rest and in transit, and clear data lifecycle and deletion rules compatible with retention obligations.

---

## 9. Operations, certification, and scenarios

### 78. How does certification with a scheme work, and why does it matter for design?

Before going live, an acquirer or processor must pass the scheme's connectivity and message-format certification, using test environments and scripted scenarios (approvals, declines, reversals, timeouts, EMV, e-commerce). Changes to mandated fields happen in scheduled scheme **releases**, often twice a year, and require updates and re-testing.

Design for evolution: configuration-driven field mapping, automated regression suites with recorded messages, feature flags per link, and the ability to run a new release in parallel before cutover.

### 79. How would you migrate or replace a legacy switch without downtime?

Use a **strangler approach**: place a routing layer in front, mirror (shadow) production traffic to the new platform with its responses discarded, compare decisions and messages, then move small slices (one merchant group, one BIN range, one link) with instant rollback. Move clearing and settlement only after authorization parity is proven.

Scheme link cutovers need coordination and certification, and the ledger and reconciliation must keep running across both systems during the migration.

### 80. Design an authorization engine for 150M+ users and high TPS. What are the key decisions?

Clarify peak TPS, latency SLO, schemes, channels, and regions. Then: stateless horizontally scaled processing tier behind a small connection tier that owns scheme sessions; in-memory reference data (BIN tables, merchant config); a single durable journal write on the hot path; HSM pools; deterministic risk rules in-process; an outbox feeding Kafka for ledger, clearing capture, and analytics; and a deferred clearing/settlement engine with reconciliation.

Then the senior depth: the timeout/reversal path, idempotency keys, ordering per card or merchant, multi-DC failover, overload policy, and observability by response code and link. Explain what you would measure before scaling further.

### 81. Production scenario: the approval rate suddenly drops and `91`/`96` responses rise. What do you do?

Triage by dimension: which link, BIN range, issuer, merchant, or region? If it is one issuer, the problem is likely upstream: check scheme advisories, stand-in activity, and link health. If it is one link, check sign-on state, echo response times, connection counts, and recent releases. If everything, look at your own journal, HSM, GC, and thread pools.

Mitigate (reroute, fail over a link, shed non-critical load, roll back a release), communicate impact to merchants and operations, ensure reversals and advices are not backing up, and afterwards reconcile the affected window.

### 82. How do you balance coding and architecture in a hands-on architect role?

Write the **critical components** yourself where the design risk is highest (codec, correlation and timeout handling, idempotency, state machine, ledger posting), and use those as proof of the architecture. Delegate well-understood plumbing, and review rather than rewrite others' code.

Communicate through decision records (context, options, trade-offs, failure modes), sequence diagrams of the money flows, and explicit invariants ("no approval without a journal entry"). Explain a design to non-engineers by tracing one transaction through it.

### 83. How do you demonstrate genuine payment experience in an interview?

Use specific stories: a timeout you diagnosed, a reconciliation break you traced to a root cause, a scheme release you certified, a duplicate or ordering bug you fixed, a throughput or latency target you reached and how you proved it. Name the fields, states, and numbers involved.

Be honest about boundaries (for example "I implemented the Mastercard adapter and certification; I did not own settlement"), and prefer precise statements about what the specification requires versus what a particular counterparty did.

---

## 10. Rapid revision

1. Who are the parties in a card payment?
2. What is the difference between an acquirer, a processor, a payment gateway, a PSP, and a payment facilitator?
3. How does a three-party scheme differ from the four-party model?
4. What are interchange, scheme fees, and the merchant discount rate?
5. Where does an acquiring platform sit, and what are its main subsystems?
6. How do card-present and card-not-present transactions differ for the platform?
7. What is single-message versus dual-message processing?
8. What is a BIN/IIN and how is it used?
9. What is ISO 8583 and why does it still dominate card payments?
10. How is an ISO 8583 message structured?
11. How do you decode the MTI?
12. How does the bitmap work?
13. What field encodings must a parser handle?
14. Which data elements should you know?
15. Is ISO 8583 one standard? How do you handle scheme dialects?
16. How are amounts and currencies represented?
17. What do response codes in DE39 tell you, and how should they be handled?
18. What are network management messages?
19. How are reversals represented?
20. What is the difference between a request and an advice?
21. How would you implement an ISO 8583 parser in Java, and what are the pitfalls?
22. Walk through the Auth–Clear–Settle lifecycle.
23. What does an authorization actually do?
24. What checks does the acquirer perform before sending an authorization to the scheme?
25. How does clearing work, and how is it matched back to the authorization?
26. How does settlement work?
27. What are pre-authorization, final authorization, incremental, and partial authorization?
28. What is the difference between an authorization reversal, a void, and a refund?
29. Why can the cleared amount differ from the authorized amount?
30. How does the chargeback lifecycle work?
31. How is interchange determined at clearing?
32. How are multi-currency transactions and DCC handled?
33. How do Visa and Mastercard differ in ways an engineer cares about?
34. How does an EMV chip transaction work, and what is in DE55?
35. What is liability shift?
36. How is a PIN protected end to end?
37. What is PCI DSS and what may you store?
38. What is tokenization, and how do PCI tokens differ from network tokens?
39. What are 3-D Secure and SCA, and where do they touch the switch?
40. How do HSMs fit into the transaction path, and what do they cost in latency?
41. How is fraud and risk screening done within a tight latency budget?
42. How do you keep card data out of logs, events, and debugging tools?
43. Describe a payment switch's architecture.
44. Why separate connection gateways from stateless processing?
45. How does routing work?
46. What is a canonical message model and why use one?
47. How do you correlate asynchronous responses to requests?
48. How do you manage scheme and issuer links?
49. What does the acquiring host manage besides routing?
50. What states does a transaction pass through?
51. What must be written synchronously on the authorization path?
52. How would you design the clearing and settlement engine?
53. What do you do when an authorization request times out?
54. How do you detect duplicate messages and keep processing idempotent?
55. What if a reversal arrives before the original transaction?
56. What is stand-in processing, and what does it mean for the acquirer?
57. How do you get "exactly-once" money movement?
58. How do you handle concurrent transactions on the same card or merchant?
59. How should retries, store-and-forward, and dead-letter handling work for advices and reversals?
60. What happens when the database, Kafka, or the HSM is unavailable?
61. How do you design multi-data-centre resilience for a payment switch?
62. How do you keep authorization state and the ledger consistent?
63. What is the latency budget for an authorization, and where does the time go?
64. What threading model suits a high-TPS switch in Java?
65. How do you manage GC and memory for low latency?
66. Where does Kafka fit in a payment platform, and where does it not?
67. How do you build reliable Kafka consumers for financial events?
68. How do you choose and shape the database for the hot path?
69. How do you apply backpressure and load shedding in payments?
70. How do you performance-test a switch?
71. What should you observe in a payment platform?
72. How would you design the ledger for an acquirer?
73. What is reconciliation and what are its types?
74. How should money be represented in Java?
75. How are merchant fees, reserves, and payouts calculated?
76. How do you process very large clearing and settlement files safely?
77. What regulatory and audit concerns affect the design?
78. How does certification with a scheme work, and why does it matter for design?
79. How would you migrate or replace a legacy switch without downtime?
80. Design an authorization engine for 150M+ users and high TPS. What are the key decisions?
81. Production scenario: the approval rate suddenly drops and `91`/`96` responses rise. What do you do?
82. How do you balance coding and architecture in a hands-on architect role?
83. How do you demonstrate genuine payment experience in an interview?


### Thirty-second summary

Payments are a four-party system where authorization is a real-time promise, clearing is the financial claim, and settlement is the transfer. ISO 8583 is a framework, not one wire format, so isolate each scheme dialect behind adapters and a canonical model. The hard engineering is in the unknowns: timeouts, late responses, duplicates, and reordering, handled with idempotency keys, reversals with store-and-forward, and explicit state machines. Keep the hot path small and durable, move everything else to an outbox and Kafka, represent money exactly, reconcile every stage against independent records, and measure approval rate, latency percentiles, and reversal backlog as first-class signals.

## Official references

- [ISO 8583 overview (ISO)](https://www.iso.org/standard/79451.html)
- [PCI Security Standards Council document library](https://www.pcisecuritystandards.org/document_library/)
- [EMVCo specifications](https://www.emvco.com/specifications/)
- [EMVCo 3-D Secure](https://www.emvco.com/emv-technologies/3-d-secure/)
- [Visa Developer and technical resources](https://developer.visa.com/)
- [Mastercard Developers](https://developer.mastercard.com/)
- [jPOS documentation](https://docs.jpos.org/)
- [Apache Kafka documentation](https://kafka.apache.org/documentation/)
- [ISO 4217 currency codes](https://www.iso.org/iso-4217-currency-codes.html)

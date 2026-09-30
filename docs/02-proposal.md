# 2. Proposal: Delegated Agent Identity (DAI)

## 2.1 Summary

A **Delegated Agent Identity (DAI)** is a sub-identity that a user creates
*inside their existing account* at a service (email provider, social network,
bank, card issuer, marketplace, ...). The user then hands the DAI to an AI
agent. The DAI:

- is **derived from** the user's account. It is not a separate "burner"
  account, so no second KYC or terms-of-service workaround is needed;
- carries **explicit, fine-grained permissions and limits** that the
  **service itself enforces**, not the agent or its prompt;
- is **clearly attributed**. Every action is recorded (and, where relevant,
  shown to third parties) as *"performed by Alice's agent <name>"*;
- is **short-lived, independently revocable and auditable**, just like a
  GitHub fine-grained PAT.

In short: *a fine-grained PAT for your whole digital life, with spending
limits.*

## 2.2 Design principles

1. **Enforce at the source.** The service holding the resource checks every
   limit. Client-side guardrails (prompts, skills) are still useful, but only
   as extra layers on top.
2. **Least privilege by default.** A new DAI starts with *no* permissions.
   Each capability has to be granted explicitly.
3. **Distinguishable from the human.** A DAI can never authenticate as the
   user. It has its own credential, its own identifier in logs, and it can't
   change the user's security settings.
4. **Bounded in time, value and scope.** Every grant has an expiry. Grants
   that authorize transactions (payments, transfers, purchases) always have
   a monetary cap.
5. **Human-in-the-loop escalation.** Actions above a threshold are sent back
   to the user for out-of-band approval instead of being silently allowed or
   rejected.
6. **Revocable instantly and on its own.** Revoking a DAI doesn't affect the
   user's own sessions or any other DAI.
7. **Built on existing standards.** Where possible, reuse OAuth 2.x, token
   exchange, rich authorization requests, proof-of-possession and verifiable
   credentials instead of inventing new protocols (see
   [03-prior-art.md](03-prior-art.md)).
8. **Portable vocabulary.** Services in the same category (email, social,
   banking, cards) should use a shared set of scope names, so users and
   agents meet the same model everywhere.

## 2.3 Concepts

| Concept | Description |
| --- | --- |
| **Principal** | The human user who owns the account and is accountable for the delegation. |
| **Delegated Agent Identity (DAI)** | A named sub-identity under the principal's account, for example `alice/agents/travel-assistant`. |
| **Agent** | The software that holds and uses the DAI credential. It may be operated by a third-party vendor. |
| **Grant** | The set of permissions, constraints and limits attached to a DAI. A DAI may have several grants, for example one per resource. |
| **Constraint** | A limit on a grant: amount, rate, time window, allow-list of recipients or merchants, content rules, and so on. |
| **Approval policy** | Rules that decide when an action needs real-time confirmation from the principal. |
| **Enforcement point** | The service's API or gateway that checks every agent request against the grant. |
| **Audit trail** | An immutable log of every DAI action, visible to the principal. |

```
 ┌────────────┐  creates & scopes   ┌──────────────────────────────┐
 │  Principal │ ──────────────────▶ │ Service (bank / mail / social)│
 │   (Alice)  │ ◀── approvals ───── │  ┌────────────────────────┐   │
 └────────────┘      audit          │  │ DAI: travel-assistant  │   │
        │                           │  │  grants + constraints  │   │
        │ hands DAI credential      │  └───────────┬────────────┘   │
        ▼                           │     Enforcement point         │
 ┌────────────┐  requests (signed   │   (checks every request)      │
 │   Agent    │ ──── with DAI ────▶ │                               │
 └────────────┘      credential)    └──────────────────────────────┘
```

## 2.4 Access levels by domain

The tables below suggest a **shared vocabulary** of capability levels. A
service may offer finer options, but it should map them onto these levels so
that users can compare services easily.

The levels are numbered only as a rough ordering from least to most
privileged; **they are independent capabilities, not cumulative tiers**.
Granting `mail.organize` does not implicitly grant `mail.send`, and each
capability listed in a grant must be named explicitly (as in the example
grant in §2.6).

### 2.4.1 Email

| Level | Scope (suggested) | Allows | Typical constraints |
| --- | --- | --- | --- |
| 0 | `mail.none` | Nothing | – |
| 1 | `mail.metadata.read` | Read headers, subjects and labels. No bodies. | Folders/labels, date range |
| 2 | `mail.read` | Read full messages and attachments | Folders/labels, senders, exclude "sensitive" labels |
| 3 | `mail.draft` | Create drafts that the user reviews and sends | – |
| 4 | `mail.reply` | Reply only within existing threads | Allowed domains/contacts, max N/day, required "sent by agent" footer or header |
| 5 | `mail.send` | Start new conversations | Recipient allow-list, max recipients, rate limit, approval for new recipients |
| 6 | `mail.organize` | Label, archive, mark read | – |
| 7 | `mail.delete` | Move to trash (recoverable) | Never permanent delete. Retention window. |

**Always excluded from DAIs:** changing forwarding rules, filters that forward
externally, recovery email/phone, password, MFA, and account deletion. These
are common ways attackers take over accounts, so an agent must never be able
to do them.

### 2.4.2 Social media (Instagram, X, LinkedIn, Facebook, ...)

| Level | Scope (suggested) | Allows | Typical constraints |
| --- | --- | --- | --- |
| 0 | `social.none` | Nothing | – |
| 1 | `social.read` | Read feed, own posts, notifications | – |
| 2 | `social.dm.read` | Read direct messages | Specific conversations only |
| 3 | `social.engage` | Like, follow or unfollow | Rate limit |
| 4 | `social.reply` | Comment or reply to DMs | Only on own posts, or only to existing contacts. Rate limit. |
| 5 | `social.draft` | Schedule posts for human approval | – |
| 6 | `social.publish` | Publish posts or stories | Rate limit, content policy, mandatory "AI-assisted" label, approval queue |

**Always excluded:** changing profile identity (name, handle, photo), account
settings, deleting the account, and monetization or payout settings.

### 2.4.3 Bank accounts

| Level | Scope (suggested) | Allows | Typical constraints |
| --- | --- | --- | --- |
| 0 | `bank.none` | Nothing | – |
| 1 | `bank.balance.read` | Read balances | Specific accounts |
| 2 | `bank.transactions.read` | Read transaction history | Date range, specific accounts |
| 3 | `bank.payment.prepare` | Prepare a payment that the user must approve | – |
| 4 | `bank.payment.known` | Pay **existing** beneficiaries only | Per-transaction cap, daily/monthly cap, specific beneficiaries |
| 5 | `bank.payment.new` | Pay new beneficiaries | Low cap, mandatory approval above threshold, cooling-off period |

**Always excluded:** adding or removing account holders, changing contact
details, raising the user's own limits, opening credit products, and
closing accounts.

**Requires explicit per-action approval (never silently allowed):**
international transfers above a regulatory threshold.

### 2.4.4 Credit / debit cards

This part is modelled on virtual cards and network tokenization (see
[prior art](03-prior-art.md#35-payments)).

| Level | Scope (suggested) | Allows | Typical constraints |
| --- | --- | --- | --- |
| 0 | `card.none` | Nothing | – |
| 1 | `card.statements.read` | Read card transactions | – |
| 2 | `card.pay.single_use` | One purchase with a one-time token | Exact amount or max amount, merchant, expiry in minutes or hours |
| 3 | `card.pay.merchant_locked` | Repeated purchases at one merchant | Monthly cap, merchant ID |
| 4 | `card.pay.category` | Purchases in given merchant categories (MCC) | Per-transaction and monthly caps, allowed MCCs, countries |
| 5 | `card.pay.general` | Any purchase | **Credit limit lower than the user's**, approval above threshold |

Every card-level DAI should get **its own agent-bound token** (a network
token or virtual card number). The user's real card number (PAN) is never
shared. Disputes and chargebacks can then tell the issuer that the
transaction was *agent-initiated*.

### 2.4.5 Other services (ride-hailing, travel, shopping, calendars)

The same pattern applies. For example:

- **Ride-hailing (e.g. Uber):** `ride.request` limited to saved locations,
  a max fare, and specific hours.
- **Calendar:** `calendar.freebusy.read`, `calendar.event.propose`,
  `calendar.event.write`.
- **Marketplace:** `order.create` with a max order value and a category
  allow-list, and `order.cancel`.

## 2.5 Constraint types

Every grant can combine these constraint types:

| Constraint | Example |
| --- | --- |
| **Monetary cap** | ≤ €50 per transaction, ≤ €300 per month |
| **Rate limit** | ≤ 20 emails per day, ≤ 3 posts per week |
| **Time window** | Valid 2026-10-01 → 2026-10-15; only on weekdays 08:00–20:00 |
| **Counterparty allow-list** | Only reply to `@company.com`; only pay beneficiary "Landlord" |
| **Category allow-list** | Card MCCs: travel, transport |
| **Geography** | Only merchants in EU |
| **Data filter** | Exclude emails labelled `medical` or `legal` |
| **Approval threshold** | Anything > €100, or any new recipient, needs a push approval |
| **Disclosure requirement** | Outgoing messages carry a "Sent by Alice's assistant" marker |
| **Max lifetime** | DAI expires after at most 90 days unless the user renews it |

## 2.6 Example grant (illustrative, not a specification)

```json
{
  "dai": "alice/agents/travel-assistant",
  "principal": "alice",
  "agent": {
    "name": "Travel Assistant",
    "vendor": "example-agent-vendor.com",
    "key_thumbprint": "sha256:…"
  },
  "expires_at": "2026-10-31T23:59:59Z",
  "grants": [
    {
      "resource": "mail",
      "scopes": ["mail.read", "mail.reply"],
      "constraints": {
        "labels": ["Travel"],
        "reply_domains": ["*.airline.example", "*.hotel.example"],
        "max_messages_per_day": 10,
        "disclosure": "footer"
      }
    },
    {
      "resource": "card:visa-1234",
      "scopes": ["card.pay.category"],
      "constraints": {
        "currency": "EUR",
        "max_per_transaction": 400,
        "max_per_month": 1200,
        "mcc_allow": ["3000-3299", "4111", "4121", "7011"],
        "approval_above": 150
      }
    }
  ]
}
```

## 2.7 Lifecycle

1. **Create.** The user opens *Settings → Agent access* at the service (or
   the agent starts a consent flow) and names the DAI, picks levels and
   constraints, and sets an expiry. For high-risk levels, the service requires
   step-up authentication, for example a passkey.
2. **Bind.** The DAI credential is bound to a key held by the agent
   (proof-of-possession). A leaked bearer token is then useless on its own.
3. **Use.** The agent calls the service's API. The enforcement point checks
   the scope and constraints on **every** request and logs the result.
4. **Escalate.** If an action goes over an approval threshold, the service
   pauses it and sends the user an out-of-band approval request (push
   notification or banking app). The user sees exactly what will happen
   (amount, recipient, content) and approves or declines.
5. **Observe.** The user has a dashboard per DAI with recent actions, amounts
   spent against caps, and anomalies.
6. **Revoke / expire.** One click (or automatic expiry) invalidates the DAI
   right away. Pending actions are cancelled.
7. **Dispute.** Financial actions tagged as agent-initiated go through a
   defined dispute path (see [caveats](04-caveats.md)).

## 2.8 How it maps onto existing protocols

The proposal doesn't need a brand-new protocol. A reasonable technical base
is:

| Need | Candidate building block |
| --- | --- |
| Delegated authorization | OAuth 2.1 authorization code + PKCE |
| "Agent acting for user" in tokens | OAuth 2.0 Token Exchange (RFC 8693) `act` (actor) claim; IETF drafts on AI-agent on-behalf-of flows |
| Structured, fine-grained permissions (amounts, merchants) | Rich Authorization Requests (RFC 9396) `authorization_details` |
| Binding tokens to the agent's key | DPoP (RFC 9449) or mTLS-bound tokens (RFC 8705) |
| Out-of-band user approval | Client-Initiated Backchannel Authentication (OpenID CIBA) |
| Agent identity itself | Workload identity (SPIFFE / IETF WIMSE), or provider-issued agent IDs |
| Portable, verifiable mandates | W3C Verifiable Credentials, payment mandates (e.g. AP2) |
| Instant revocation | Token revocation (RFC 7009), Shared Signals / CAEP events |
| Tool-level agent connections | Model Context Protocol (MCP) authorization, which is based on OAuth 2.1 |

## 2.9 What each party must do

| Party | Responsibilities |
| --- | --- |
| **Services** (the resource server, and usually the authorization server, in OAuth terms) | Offer a DAI concept in account settings and APIs. Enforce scopes and constraints server-side. Label agent actions. Provide audit, revocation and approvals. |
| **Agent vendors** | Request only the minimum grant. Keep keys in secure storage. Never ask for the user's primary credentials. Show the user what they are delegating. |
| **Users** | Pick levels deliberately, review audit logs, revoke unused DAIs. |
| **Standards bodies / regulators** | Agree on a shared scope vocabulary per sector. Define liability and dispute rules for agent-initiated transactions. |

## 2.10 Non-goals

- Defining *how* an agent decides what to do. This proposal is only about
  what it is **allowed** to do.
- Replacing client-side guardrails. Those remain useful as extra layers of
  defense.
- Specifying a single global identity provider. DAIs are issued by each
  service, or by a federated IdP the service trusts.

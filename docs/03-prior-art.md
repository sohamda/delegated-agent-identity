# 3. Prior Art: What Already Exists

This document looks at what identity providers, platforms, payment networks
and standards bodies already offer that comes close to *delegated agent
identity*. It also points out what is still missing.

> ⚠️ This area is changing quickly. Product names, features and draft status
> below reflect public announcements at the time of writing. Check the linked
> sources before relying on any detail.

## 3.1 Standards and protocols

| Standard | What it provides | Relevance / gap |
| --- | --- | --- |
| **OAuth 2.0 / OAuth 2.1** ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749), [OAuth 2.1 draft](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/)) | Scoped, revocable access tokens that a user grants to a client app. | This is the foundation. But scopes are coarse and defined by each provider, and there is no standard notion of "agent" or spending limits. |
| **OAuth 2.0 Token Exchange** ([RFC 8693](https://www.rfc-editor.org/rfc/rfc8693)) | `act` (actor) claim that expresses "*X acting on behalf of Y*". Supports delegation chains. | This directly models *agent acting for user*. It is not yet widely used in consumer services. |
| **Rich Authorization Requests (RAR)** ([RFC 9396](https://www.rfc-editor.org/rfc/rfc9396)) | Structured `authorization_details` (for example payment amount, creditor, account). | Created for open banking. It is a natural format for amount, merchant and recipient constraints. |
| **DPoP** ([RFC 9449](https://www.rfc-editor.org/rfc/rfc9449)) / **mTLS tokens** ([RFC 8705](https://www.rfc-editor.org/rfc/rfc8705)) | Bind tokens to a key held by the client. | Stops a leaked agent token from being replayed by someone else. |
| **OpenID CIBA** ([spec](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html)) | The client asks the user for approval out-of-band, for example with a push notification. | This is exactly the "human-in-the-loop above threshold" pattern. |
| **GNAP** ([RFC 9635](https://www.rfc-editor.org/rfc/rfc9635)) | Next-generation grant negotiation with rich access requests and key-bound clients. | Well suited to agents in theory, but little adoption so far. |
| **UMA 2.0** ([Kantara](https://docs.kantarainitiative.org/uma/wg/rec-oauth-uma-grant-2.0.html)) | User-managed access: the resource owner sets policies for who may access what. | Covers user-set policies well, but consumer adoption is limited. |
| **IETF draft: OAuth for AI agents on behalf of users** ([draft-oauth-ai-agents-on-behalf-of-user](https://datatracker.ietf.org/doc/draft-oauth-ai-agents-on-behalf-of-user/)) | Adds `requested_actor` / `actor_token` so the user consents to a *specific agent*, and tokens record the user → client → agent chain. | Very close to this proposal on the authorization side. It doesn't define sector vocabularies or spending limits. Expired individual draft (`-02` expired 2026-02-27, not adopted by an IETF working group). |
| **IETF WIMSE** ([WG](https://datatracker.ietf.org/wg/wimse/about/)) and AI-agent auth drafts (for example `draft-klrc-aiagent-auth`, currently expired) | Workload identity and ways to combine workload, agent and user identity. | Covers **agent identity**. Still needs **user delegation** added on top. |
| **Model Context Protocol (MCP) authorization** ([spec](https://modelcontextprotocol.io/specification/latest/basic/authorization)) | MCP servers act as OAuth 2.1 resource servers. Agents get scoped tokens per tool server. | This is the most common way agents connect to tools today. Scope design is left to each server. |
| **W3C Verifiable Credentials** ([VC Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/)) | Signed, portable claims. | Useful for portable "mandates" that any party can verify (see AP2 below). |
| **Shared Signals / CAEP** ([OpenID SSF](https://openid.net/wg/sharedsignals/)) | Real-time security events such as revocation and session changes. | Can spread DAI revocation quickly across services. |

## 3.2 Identity providers (workforce and customer identity)

| Provider / product | What it offers for agents | Gap vs. this proposal |
| --- | --- | --- |
| **Microsoft Entra Agent ID** ([announcement](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/announcing-microsoft-entra-agent-id-secure-and-manage-your-ai-agents/3827392)) | Gives AI agents (Copilot Studio, Azure AI Foundry, ...) their own directory identities, so admins can apply conditional access, lifecycle management and auditing to them. | Built for **enterprise** tenants and admin-managed. It is not a consumer feature where a person delegates *their own* bank or social account. |
| **Okta, Cross App Access (XAA) / Auth for GenAI** ([Okta](https://www.okta.com/)) | The enterprise IdP controls which agent or app may get tokens to which other app on the user's behalf (the *Identity Assertion Authorization Grant* draft). | Enterprise-focused. The IdP brokers access but doesn't set per-action financial limits. |
| **Auth0 for AI Agents** ([Auth0](https://auth0.com/ai)) | *Token Vault* (stores and refreshes third-party OAuth tokens for agents), *asynchronous authorization* with CIBA for human approval, and fine-grained authorization (FGA) for data access. | Very close in developer experience. But it relies on the **downstream service's** scopes, so it can't enforce limits the downstream service doesn't support. |
| **Google Cloud / Workspace** | OAuth granular consent. Service-account impersonation. Workspace **domain-wide delegation** (admin-granted). Gmail **mail delegation** (a human delegate can read, send and delete on your behalf). | Delegation exists but is either all-or-nothing (mail delegate) or admin-only (domain-wide). Nothing is specific to agents. |
| **AWS IAM / STS**, **Amazon Bedrock AgentCore Identity** | Roles with short-lived credentials and *session policies* that can only narrow permissions. AgentCore Identity manages agent identities and outbound OAuth credentials. | Session policies are a strong model for "delegate with less power". This only applies to cloud resources. |
| **GitHub fine-grained PATs / GitHub Apps** ([docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)) | Per-repository, per-permission, expiring, revocable tokens. Actions are attributed to the app or token. | This is the **reference UX** this proposal wants to bring to consumer services. |
| **Other CIAM vendors** (e.g. Descope, Stytch, WorkOS) | Agent-oriented OAuth / MCP authorization servers, consent screens and connected-app management. | Useful tooling for services that want to *implement* DAIs. |

## 3.3 Email and social platforms

| Platform | Existing mechanism | Gap |
| --- | --- | --- |
| **Gmail** | OAuth scopes such as `gmail.readonly`, `gmail.compose`, `gmail.send`, `gmail.modify`, `gmail.metadata`. Mail delegation. | The scopes are fairly granular, but there are **no constraints** (recipient allow-lists, rate caps, label filters), no "sent by agent" marking and no approvals. |
| **Microsoft 365 / Outlook** | Microsoft Graph `Mail.Read`, `Mail.ReadWrite`, `Mail.Send`. Exchange *Send on Behalf* and delegate access. Exchange application access policies (for admins). | "Send on behalf of" shows the delegate to recipients, which is a good precedent for disclosure. Consumer-level constraints are missing. |
| **Meta (Facebook / Instagram)** | Business portfolio roles and Page *tasks* (content, messages, community activity, ads, insights). Instagram Graph API permissions (`instagram_basic`, `instagram_content_publish`, `instagram_manage_comments`, `instagram_manage_messages`). | Role-based delegation exists for **business** accounts. Personal accounts can't create scoped agent access. |
| **X (Twitter)** | OAuth 2.0 scopes (`tweet.read`, `tweet.write`, `dm.read`, `dm.write`, `like.write`, ...). Automated-account label. | The **"Automated" account label** is a precedent for disclosing agent activity. It applies to separate bot accounts, not to delegated access. |
| **LinkedIn** | OAuth scopes (e.g. `w_member_social`). Page admin roles. | Coarse scopes and no constraints. |

## 3.4 Banking

| Mechanism | What it offers | Gap |
| --- | --- | --- |
| **PSD2 / Open Banking (EU, UK)** | Regulated third parties: **AISPs** (read accounts) and **PISPs** (initiate payments), with explicit user consent, strong customer authentication (SCA) and consent expiry or re-authentication. | This is the closest regulatory precedent: *delegated, scoped, revocable access to your bank without sharing credentials*. But access is granted to licensed **providers**, not to a user's personal agent, and agent-specific attribution is missing. |
| **UK Variable Recurring Payments (VRP)** ([Open Banking UK](https://www.openbanking.org.uk/variable-recurring-payments-vrps/)) | A long-lived payment mandate with **per-payment and per-period limits** that the user sets and the bank enforces. | This is **essentially a financial DAI grant**. It could be reused directly for agents. |
| **Brazil Open Finance, US CFPB §1033 rule, Australia CDR** | Consent-based data sharing and (in some cases) payment initiation. | Still focused on data portability. Agent-specific rules are emerging. **Status note:** the US CFPB §1033 rule is not currently an operative example — a federal court stayed its compliance deadlines on 2025-10-29 and the CFPB is reconsidering the rule, so it should not be treated as an available mechanism today. |
| **Secondary account users / power of attorney** | Banks let a second person access an account with limited rights. | Meant for humans, paper-heavy and not API-driven. |

## 3.5 Payments

| Product / protocol | What it offers | Relevance |
| --- | --- | --- |
| **Visa Intelligent Commerce** ([Visa](https://corporate.visa.com/en/products/intelligent-commerce.html)) | Agent-specific **network tokens** that replace the card number, user-set spending controls, and signals that identify agent-initiated transactions. Visa also published a *Trusted Agent Protocol* for merchants to recognize legitimate agents. | This is a direct industry implementation of `card.pay.*` levels. |
| **Mastercard Agent Pay** ([Mastercard](https://www.mastercard.com/news/press/2025/april/mastercard-unveils-agent-pay-pioneering-agentic-payments-technology-to-power-commerce-in-the-age-of-ai/)) | **Agentic tokens** that are registered to a specific agent and bound by user-defined rules. | Same idea as Visa, on a different network. |
| **Agent Payments Protocol (AP2)** ([Google](https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol)) | An open protocol that uses signed **mandates** (verifiable credentials): an *Intent Mandate* (what the user authorized, with limits) and a *Cart Mandate* (the exact purchase). This gives a non-repudiable audit trail. | Very close to the "grant + approval" model. It can be used for portable, verifiable delegation. |
| **Agentic Commerce Protocol (ACP)** (OpenAI & Stripe, [docs](https://docs.stripe.com/agentic-commerce)) | A **Shared Payment Token** scoped to a specific merchant, amount and time window, so the agent never sees the card details. | Implements `card.pay.single_use` / `card.pay.merchant_locked`. |
| **Virtual / disposable cards** (Stripe Issuing, Privacy.com, Revolut, Capital One virtual numbers, ...) | Per-merchant or single-use card numbers with spending limits, available today. | A **practical workaround you can use now**: give the agent a capped, merchant-locked virtual card instead of your real card. |
| **Corporate spend management** (Ramp, Brex, Amex employee cards) | Per-cardholder limits, MCC restrictions and approval workflows. | A proven UX for "less credit than the account owner". |
| **Family / business profiles** (Uber for Business, Uber Family, Apple/Google family purchase approvals) | Sub-users with spending caps and approval requests ("Ask to Buy"). | A consumer precedent for approval thresholds. |

## 3.6 Gap analysis

| Capability | Exists today? | Where |
| --- | --- | --- |
| Scoped, revocable, user-granted tokens | ✅ widely | OAuth everywhere |
| Explicit *agent* identity (distinct from app/user) | 🟡 emerging | Entra Agent ID, IETF drafts, Visa/Mastercard agent tokens |
| "Acting on behalf of" in tokens | 🟡 standardized, rarely used | RFC 8693 `act` claim |
| Spending limits for delegated payments | 🟡 in payments only | VRP, virtual cards, Visa/Mastercard/ACP/AP2 |
| Constraints for email/social (recipient lists, rate caps, content disclosure) | ❌ mostly missing | – |
| Human-in-the-loop escalation | 🟡 available as a building block | CIBA, Auth0 async auth, "Ask to Buy" |
| Consumer self-service "create an agent identity" in account settings | ❌ missing | – |
| Shared cross-service vocabulary of agent access levels | ❌ missing | – |
| Clear liability rules for agent-initiated actions | ❌ missing | – |

**Conclusion.** Most of the building blocks already exist. Payments are
furthest along, enterprise identity is second, and consumer email and social
platforms are furthest behind. What is still missing is:

1. a **consumer-facing, self-service DAI** at every service,
2. **constraint types beyond scopes** for non-financial actions, and
3. a **shared vocabulary and liability framework** across sectors.

This proposal ([02-proposal.md](02-proposal.md)) aims to fill those gaps.

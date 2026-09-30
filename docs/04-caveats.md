# 4. Caveats, Risks and Open Questions

Delegated Agent Identities (DAIs) reduce risk, but they don't remove it. This
document lists the known limitations of the proposal so that they can be
discussed openly.

## 4.1 Security caveats

1. **Scopes don't prevent misuse *inside* the scope.** An agent with
   `mail.reply` can still send a harmful reply to an allowed contact. An
   agent with a €300/month card budget can still waste €300. DAIs limit how
   bad the damage can get. They don't guarantee good behaviour.
2. **Prompt injection still works, just with a smaller impact.** A malicious
   email or web page can still steer the agent. Source enforcement only
   limits what the hijacked agent can do. For example, a `mail.read` +
   `mail.send` combination is still an exfiltration channel. Services should
   warn about "toxic combinations" of read-sensitive plus write-external
   permissions.
3. **Credential theft moves to the agent.** The DAI credential becomes a
   valuable target. It needs proof-of-possession binding, short lifetimes and
   secure key storage. Otherwise it is just another password.
4. **Approval fatigue.** If too many actions need approval, users will
   approve without reading, which is the same problem as MFA-push fatigue.
   Thresholds must be tuned carefully, and approval prompts must show clearly
   what will happen.
5. **Confused deputy and delegation chains.** Agents may call sub-agents or
   other services. Each hop must *narrow* permissions, never widen them, and
   the full chain must stay visible (RFC 8693 `act` nesting). Many
   implementations get this wrong.
6. **Splitting large actions to stay under limits.** An agent (or attacker)
   can break one large action into many small ones to stay under
   per-transaction thresholds. Limits need to cover time periods and total
   amounts, not only single transactions.
7. **Social engineering at creation time.** Attackers can trick users into
   creating an over-privileged DAI ("click here to connect your AI
   assistant"), just like OAuth consent phishing today. Consent screens have
   to be clear, and high-risk levels need step-up authentication.
8. **Account-takeover paths must stay out of reach.** If a DAI can change
   forwarding rules, recovery methods or security settings, all other limits
   are meaningless. These must be excluded, with no exceptions.

## 4.2 Adoption and ecosystem caveats

1. **The services have to build it.** This proposal only works if Gmail,
   Instagram, banks and card issuers implement DAIs themselves. They may not
   see a business case, or they may prefer their own proprietary agents
   (walled gardens).
2. **Fragmentation.** Without a shared vocabulary every service will invent
   its own levels, and we are back to the current situation. Standardization
   (IETF, OpenID Foundation, FIDO, EMVCo, Open Banking bodies) takes years.
3. **Legacy and API-less services.** Many services have no public API, so
   agents use browser automation with the user's session. DAIs can't protect
   anything that is accessed by scraping the full user session. Services
   would need to detect and block that, or offer an agent-grade API instead.
4. **Terms of service conflicts.** Many platforms currently *ban* automated
   access. They would have to change their terms to allow agents with DAIs,
   while still blocking abusive bots.
5. **The UX is hard.** Users already struggle with OAuth consent screens.
   Rich constraints (MCC codes, allow-lists, rate limits) need sensible
   presets (e.g. "Shopping assistant – €100/month") or users will either give
   up or pick "allow all".
6. **Agent vendor lock-in.** If DAIs are only issued through a few big
   identity brokers, those brokers become gatekeepers.

## 4.3 Legal, financial and regulatory caveats

1. **Liability for agent-initiated transactions is undefined.** Consumer
   protection rules (for example PSD2 unauthorized-transaction rules, or US
   Regulation E / Z) assume a human either authorized a payment or didn't.
   An agent acting *within* its limits but *against* the user's intent is a
   new category: authorized, but not what the user wanted. Chargeback and
   dispute rules need updating.
2. **KYC / AML.** Banks must know who initiates payments. A DAI stays tied to
   the verified principal, which helps. But regulators may still require
   additional disclosure of the agent operator, or limits on autonomous
   payments.
3. **Strong Customer Authentication (SCA).** PSD2 requires SCA for many
   payments. It is still an open question whether a pre-authorized DAI
   mandate counts as SCA (like VRP or merchant-initiated transactions do) or
   whether each payment needs fresh authentication.
4. **Credit implications.** Giving an agent "credit with a lower limit" may
   legally count as issuing a new credit line or an authorized-user card,
   which comes with its own regulatory requirements.
5. **Privacy and data protection.** `mail.read` exposes third parties'
   personal data (the people who emailed you) to an agent vendor. Under GDPR
   and similar laws, it is not always clear who is the controller or
   processor, and who handles data subject rights for the agent vendor.
6. **Disclosure obligations.** Some jurisdictions (for example the EU AI Act
   transparency rules) may require that AI-generated content or AI
   interactions be disclosed. DAIs make this possible through markers such as
   "Sent by Alice's assistant", but the exact rules differ by jurisdiction.
7. **Legal agency.** In many legal systems, a contract concluded by an
   automated agent binds the principal. Users need to understand that "my
   agent did it" is not a defense.

## 4.4 Social and platform-integrity caveats

1. **Authenticity on social media.** Letting agents post and reply on behalf
   of real people at scale could flood platforms with low-quality or
   deceptive content. Mandatory disclosure labels help, but they may be
   stigmatized or stripped away.
2. **Bot detection conflicts.** Platforms spend a lot on detecting automated
   behaviour. They would need to tell legitimate DAI traffic apart from
   abusive automation, which is a new trust tier.
3. **Recipients' expectations.** People emailing or messaging you may not
   want their messages read or answered by an AI. Recipient-side controls
   (for example "don't let agents process my messages") are an open question.

## 4.5 Technical open questions

- Should DAIs be issued **per service** (each bank issues its own) or by a
  **central, federated agent-identity provider** that services trust? Or a
  mix of both?
- How are constraints expressed so that they are both **machine-enforceable
  and human-readable**? Possible options: RAR `authorization_details`, a
  policy language (Cedar, OPA/Rego), or verifiable-credential mandates.
- How is **revocation spread** quickly across services and delegation chains
  (Shared Signals / CAEP)?
- How do services **identify the agent software** in a trustworthy way
  (attestation, signed agent manifests, Visa's Trusted Agent Protocol)
  rather than just the credential?
- What is the **minimum audit record** every service must keep and show to
  the user?
- How should **multi-user or shared accounts** (joint bank accounts, shared
  inboxes) handle DAIs? Does every account holder have to approve?

## 4.6 What users can do today (interim mitigations)

Until services offer real DAIs, these existing tools get closest to
source-side enforcement:

| Need | Interim approach |
| --- | --- |
| Payments | Give the agent a **virtual card** (single-use or merchant-locked, with a low limit), or use agent-specific tokens (Visa Intelligent Commerce, Mastercard Agent Pay, Stripe/ACP shared payment tokens) where available. Never share the real card number. |
| Banking | Use **Open Banking** consents (read-only AISP, or VRP with limits) through a regulated provider instead of sharing bank logins. |
| Email | Grant OAuth with the **narrowest scope** (for example `gmail.readonly` for read-only agents, or `gmail.metadata` when bodies aren't needed) instead of app passwords. Note that `gmail.compose` is **not** draft-only — Google's scope docs let it create, read, update and **send** drafts — so it does not by itself keep a human in the loop; if you want that, gate sending through a separate review step (for example a client that only creates drafts and requires the user to hit *Send* in Gmail). Consider a separate mailbox with forwarding filters for the agent. |
| Social media | For business accounts, use platform **roles/tasks** (Meta Business). For personal accounts, prefer draft/schedule tools that need human publishing. |
| Everything | Keep client-side guardrails (prompts, skills, tool allow-lists) as an **extra** layer, review connected-app lists regularly, and revoke unused access. |

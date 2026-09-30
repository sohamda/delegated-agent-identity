# 1. Situation and Problem

## 1.1 The situation today

AI agents (browser agents, coding agents, personal assistants, and so on) are
increasingly asked to act **on behalf of** a person: triage and answer email,
post on social media, book a ride, pay a bill, or buy something online.

To do this, the agent has to authenticate to the service where the action
happens (Gmail, Instagram, Uber, a bank, a card issuer, ...). Right now users
generally have four options, and none of them are good enough on their own:

| Option | What it looks like | Why it is a problem |
| --- | --- | --- |
| **Share your own credentials** | Give the agent your password, session cookies, API key or card number. | The agent has *all* of your power. The service can't tell you apart from your agent. Revoking means changing your password or card. Your MFA gets bypassed, or you end up approving prompts you can't really check. |
| **Create a separate "burner" identity** | A second Gmail, Instagram or Uber account just for the agent. | This gives you a boundary, but it is a workaround. It often breaks terms of service (for example one-person-one-account rules and KYC). It doesn't connect to your real data, you have more accounts to secure, and banks and card issuers don't allow it at all. |
| **Restrict the agent in the prompt** | System prompts, custom skills, tool allow-lists or guardrails in the agent runtime. | The limits are **advisory** and live on the **client side**. Prompt injection, model mistakes, a compromised runtime or a misconfigured skill can all get around them. The service still sees a fully privileged session. |
| **Use existing source-side delegation** | Scoped OAuth tokens, Open Banking (AISP/PISP/VRP) consents, platform roles (Meta Business, Workspace domain-wide delegation), virtual or agent-bound card tokens (Visa/Mastercard/ACP). | These *do* enforce limits at the source and are the closest analogues to what this proposal describes. But they were not designed for agents: there is **no standardized agent identity**, **no rich per-grant constraints beyond coarse scopes** for non-financial actions (recipient allow-lists, rate caps, content disclosure), and **no consistent attribution** of an action to a specific agent. Coverage is also uneven — payments are furthest along, consumer email and social are furthest behind. |

## 1.2 The core problem

> **Access control for agents is enforced at the wrong place.**

The only party that can *guarantee* a limit is the **resource owner's
service**: the mail provider, the social network, the bank, the card network.
Today that service has no concept of *"this request comes from Alice's agent,
not Alice, and it may only do X, Y, Z up to limit L"*.

This causes these concrete problems:

1. **No consistent least privilege.** Where scoped OAuth exists, scopes are
   coarse and provider-specific: an agent that only needs to *read* email
   often also gets the ability to *send* and *delete*. Outside payments,
   there is no standard way to further constrain a scope with amount, rate,
   recipient or content limits.
2. **No attribution.** Logs, fraud systems and dispute processes see "Alice"
   and not "Alice's agent". Nobody can tell afterwards whether a human or an
   agent performed an action.
3. **No clean revocation across the board.** OAuth tokens can be revoked,
   but shared credentials, session cookies and card numbers can't — removing
   that kind of access means rotating the user's primary credentials, which
   breaks every other session and device.
4. **No standard.** Every service (if it offers anything at all) has its own
   mechanism: app passwords, API keys, OAuth apps, page roles, virtual cards,
   and so on. Agents and users can't rely on a consistent model.
5. **Financial risk is uncapped outside payments.** Virtual cards, VRP and
   agent-bound network tokens do allow per-delegate spending limits, but
   they are not universally available and don't cover bank transfers or
   non-financial actions. Ordinary card and bank credentials still have no
   per-delegate caps the user can set for an agent.
6. **Liability is unclear.** When an agent makes an unwanted purchase or sends
   an embarrassing message, it isn't clear whether the user, the agent vendor
   or the service is responsible, because the service usually never knew an
   agent was involved.

## 1.3 An analogy: GitHub fine-grained personal access tokens

GitHub already solves a small version of this problem for developer
automation. A **fine-grained personal access token (PAT)**:

- is created by the user and tied to their account,
- is limited to specific repositories and specific permissions
  (for example `contents: read`, `issues: write`),
- has a mandatory expiry,
- can be revoked on its own without touching the user's password,
- shows up in audit logs as the token and not only as the user.

What is missing is **the same idea, applied consistently to email, social
media, banking and payment cards**, and designed for autonomous agents: it
also needs spending limits, human-in-the-loop approvals and clear attribution
of agent actions.

## 1.4 Goal

Services should let a user create a **Delegated Agent Identity**: a
first-class, revocable sub-identity of their own account with explicitly
scoped, source-enforced permissions and limits. The next document describes
this in detail: [02-proposal.md](02-proposal.md).

<!--
  draft-agent-identity-oauth-extension.md
  OAuth 2.0 Extension: Agent Identity and Delegation for AI Agents
  Intended status: Standards Track
  Expires: 2025‑12‑13
-->

# OAuth 2.0 Extension: Agent Identity and Delegation for AI Agents

*Version 0.1 – **DRAFT** (2025‑06‑13)*  
*Target venue: IETF OAuth Working Group*  
*Author: Brandon Werner (Microsoft) — crowd‑edited with community input*

---

## Abstract

This document specifies an OAuth 2.0 extension that enables **autonomous software agents** (AI agents, bots, LLM‐based assistants, etc.) to obtain finely scoped, auditable authorization to act **on behalf of a human user**.  
The extension introduces two new parameters—`requested_actor` (authorization request) and `actor_token` (token request)—plus explicit JWT claims so that the resulting **delegated access token** carries both the end‑user (subject) and the agent (actor) identities.  
The design:

* Re‑uses the OAuth 2.0 Authorization Code flow (with PKCE)  
* Aligns with OAuth 2.0 Token Exchange (RFC 8693) claim syntax  
* Avoids new grant‑type proliferation  
* Supports dynamic provisioning, revocation, and governance of agent credentials  
* Preserves backward compatibility with existing OAuth/OIDC deployments

Adoption of this profile allows enterprises to enforce least‑privilege, create tamper‑proof audit trails, and revoke misbehaving agents without impacting the user’s own tokens.

---

## Status of This Memo

This Internet‑Draft is submitted for discussion in the IETF OAuth Working Group.  
It is **work in progress** and will expire on 13 December 2025.

Distribution is unlimited. Comments welcome via pull requests or issues in this GitHub repository.

---

## Table of Contents

1. [Introduction](#1-introduction)  
2. [Terminology](#2-terminology)  
3. [Design Goals & Requirements](#3-design-goals--requirements)  
4. [Solution Overview](#4-solution-overview)  
5. [Protocol Details](#5-protocol-details)  
   5.1. [Authorization Request Extension](#51-authorization-request-extension)  
   5.2. [Token Request Extension](#52-token-request-extension)  
   5.3. [Delegated Access Token Format](#53-delegated-access-token-format)  
6. [Agent Provisioning & Lifecycle](#6-agent-provisioning--lifecycle)  
7. [Security Considerations](#7-security-considerations)  
8. [Privacy Considerations](#8-privacy-considerations)  
9. [Compatibility with Existing Standards](#9-compatibility-with-existing-standards)  
10. [Conclusion](#10-conclusion)  
11. [Normative References](#11-normative-references)  
12. [Informative References](#12-informative-references)  
13. [Appendix A — Comparative Analysis](#appendix-a--comparative-analysis)  

---

## 1  Introduction

Modern AI assistants increasingly perform high‑impact tasks—drafting contracts, moving money, provisioning resources—under minimal human supervision.  
Traditional OAuth 2.0 / OpenID Connect flows treat an “application” as the only acting party, conflating:

* **Subject**: the human resource owner; and  
* **Actor**: the software making the call.

This conflation prevents:

* Unique, verifiable **agent identities**  
* Fine-grained, agent-specific scopes  
* Lifecycle controls and targeted revocation  
* Tamper-proof audit linking every request to both user **and** agent

Existing drafts tackle parts of the problem but either create new grant types for every flow or leave lifecycle governance undefined.  
This document synthesizes the most practical elements into a small, interoperable extension.

---

## 2  Terminology

| Term | Definition |
|------|------------|
| **Agent / Actor** | Autonomous software component (bot, LLM, script) that initiates API calls. |
| **User / Resource Owner** | Human end‑user whose resources are being accessed. |
| **Client** | OAuth client application orchestrating the flow. May embed or invoke an agent. |
| **Authorization Server (AS)** | Issues tokens after authenticating user and agent. |
| **Resource Server (RS)** | API that validates tokens and enforces scopes. |
| **Actor Token** | Credential proving the agent’s own identity. |
| **Delegated Access Token** | JWT access token containing both `sub` (user) and `act` (agent) claims. |

---

## 3  Design Goals & Requirements

1. **Backward compatible** — build on OAuth2/OIDC, no bespoke protocol.  
2. **No new grant types** — extend Authorization Code + PKCE and map cleanly to RFC 8693.  
3. **Explicit user consent** — UI must name the agent and requested scopes.  
4. **Distinct agent identity** — tokens carry an `act` claim separate from `sub`.  
5. **Least privilege** — scopes can be agent‑specific and tokens short‑lived.  
6. **Dynamic provisioning** — supports RFC 7591 dynamic client registration.  
7. **Revocation friendly** — user/admin can kill an agent without nuking user tokens.  
8. **Interoperable** — one agent identity works across vendors.  

---

## 4  Solution Overview

* **Front‑channel**: add `requested_actor` to the authorization request.  
* **Back‑channel**: add `actor_token` (+ optional `actor_token_type`) to the token request.  
* **AS binds** code ↔ user ↔ agent ↔ client.  
* **JWT access token** issued with:

  ```json
  {
    "sub": "alice@example.com",
    "azp": "client123",
    "scope": "calendar.add calendar.view",
    "act": { "sub": "agent-x-123" }
  }
  ```
* **RS** logs & enforces based on both identities.  
* **Lifecycle**: agent identities provisioned via static admin, dynamic registration, or federated trust; revocation handled via token revocation + credential invalidation.

---

## 5  Protocol Details

### 5.1  Authorization Request Extension

```
GET /authorize?
  response_type=code
  &client_id=12345
  &redirect_uri=https://app.example/cb
  &scope=calendar.add calendar.view
  &requested_actor=agent-x-123
  &state=af0ifjsldkj
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
```

* `requested_actor` **MUST** reference a registered agent ID.  
* AS authenticates user, validates agent, and displays consent:  
  *“Allow **Agent X** to add events to your calendar?”*

### 5.2  Token Request Extension

```
POST /token
grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https://app.example/cb
&code_verifier=dBjftJeZ4CVP-m ­ ryBCf2r2W ­ dX ­ bQ ­ f ­ le4z
&actor_token=eyJhbGciOiJSUzI1Ni...
&actor_token_type=urn:ietf:params:oauth:token-type:jwt
&client_id=12345
&client_secret=secret
```

* AS verifies `actor_token` → maps to same `requested_actor`.  
* On success, issues JWT access token (plus optional refresh / ID tokens).

### 5.3  Delegated Access Token Format

Minimal required claims:

| Claim | Value |
|-------|-------|
| `sub` | User identifier |
| `act.sub` | Agent identifier |
| `azp` | Client ID |
| `aud` | RS audience |
| `scope` | Space‑delimited scopes |
| `exp` | Expiry (short!) |

Additional agent metadata (e.g., version, assurance) MAY be included via nested fields in `act`.

---

## 6  Agent Provisioning & Lifecycle

* **Static admin registration** → service principal / client credentials.  
* **RFC 7591 dynamic registration** → per‑instance credentials, auto‑cleanup.  
* **Federated issuer** → external IdPs (e.g., OpenAI, Anthropic) mint actor tokens; AS trusts via OIDC federation.  
* Revocation:  
  * Token revocation (RFC 7009)  
  * Credential disable / cert revocation  
  * User consent withdrawal  

---

## 7  Security Considerations

* `actor_token` must be strongly validated (signature, issuer, expiry).  
* PKCE **mandatory** for public clients; recommended for all.  
* Short token lifetimes + optional sender‑constrained tokens (mTLS, DPoP).  
* Clear consent UX to avoid confusion & phishing.  
* Audit logs should record both `sub` and `act.sub`.  

---

## 8  Privacy Considerations

Agent IDs may be PII; treat access tokens and logs accordingly.  
Explicit delegation enables tighter data‑minimization—agents receive only the scopes they need.

---

## 9  Compatibility with Existing Standards

| Spec | Alignment |
|------|-----------|
| **RFC 6749** | Re‑uses Authorization Code flow. |
| **RFC 7636 (PKCE)** | Required. |
| **RFC 8693 (Token Exchange)** | Shares `act` claim and parameter naming; server MAY implement as internal exchange. |
| **RFC 7591 (Dynamic Reg)** | Optional mechanism to mint agent credentials. |
| **RFC 9068 (JWT AT)** | Recommended token profile. |

---

## 10  Conclusion

This extension turns AI agents into **first‑class, governable identities** inside the OAuth ecosystem—delivering accountability, least privilege, and fast revocation without inventing a new protocol.  
We invite implementers and the OAuth WG to review, prototype, and iterate toward a formal RFC.

---

## 11  Normative References

* RFC 2119 – Key words for Use in RFCs  
* RFC 6749 – OAuth 2.0 Framework  
* RFC 7636 – PKCE  
* RFC 7591 – Dynamic Client Registration  
* RFC 8693 – Token Exchange  
* RFC 9068 – JWT Access Token Profile  

## 12  Informative References

* Werner, B. B. et al., *Proposal: Establishing “Agent ID” as an Open Standard*, 2025  
* Senarath & Dissanayaka, *OAuth 2.0 On‑Behalf‑Of User Authorization for AI Agents*, 2025  

---

## Appendix A — Comparative Analysis

| Aspect | Microsoft “Agent ID” | WSO2 Draft (Senarath & Dissanayaka) | This Draft |
|--------|----------------------|--------------------------------------|------------|
| **Scope** | Identity, lifecycle, governance | Narrow OAuth flow | Combines flow **and** lifecycle hooks |
| **New Grant Types** | Potentially several | None (extends code flow) | None |
| **Lifecycle APIs** | Yes (directory, revocation) | Out of scope | Recommended patterns |
| **Token Format** | Delegation token (user+agent) | JWT with `act` claim | Same, using JWT AT profile |
| **Complexity** | High if fully implemented | Low | Moderate—incremental upgrade |
| **Inter‑vendor** | Emphasized | Not addressed | Emphasized |
| **Standards Path** | OpenID FND / IETF | IETF draft | IETF OAuth WG (this file) |

---


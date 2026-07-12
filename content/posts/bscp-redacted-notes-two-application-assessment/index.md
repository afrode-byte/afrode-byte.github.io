+++
title = "Redacted BSCP Notes: Two Applications, Multiple Attack Surfaces"
date = 2026-07-12
description = "A redacted overview of vulnerabilities identified across two web applications during BSCP-style testing."
tags = [
"BSCP",
"PortSwigger",
"Burp Suite",
"Web Application Security",
"CORS",
"HTTP Request Smuggling",
"Remote File Inclusion",
"Command Injection",
"Privilege Escalation"
]
+++

# Redacted BSCP Notes: Two Applications, Multiple Attack Surfaces

## Overview

The Burp Suite Certified Practitioner (BSCP) exam is designed to test practical web application security skills across realistic applications and vulnerability classes.

This post is intentionally redacted. It does not include exam-specific details, exact payloads, target-specific paths or step-by-step exploitation chains. Instead, it summarizes the types of issues I worked through and the methodology behind analyzing multiple attack surfaces.

The goal is to document the learning process without exposing sensitive exam content.

---

## Scope

The assessment involved two separate applications with different behaviors, controls and vulnerability patterns.

For clarity, I will refer to them as:

- **App 1**
- **App 2**

Each application required a different testing strategy. Some findings were straightforward once the right feature was identified, while others required chaining behavior, understanding application logic or carefully observing how requests were processed.

---

## Exam Objective Chain

PortSwigger's own BSCP exam guidance explains that the exam contains two applications, and each application can be completed in three stages:

1. Obtain access to a low-privileged account in the application.
2. Use that account to access the admin interface, usually by escalating privileges or compromising the administrator account.
3. Use the admin interface to read `/home/carlos/secret` from the server filesystem.

This structure applied to both applications. The important part was not treating each vulnerability as isolated. Each issue helped move the assessment closer to the next stage of the objective chain.

---

## App 1

### User Enumeration through Password Reset

The first issue involved user enumeration through the password reset flow.

Password reset functionality is a common place to look for subtle differences in application behavior. Even when the UI appears generic, the application may still leak whether an account exists through:

- response messages;
- timing differences;
- status codes;
- email delivery behavior;
- redirects or workflow differences.

In this case, the reset password flow exposed enough behavioral difference to distinguish valid users from invalid ones.

The impact depends on context, but user enumeration can support targeted attacks, credential stuffing, phishing and account takeover attempts when combined with other weaknesses.

In the context of the objective chain, this helped with the first goal: obtaining a valid low-privileged account to continue testing authenticated functionality.

---

### CORS Misconfiguration Leading to User Data Exposure

The second issue involved a CORS misconfiguration that allowed user data to be read from another origin.

CORS issues become dangerous when an application:

- reflects or trusts untrusted origins;
- allows credentials in cross-origin requests;
- exposes sensitive authenticated data;
- fails to restrict trusted origins properly.

In this scenario, the misconfiguration allowed an attacker-controlled origin to access sensitive user information from an authenticated victim session.

The important lesson was to test CORS behavior against authenticated endpoints, not only public API responses.

This finding supported the privilege escalation phase because exposed user data helped move from a low-privileged user context to Admin access.

---

### Remote File Inclusion

The third issue was a Remote File Inclusion (RFI) vulnerability.

The application accepted user-controlled input that influenced a file or resource loading behavior. With insufficient validation, this allowed an external resource to be included by the application.

RFI can be severe because it may lead to:

- server-side request behavior;
- sensitive data exposure;
- remote code execution in dangerous configurations;
- chaining with other server-side weaknesses.

The key was identifying where user-controlled input crossed from a normal application parameter into backend resource loading.

In the final stage of the chain, RFI was the path toward remote code execution and the objective of reading `/home/carlos/secret`.

---

## App 2

### HTTP Request Smuggling with Reflected XSS

One of the more interesting issues involved HTTP Request Smuggling behavior using a TE.CL-style discrepancy, combined with reflected XSS in the `User-Agent` header.

Request smuggling requires careful observation of how front-end and back-end servers interpret request boundaries differently. When a desynchronization condition exists, it may be possible to influence how another request is processed.

In this case, the request parsing behavior became more impactful because a reflected XSS sink existed in a header-derived value.

This reinforced an important point: request smuggling is often not only about the desync itself, but about what the desync can be chained with.

In this case, chaining the desync behavior with the reflected XSS helped complete the first stage of the exam objective chain: obtaining access to a low-privileged user account in the application.

---

### Privilege Escalation through Application Logic

Another issue involved privilege escalation caused by flawed application logic.

The vulnerable behavior was related to a password-related workflow and a cookie-controlled state. By understanding how the application trusted a specific cookie value and how the password flow handled privileged identities, it was possible to reach behavior that should have been restricted.

I would classify this as a business logic or application logic vulnerability rather than a classic injection issue.

The important part was not a single payload. The issue came from understanding:

- how the application tracked user state;
- which values were trusted by the server;
- how privileged identities were handled;
- how password-related functionality interacted with authorization checks.

Logic flaws like this are easy to miss because they often look like normal application behavior until the tester understands the workflow deeply.

In the exam objective chain, this issue mapped to the second stage: escalating privileges from a low-privileged user context to Admin access.

---

### Command Injection

The final issue involved command injection.

Command injection occurs when user-controlled input reaches an operating system command without safe handling. The impact is usually high because successful exploitation may allow command execution in the context of the application server.

The testing process focused on identifying where input influenced backend command execution and confirming the behavior safely.

The most important lesson was to look beyond obvious form fields. Command execution behavior can sometimes be hidden behind utility features, integrations, diagnostics, file operations or background processing.

---

## Methodology Notes

This type of assessment requires moving between different testing modes:

- mapping application functionality;
- looking for behavioral differences;
- testing authentication and authorization flows;
- inspecting headers, cookies and state management;
- reviewing API behavior;
- testing parser and protocol edge cases;
- validating impact without over-testing.

Burp Suite was central to the workflow, especially for request analysis, repeater-based testing, proxy history review and observing small differences between similar requests.

---

## Key Takeaways

- Authentication flows often expose more than login forms.
- CORS testing should include authenticated endpoints.
- Remote File Inclusion issues require understanding backend resource loading and execution context.
- HTTP Request Smuggling becomes more powerful when chained with another sink.
- Application logic flaws depend on workflow understanding, not just payloads.
- Command injection may appear in unexpected features.
- A strong methodology matters more than memorizing vulnerability names.

---

## References

- [PortSwigger - Burp Suite Certified Practitioner](https://portswigger.net/web-security/certification)
- [PortSwigger - BSCP Exam Hints and Guidance](https://portswigger.net/web-security/certification/exam-hints-and-guidance)
- [PortSwigger Web Security Academy - CORS](https://portswigger.net/web-security/cors)
- [PortSwigger Web Security Academy - HTTP Request Smuggling](https://portswigger.net/web-security/request-smuggling)
- [PortSwigger Web Security Academy - Cross-Site Scripting](https://portswigger.net/web-security/cross-site-scripting)
- [OWASP WSTG - Testing for Remote File Inclusion](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.2-Testing_for_Remote_File_Inclusion)
- [PortSwigger Web Security Academy - OS Command Injection](https://portswigger.net/web-security/os-command-injection)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

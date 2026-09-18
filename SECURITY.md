# Security Policy

This repository contains design documentation only. It holds no application source code,
credentials or data. Security reports about the documentation itself are rarely needed. If you
believe you have found a vulnerability in the **HOROOMART system**, please report it privately
as described below.

## Reporting a vulnerability

**Please do not open a public GitHub issue for security problems.**

Email **horooinnovations@gmail.com** with the subject line `Security: HOROOMART`, and include:

- a description of the issue and its potential impact
- the steps needed to reproduce it
- any proof-of-concept, logs or screenshots that help us confirm it
- whether you would like to be credited once it is resolved

We will acknowledge your report, investigate it, and keep you informed of progress until
it is resolved. Please give us a reasonable opportunity to fix the issue before disclosing it
publicly.

## Scope

In scope: authentication and session handling, authorization and tenant isolation, data exposure,
injection, and financial-integrity flaws in HOROOMART.

Out of scope: denial-of-service testing, social engineering, physical attacks, and findings that
depend on an already-compromised device or account.

**Do not test against any live HOROOMART deployment** without prior written permission. The
system runs real businesses' operations.

For how HOROOMART is secured, see [docs/SECURITY_ARCHITECTURE.md](docs/SECURITY_ARCHITECTURE.md).

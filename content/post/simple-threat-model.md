---
title: "15 Minute Threat Modelling"
date: 2025-12-11T10:28:06-05:00
draft: false
tags: ["security"]
---


##### Neglected Security
You don't have to look very hard to find an insecure product on ProductHunt. Small teams working on new products often neglect security. They don't have security expertise. They don't have the budget to hire that expertise. And they understandably want to prioritize iteration and speed. Threat modelling can feel so daunting and heavy that it gets deprioritized. After all, for a new product, market risk is what will kill the business, right?

Not necessarily. I think the mistake here is applying enterprise methodology to a tiny team, in much the same way that some devs will deploy a Kubernetes cluster and GraphQL for their personal blog that only their mom reads. You don't need a two week long process to identify and mitigate risks. You need fifteen focused minutes and a process to follow. The goal is to force you to think clearly about what you're building, how attackers will reach it, and what can go wrong.

##### 15 Minute Threat Model
Here is a minimal approach that works.

###### 1. Define the App in One Sentence
What exactly are you building? If you can't define it in one sentence, you don't understand it well enough to secure it. Something like:

> A multi-user CRUD API for managing tasks, with user authentication, role-based permissions, and a Postgres backend.

is sufficient. 


###### 2. Identify Your High Value Assets
A threat model isn't useful if everything is deemed critical. Pick a set of assets that genuinely matter. For a typical CRUD app, that means:
- User accounts  
- Authentication tokens or session cookies  
- The primary database  
- Privileged endpoints (admin or internal)  
- Logs that might contain sensitive data

If any of these are compromised, you are in trouble. Everything else is secondary.


###### 3. Identify Entry Points
How can attackers potentially access your application?
- Public HTTP endpoints
- Authentication and registration flows  
- File uploads (if you have them)  
- Admin or management endpoints  
- Cloud resources: buckets, object storage, security-group misconfigurations  
- CI and deployment hooks

Just enumerate where the outside world touches your system.


###### 4. Run Simplified STRIDE
For each category, write the risks in simple terms:
- **Spoofing:** Weak login checks, token leakage, session hijacking, missing MFA on admin accounts.
- **Tampering:** Users editing or deleting data they don’t own, insecure direct object references, overly permissive PATCH/PUT handlers.
- **Repudiation:** Missing audit logs or logs without enough context to tie actions to users.
- **Information Disclosure:** Verbose error messages, stack traces, overbroad serializers returning fields the UI never uses, misconfigured storage with public read access.
- **Denial of Service:** Unbounded queries, missing rate limits, endpoints that let users create runaway database load.
- **Elevation of Privilege:** No authorization checks beyond “user is authenticated,” admin endpoints structured like normal routes, forgotten test-only routes left in prod.

No diagrams, rituals, or incantations needed. A few bullet points for each SRIDE bucket is enough to reveal where you might be vulnerable.


###### 5. Define Controls
If you have fifteen minutes, spend most of the time deciding what prevents the worst failures. For CRUD apps, the same five show up every time:

1. Strong, well-configured authentication and session management.
2. Authorization checks on every read/write.
3. Input validation on all parameters, even “internal” IDs.
4. Least-privilege database access and tightly scoped, prepared queries.
5. Minimal, safe, structured logs with stable identifiers.

If these five are solid, you’ve removed most of the real risk.




###### 6. Define Attack Scenarios
Capture a few realistic, concrete attack paths:
- An attacker changes a resource ID in the request and reads another user’s data because the handler never checks ownership.  
- A session cookie leaks through an insecure flag, and the attacker takes over the user’s account.  
- A malicious file upload slips past weak validation and leads to execution or exfiltration.  
- An unprotected admin endpoint lets an authenticated user escalate privileges.

If you can’t describe your top attack scenarios, you aren't done yet.


###### 7. Summarize
The output of this process should be a one-pager:

**Overview:** One sentence.  
**Assets:** 3–5 items.  
**Entry Points:** 5–7 items.  
**Threats:** STRIDE bullets.  
**Controls:** Five core mitigations.  
**Scenarios:** Three or four realistic attack paths.

This is short enough to get read, and easy to action.

That's all you need to get started. Remove the ceremony and you're left with something that's fast and useful. It forces engineers to discover the most exploitable parts of the application, and it does it quickly enough that they’ll actually use it during development. Spending fifteen minutes finding vulnerabilities is a lot cheaper than the cost of an attacker exposing them to you - and your customers - in production.
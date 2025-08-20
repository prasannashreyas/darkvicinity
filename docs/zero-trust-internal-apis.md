---
title: Zero-Trust for Internal APIs — Short, Practical Notes
date: 2025-08-20
last_review: 2025-10-01
tags: [aws, zero-trust, api-gateway, privatelink, sigv4]
---

Idea in one line: Don’t trust the session. Make every request prove itself.

<img width="808" height="705" alt="image (1)" src="https://github.com/user-attachments/assets/ca3158c9-8cc9-4821-9326-c3dc7776e2fc" />

## What this setup looks like
- Private API Gateway (no public internet), reachable only via a VPC interface endpoint (PrivateLink).
- Identity on each call: either IAM + SigV4 (service→service) or a JWT checked by Cognito or a Lambda authorizer.
- Policies everywhere: API resource policy, VPC endpoint policy, caller’s IAM policy. Deny by default, allow minimal.

## Why API Gateway?
Central gate for routing, throttling, auth, and logging. One place to enforce rules and observe traffic.

## PrivateLink in one paragraph
Create a VPC interface endpoint for `execute-api`. DNS for your API resolves to private ENI IPs inside your VPC.
Traffic stays on the AWS backbone (no IGW/NAT). Lock down with security groups and an endpoint policy.

## Auth choices
- IAM + SigV4: best for service→service calls.
- Cognito authorizer: validates JWTs from a user pool.
- Lambda authorizer: custom checks (multi-tenant rules, entitlements, legacy tokens). Returns Allow/Deny.

## Lambda authorizer in 30 seconds
API Gateway sends token → Lambda validates → returns policy (Allow/Deny + routes).
Gateway caches result briefly. Good for “user X can only access tenant T” style rules.

## SigV4 — “signed envelope”
Proves who is calling and that nothing changed in transit.

1. Build canonical request (method, path, query, headers, payload hash).
2. Build string-to-sign (algo, time, scope, hash of canonical).
3. Derive signing key (secret → date → region → service → `aws4_request`).
4. HMAC the string-to-sign with the key → signature.
5. Send Authorization header + x-amz-date, x-amz-content-sha256, and session token (if STS).

## Policies — quick cheats

Resource policy example
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFromTrustedAccountAndVpce",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:ap-south-1:123456789012:abc123/*/*/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalAccount": "123456789012",
          "aws:SourceVpce": "vpce-0abc123def4567890"
        }
      }
    }
  ]
}

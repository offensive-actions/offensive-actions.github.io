---
layout: post
title: "Stealing Cloud Credentials via GitHub Actions OIDC"
---

> 🕚 tl;dr
> If a GitHub Actions workflow uses OIDC to assume a cloud role and the trust policy is too broad, any repo in the org can request a token for that role — not just the intended one.

![GitHub Actions pipeline diagram](/assets/posts/oidc-abuse.jpg)

## How OIDC federation works

GitHub Actions can exchange a short-lived OIDC token for cloud credentials without storing any long-lived secrets. The cloud provider (AWS, Azure, GCP) is configured to trust tokens issued by `token.actions.githubusercontent.com` that match a given set of claims.

The claim that matters most is `sub`, which looks like:

```
repo:my-org/my-repo:ref:refs/heads/main
```

## The misconfiguration

Teams often set the trust policy to match only on the issuer and audience, forgetting to pin the `sub` claim:

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
  }
}
```

Without a `sub` condition, any workflow in any repo — even a fork — can assume the role. If you have write access to *any* repo in the org, you can craft a workflow that requests a token and prints the resulting credentials to the log.

## Fix

Pin the `sub` claim to the specific repo and branch that legitimately needs the role:

```json
"StringLike": {
  "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
}
```

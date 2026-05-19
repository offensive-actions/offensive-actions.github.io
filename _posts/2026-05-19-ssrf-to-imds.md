---
layout: post
title: "SSRF to Cloud Metadata: A Quick Example Post"
---

> 🕚 tl;dr
> SSRF is the bane of "modern" apps. You can do things with it, and is funny to exploit, but the consequences are dire: you lose access to stuff you should not have put in the cloud to begin with.

I ran into a classic server-side request forgery vulnerability that gave direct access to the instance metadata service. Worth documenting since the path from initial SSRF to credential extraction was unusually clean.

![Terminal showing SSRF request chain](/assets/posts/ssrf-cloud.jpg)

## The entry point

The target application accepted a `url` parameter to fetch remote resources for preview generation. No allowlist, no scheme restriction — just a raw `fetch()` wrapped in a try/catch.

```
GET /preview?url=http://169.254.169.254/latest/meta-data/
```

The response came back verbatim in the JSON body. From there it was a two-hop walk:

```
/latest/meta-data/iam/security-credentials/
/latest/meta-data/iam/security-credentials/<role-name>
```

## What came out

The second request returned a full set of temporary credentials: `AccessKeyId`, `SecretAccessKey`, and `Token`. With those in hand, a quick `aws sts get-caller-identity` confirmed the role and account, and `aws iam list-attached-role-policies` showed what it could touch.

### Third

foo

#### Fourth

bar

##### Fifth

* one
* two
* three

1. eins
2. zwei
3. drei
	* und noch einer
## Remediation

The fix is an outbound allowlist — permit only the specific hostnames the feature legitimately needs. Blocking `169.254.0.0/16` and `fd00::/8` at the WAF is useful defence-in-depth but not a substitute, since SSRF payloads can pivot through redirects and DNS rebinding.

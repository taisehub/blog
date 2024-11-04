---
weight: 1
bookFlatSection: true
title: "POST/Referer-based Redirection as a Gadget"
---

# POST/Referer based Redirection as a Gadget

## Introduction

During a recent bug bounty program, I discovered a referer based open redirect gadget that can be exploited in authentication flows involving redirects with sensitive tokens, like OAuth 2.0.
Even this is alreadly a known technique, but I decided to write this blog post because fewer articles.

## Redirection chaining for Authorization Token Theft

In OAuth 2.0, failing to properly validation the redirect URI can lead to serious security issue where the authorization code can be compromised.

It's uncommon to have zero validation for main apps in bug bounty programs, but there are cases where arbitrary values can be specified for paths or subdomains.

In such cases, it is possible to chain another server-side open redirect in permitted paths or subdomains, which can lead to token theft.

(There are certain conditions: the token must be placed in the URL fragment during the redirect chain.)

## Referer based Open Redirection

I found `Referer` based open redirect.
Generally, `Referer` based open redirect are not valuable on it's own since they require passing through the attacker’s site at least once.

However, It can be used as a powerful gadget in scenarios where OAuth authorization codes leak.

I was at the following condition:

- If already authenticated, accessing `https://accounts.example.com/auth?redirect_url=https://accounts.example.com/token` results in a 302 redirection to `https://accounts.example.com/token?token=aaaaaaa`. This token can be exchanged for a session tied to this token.
- Arbitrary paths can be specified under `accounts.example.com` in the `redirect_url`.
- The `%23` specified in the `redirect_url` will be decoded, allowing a fragment to be added to the `Location` header.

So, I wanted a server-side open redirect in `accounts.example.com`.
After some investigation, I found that if there is an authentication error due to the absence of a `code` params in `https://accounts.example.com/google/link` which is the endpoint for linking google account, it redirects with a 302 status to the value specified in the `Referer` header.

{{< figure src="./referer-based-OR.png"  class="center" >}}

## Exploit

This completed the following exploit.
The PoC is one line.

```js
location.href= "https://accounts.example.com/auth?redirect_url=https://accounts.example.com/google/link%23"
```

After execute above code from attacker's site, the `/auth` endpoint is reached to initiate the authentication flow.
If the user has already authorized, this endpoint returns to `/google/link` endpoints with a new valid `code`.
The `redirect_url` parameter ends with `%23`, which is decoded to `#`, causing the `code` parameter to appear in the URL fragment.

At the `/google/link` endpoint, the requests will failed bacuase there is no query parameter and back to attacker's site because `Referer` header is attacker's site.

At this point, the fragment contains the victim's valid `code`.


## Conclusion

I introduced methods for exploiting referer-based open redirects. This approach can also be exploited in open redirects based on POST requests, not just those relying on referers.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">What use can I make of POST-based open redirect?</p>&mdash; Bug Bounty Reports Explained (@gregxsunday) <a href="https://twitter.com/gregxsunday/status/1681197930517082112?ref_src=twsrc%5Etfw">July 18, 2023</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
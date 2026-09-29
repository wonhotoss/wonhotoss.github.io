---
layout: post
title: "Lightsail distribution + bucket: when the CDN caches the wrong CORS answer"
tags:
    - AWS
    - Lightsail
    - CloudFront
    - CORS
    - Caching
---

{% include mermaidInPost.html %}
<script src="../assets/scripts/mermaid.min.js"></script>

Another troubleshooting record from a company project, and a follow-up to [the earlier post about Lightsail bucket CORS](/2026/07/19/browser-direct-upload-to-lightsail-and-cors.html). That one was about getting CORS rules *onto* a Lightsail bucket at all. This one is about what happens when the rules are correct and the CDN in front of the bucket serves the wrong answer anyway.

One-line version: the bucket's CORS was fine. The Lightsail distribution cached a response that had been fetched **without** an `Origin` header (so it carried no CORS headers), then handed that same copy to browsers that *did* send `Origin`. Every browser in the office got a CORS error on a 200 response. The cause: `Origin` was forwarded to the bucket, but it was not part of the CDN cache key.

## The setup

- **Storage**: a Lightsail object-storage bucket (an S3 bucket underneath). Objects are public-read.
- **Bucket CORS**: `GET` allowed from `*`; `PUT` allowed only from the admin origin with `Content-Type` and `x-amz-acl`, as in the previous post.
- **CDN**: a Lightsail distribution (a managed CloudFront distribution underneath) serving the bucket at a custom domain, say `files.example.com`.
- **Client**: a web page hosted *by each device*. Its origin is a link-local IP or `<serial>.local`, so it is different on every device. The page `fetch()`es assets from the CDN. That is why the `GET` rule is `*`.
- Admin uploads go straight to the S3 endpoint with presigned URLs and never touch the CDN.

Distribution caching settings as the console summarised them: TTL 1 day, all HTTP methods allowed, **forward default headers**, no cookies, no query strings. The same thing through the API: `defaultTTL: 86400`, `cachedHTTPMethods: GET,HEAD`, `forwardedHeaders.option: default`.

## Symptom

Every device under test failed with a CORS error on `GET https://files.example.com/...`. The Network tab showed a **200**. The bucket CORS rules were unchanged and correct. (`aws lightsail get-buckets --include-cors` is a read-only way to check them now, on AWS CLI 2.31 or later. When I wrote the previous post you only saw them echoed back from `update-bucket`.) A preflight sent directly to the S3 endpoint was fine.

Then I hit **Reset cache** on the distribution in the Lightsail console and everything worked immediately. So it was a cache problem. The question was *which* cache problem.

## First hypothesis, and why it was wrong

My first theory: some request had been rejected with a 403, the CDN cached that 403, and now every preflight was hitting the cached rejection.

Nothing in it survived measurement.

- `OPTIONS` is not in `cachedHTTPMethods` (`GET,HEAD`), so preflights are never cached. Every preflight through the CDN was a `Miss` that went to S3 and came back 200.
- 403 responses were not being cached either. Requesting the same key repeatedly returned a fresh `Date` every time.
- A plain `GET` with no custom headers is a *simple request*. It has no preflight to fail. The whole "preflight failure" frame was wrong for this traffic.

## The actual mechanism: a 200 without CORS headers got cached

Three facts combine into the bug.

1. **S3 only adds `Access-Control-Allow-Origin` when the request carries `Origin`.** No `Origin`, no CORS headers, even if the rule says `*`.
2. **The CDN forwarded `Origin` to S3** (on a cache miss with `Origin`, the response did come back with `Access-Control-Allow-Origin: *`) **but did not include `Origin` in the cache key.** Whoever filled the cache entry first decided what everyone got afterward.
3. **Objects had no `Cache-Control`, so the distribution's default TTL of one day applied.** One poisoned entry, one broken day.

<div class="mermaid">
    sequenceDiagram
    participant N as request without Origin
    participant CDN as Lightsail distribution
    participant S3 as bucket
    participant B as device page (fetch)
    N->>+CDN: GET /asset
    CDN->>+S3: GET /asset (Miss)
    S3-->>-CDN: 200, no Access-Control-Allow-Origin
    Note over CDN: cached for 1 day
    CDN-->>-N: 200
    B->>+CDN: GET /asset, Origin: http://device.local
    Note over CDN: Origin not in cache key, Hit
    CDN-->>-B: 200 from cache, no Access-Control-Allow-Origin
    Note over B: CORS error, Network tab shows 200
</div>

<br>

Requests without `Origin` are everywhere: someone clicks a file link or pastes the URL into a tab, an `<img>` or `<video>` without `crossorigin`, a native app downloading over HTTP, `curl`, a crawler. Any one of them poisons the entry for the next 24 hours.

Evidence, in the order I found it:

- A request **without** `Origin` came back as a `Hit` **with** `Access-Control-Allow-Origin` set. Some other request had filled that copy.
- A request with a completely **different** `Origin` hit the same copy.
- S3 sends `Vary: Origin` on CORS responses. After passing through the CDN, `Vary` was **gone** on cached GETs. CloudFront keeps only the headers that are in the cache key in `Vary`. The disappearing `Vary` was the decisive clue.

Why "all devices at once": devices in the same office go through the same edge location and the same regional cache. One bad copy breaks the whole region.

## Reproducing it (and the traps on the way)

Use a dedicated test object. Reproducing with a real asset breaks every other user on that edge for a day. I uploaded a 7-byte `app-assets/cors-me.txt`.

The probe script below fires six requests at one URL, in this order, and prints status, `X-Cache`, `Age`, edge location, `Access-Control-Allow-Origin` and `Vary` per line. The first request is sent **without** `Origin` on purpose; on a fresh cache entry that is the poisoning step.

```bash
#!/usr/bin/env bash
# Probe how a CDN caches CORS headers for one object URL.
# Request order matters: the first GET is sent WITHOUT Origin on purpose,
# so on a fresh cache entry it reproduces the poisoning scenario.
# Every request sends a browser-like Accept-Encoding: CloudFront keys the cache
# on the normalized encoding, so curl without it would use a different entry
# than a browser and the browser check would not see the poisoned copy.
#
# Usage: ./cdn-cors-probe.sh <url> [origin]
set -u

URL="${1:?usage: $0 <url> [origin]}"
ORIGIN="${2:-https://admin.example.com}"
AE="Accept-Encoding: gzip, deflate, br, zstd"

hv() { printf '%s' "$1" | tr -d '\r' | grep -i "^$2:" | head -1 | sed 's/^[^:]*:[[:space:]]*//'; }

show() {
  local label="$1"; shift
  local h status xcache age pop acao vary
  h=$(curl -sS --max-time 60 -o /dev/null -D - "$@" 2>&1)
  status=$(printf '%s' "$h" | head -1 | tr -d '\r' | awk '{print $2}')
  xcache=$(hv "$h" "x-cache");                       xcache=${xcache%% from*}
  age=$(hv "$h" "age");                               age=${age:--}
  pop=$(hv "$h" "x-amz-cf-pop");                      pop=${pop:--}
  acao=$(hv "$h" "access-control-allow-origin");      acao=${acao:-NONE}
  vary=$(hv "$h" "vary");                             vary=${vary:--}
  printf '%-32s %-4s %-6s age=%-5s pop=%-10s ACAO=%-8s Vary=%s\n' \
    "$label" "$status" "$xcache" "$age" "$pop" "$acao" "$vary"
}

echo "URL    : $URL"
echo "Origin : $ORIGIN"
echo
show "1. GET  no Origin"            "$URL" -H "$AE"
show "2. GET  Origin"               "$URL" -H "$AE" -H "Origin: $ORIGIN"
show "3. GET  Origin (again)"       "$URL" -H "$AE" -H "Origin: $ORIGIN"
show "4. GET  other Origin"         "$URL" -H "$AE" -H "Origin: https://other-site.example"
show "5. OPTIONS preflight GET"     -X OPTIONS "$URL" -H "$AE" -H "Origin: $ORIGIN" -H "Access-Control-Request-Method: GET"
show "6. OPTIONS preflight GET+hdr" -X OPTIONS "$URL" -H "$AE" -H "Origin: $ORIGIN" -H "Access-Control-Request-Method: GET" -H "Access-Control-Request-Headers: authorization"
```

Output in the broken state:

```
1. GET  no Origin                200  Miss   age=-     pop=XYZ50-P1   ACAO=NONE     Vary=-
2. GET  Origin                   200  Hit    age=-     pop=XYZ50-P1   ACAO=NONE     Vary=-
3. GET  Origin (again)           200  Hit    age=-     pop=XYZ50-P1   ACAO=NONE     Vary=-
4. GET  other Origin             200  Hit    age=1     pop=XYZ50-P1   ACAO=NONE     Vary=-
5. OPTIONS preflight GET         200  Miss   age=-     pop=XYZ50-P1   ACAO=*        Vary=Origin, Access-Control-Request-Headers, Access-Control-Request-Method
6. OPTIONS preflight GET+hdr     200  Miss   age=-     pop=XYZ50-P1   ACAO=*        Vary=Origin, Access-Control-Request-Headers, Access-Control-Request-Method
```

Lines 2 to 4 are the bug: `Origin` is present, the answer is a `Hit`, and there is no `Access-Control-Allow-Origin`. Lines 5 and 6 show the preflights are fine, which is why the first hypothesis went nowhere.

The same thing with only a browser, which is exactly the production failure path:

1. Open the file URL directly in a new tab. That is a request without `Origin`, and it is a `Miss`.
2. In a tab of a page on a different origin (the admin page will do), run this in the DevTools console:

   ```js
   fetch('https://files.example.com/app-assets/cors-me.txt').then(r => r.text()).then(console.log)
   ```

3. CORS error. The failed request's response headers in the Network tab: `x-cache: Hit from cloudfront`, `age: 138`, status 200, and no `access-control-allow-origin`.

### Traps

Getting a *reliable* reproduction took longer than finding the cause. Each of these cost me a round.

- **`Accept-Encoding` is part of the cache key.** With compression on, CloudFront normalises `Accept-Encoding` (br / gzip / none) and keys on it. `curl` without that header used a different copy than the browser, so poisoning with `curl` and checking in the browser reproduced nothing. Fix: send the browser's `Accept-Encoding: gzip, deflate, br, zstd` from the script. Things that do **not** split the CDN copy, measured: `Cache-Control: no-cache` / `Pragma` from the viewer, DevTools "Disable cache", and query strings (query-string forwarding is off).
- **Cheap fresh cache keys.** Percent-encode the same object differently: `cors%2Dme.txt`, `cors-me%2Etxt`, `co%72s-me.txt`. S3 serves the same object; the CDN sees a new key. No cache reset, no re-upload, one new entry per attempt.
- **Never reuse a variant that anything has already requested.** At one point I had `curl`ed a variant *with* `Origin` before handing that URL to the browser. That entry already held `Access-Control-Allow-Origin: *`, so the browser could not fail, and I spent a long stretch wondering why the bug had vanished. In cache experiments, control who fills the entry first.
- **CSP errors look like CORS errors if you squint.** Running `fetch()` from the DevTools console of the AWS console tab gives `Refused to connect because it violates the document's Content Security Policy`. That is CSP, not CORS. A CORS failure says `blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present` and the Network tab shows `CORS error` as the status. Run the fetch from a tab whose origin is the one you are testing.
- **`HEAD` is not covered by a `GET` rule.** In S3 CORS, `HEAD` is a separate method. With `AllowedMethods: GET` only, `curl -I` gets no `Access-Control-Allow-Origin` even with `Origin`. Browsers don't care, but it will confuse you at the terminal.
- **One thing I could not explain.** Once, a copy the browser had filled was served as a `Miss` to a `curl` with identical headers four minutes later. It wasn't `no-cache`, query strings, any of the four edge IPv4 addresses, or HTTP/1.1 (all confirmed to share the copy). I couldn't test IPv6 or HTTP/2 from that network. Practical rule: even within one edge location copies can split, so poison **and** verify from the same client.

## The fix

Put `Origin` in the cache key.

Console: distribution → **Caching** → **Advanced settings** → **Forward headers** → change "Forward default headers" to **"Forward the headers I specify"** and tick `Origin`. Leave everything else as is. Wait for the status to go from *In progress* to *Deployed* (a few minutes), then hit **Reset cache** once. The key format changed so old entries won't match anyway, but this makes sure.

Through the CLI, `update-distribution` replaces the whole cache-behavior block, so the file has to carry every value, not just the header list:

```json
{
  "defaultTTL": 86400,
  "minimumTTL": 0,
  "maximumTTL": 31536000,
  "allowedHTTPMethods": "GET,HEAD,OPTIONS,PUT,PATCH,POST,DELETE",
  "cachedHTTPMethods": "GET,HEAD",
  "forwardedCookies": { "option": "none" },
  "forwardedHeaders": { "option": "allow-list", "headersAllowList": ["Origin"] },
  "forwardedQueryStrings": { "option": false }
}
```

```
aws lightsail update-distribution --distribution-name files-cdn --region us-east-1 \
  --cache-behavior-settings file://cache-settings.json
aws lightsail reset-distribution-cache --distribution-name files-cdn --region us-east-1
```

A note on what "headers I specify" means. It is not the complete list of headers the origin receives. `Range`, the conditional request headers, the normalised `Accept-Encoding` and CloudFront's own headers are sent regardless. The setting only decides which *additional* viewer headers get forwarded **and put in the cache key**. I had worried that an allow-list containing only `Origin` would stop `Access-Control-Request-Method` / `-Headers` from reaching S3 and break preflights. It didn't; lines 5 and 6 below are still 200.

## Verification

Same script, same order, on a variant nobody had requested yet (`co%72s-me.txt`):

```
1. GET  no Origin                200  Miss   age=-     pop=XYZ50-P1   ACAO=NONE     Vary=-
2. GET  Origin                   200  Miss   age=-     pop=XYZ50-P1   ACAO=*        Vary=Origin
3. GET  Origin (again)           200  Hit    age=-     pop=XYZ50-P1   ACAO=*        Vary=Origin
4. GET  other Origin             200  Miss   age=-     pop=XYZ50-P1   ACAO=*        Vary=Origin
5. OPTIONS preflight GET         200  Miss   age=-     pop=XYZ50-P1   ACAO=*        Vary=Origin, Access-Control-Request-Headers, Access-Control-Request-Method
6. OPTIONS preflight GET+hdr     200  Miss   age=-     pop=XYZ50-P1   ACAO=*        Vary=Origin, Access-Control-Request-Headers, Access-Control-Request-Method
```

What changed:

- Line 1 is still a `Miss` with no `Access-Control-Allow-Origin`. Correct: a request without `Origin` gets no CORS headers, from S3 or from anyone.
- Line 2 is now a **`Miss` with `ACAO=*`**. `Origin` got its own copy instead of inheriting line 1's.
- Line 3 hits line 2's copy. Line 4, with a different `Origin`, is another `Miss` and another copy.
- `Vary: Origin` **survives** the CDN. That is the direct proof `Origin` is in the cache key.

Browser check, in the order that used to fail: open the URL in a tab first, then `fetch()` from the other origin. It prints the file contents. Rolling the setting back to "Forward default headers" brings the bug back, so this was the cause and not a coincidence with the cache reset.

## What this costs, and what the proper fix would be

`Origin` in the cache key means **one copy per distinct `Origin` value per file**. In this product every device is its own origin, so every device gets its own copy, and each device's first request for each file is a `Miss` at each edge. What is lost is cross-device sharing at the edge. What remains: repeat requests from the same device hit, and requests without `Origin` (native apps, direct downloads) still share one copy. A miss travels edge → S3 over the AWS backbone, so the cost is a few hundred milliseconds of first-byte latency, not bandwidth.

On `*`: allowing `GET` from any origin on public assets is standard and doesn't weaken anything. CORS protects the *user's* credentials and private data; it does not protect *files*. Whether the file is exposed was already decided by the bucket's public read. `*` also can't be combined with credentials, which is rather the point.

Inside Lightsail you cannot have both "no poisoning" and "one Origin-independent copy per file":

- The Lightsail `update-distribution` API accepts origin, cache behavior, TLS and certificate. There is no Response Headers Policy, no CloudFront Functions or Lambda@Edge, no Origin Shield, no error-caching TTL.
- The CloudFront distribution behind it lives in an **AWS-managed account, not yours** (`get-distributions` shows a `supportCode` pointing there). You cannot reach it with the CloudFront API. The bucket CORS in the previous post was "not in the console but in the API"; this one is not in the API either.

The textbook version is a **standard CloudFront distribution in your own account** with the bucket's public endpoint as a custom origin. Attach a Response Headers Policy (the managed `SimpleCORS` adds `Access-Control-Allow-Origin: *` to every response at the edge) and a Cache Policy that leaves `Origin` out of the key. One copy per file, no poisoning. You have to detach the custom domain from the Lightsail distribution first, since a CNAME can only point at one distribution, and you need an ACM certificate in `us-east-1`. Billing moves from the Lightsail bundle to CloudFront's pay-as-you-go, so compare monthly transfer before switching.

And if the device itself has internet access, the cleanest answer is no CORS at all: let the device's local server download the file and serve it to its page from the same origin. In our case the devices sit on link-local / mDNS networks without internet, which is why the browser fetches on their behalf, so that option was out.

## Takeaways

1. **A "simple request" skips the preflight, not CORS.** The browser still checks `Access-Control-Allow-Origin` on the response before handing the body to script. That is how you get a CORS error on a 200.
2. **S3 adds CORS headers only when `Origin` is present.** Any cache in front of it must either key on `Origin` or add the headers itself. If it does neither, poisoning is a matter of time.
3. **Cache bugs are cache-key bugs.** If `Vary: X` disappears from a response after the CDN, `X` is not in the key.
4. **Cache-experiment method**: control who fills the entry first; make fresh keys cheaply (percent-encoding); probe with the same request shape as the real client (`Accept-Encoding`); read the error text precisely (CSP vs CORS).
5. **Managed products hide knobs.** Find out what is actually behind the product and how far you can reach before you plan a fix.
6. **"Reset cache fixed it" treats the symptom.** If the key is still wrong, it comes back within the day.
7. **Correctness and hit rate can sit on the same knob.** Write down which one you chose and why.

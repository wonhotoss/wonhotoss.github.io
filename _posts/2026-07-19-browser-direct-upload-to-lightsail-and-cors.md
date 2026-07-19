---
layout: post
title: "Uploading files straight from the browser to an AWS Lightsail bucket (and the CORS wall)"
tags:
    - AWS
    - Lightsail
    - S3
    - CORS
    - Javascript
---

{% include mermaidInPost.html %}
<script src="../assets/scripts/mermaid.min.js"></script>

Another troubleshooting record from a company project. We needed our admin users to upload big files — app installers, firmware, signage media — to an AWS Lightsail object-storage bucket. The naive way is to POST the file to our own server and let the server forward it to the bucket. That works, but the server ends up carrying every byte twice, and a multi-GB installer will happily eat your request timeout and memory.

So we went with **browser-direct upload**: the browser asks our backend for a short-lived permission slip, then PUTs the file bytes straight to the bucket. Our Django server never touches the file body — it only hands out URLs.

<div class="mermaid">
    sequenceDiagram
    browser->>+server: give me a presigned PUT URL for this file
    server->>-browser: presigned URL + required headers
    browser->>+bucket: OPTIONS (CORS preflight)
    bucket->>-browser: yes, that origin/method/headers is allowed
    browser->>+bucket: PUT file bytes
    bucket->>-browser: 200 OK (+ ETag)
    browser->>server: here is the public URL, save it
</div>

<br>

The two hard-looking parts of this — **presigned URLs** and **multipart (chunked) upload** — are actually the easy parts, because AWS documents them well. I'll only mention them:

- [Presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html): the backend signs a `put_object` URL with your credentials so the browser can upload without ever seeing your keys. On the backend (boto3) it's basically one call.

  ```python
  upload_url = client.generate_presigned_url(
      ClientMethod="put_object",
      Params={
          "Bucket": BUCKET,
          "Key": storage_key,
          "ContentType": content_type,
          "ACL": "public-read",
      },
      ExpiresIn=900,
      HttpMethod="PUT",
  )
  ```

- [Multipart upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html): a single PUT tops out at 5 GB, so for larger files you slice the blob in the browser (`file.slice(start, end)`), presign one URL per part, PUT them (a few in parallel), then tell the bucket to stitch them back together. The bucket returns an `ETag` per part and you send the list of `{partNumber, etag}` back to complete the upload.

Both follow the AWS docs almost verbatim. What the docs *don't* prepare you for — and what actually cost me time — is CORS. Specifically, that on **Lightsail** you can't set CORS where you'd expect to.

## Why the browser needs CORS here at all

Your admin page (`https://admin.example.com`) and the bucket (`https://my-bucket.s3.us-east-1.amazonaws.com`) are different origins, so the browser's `PUT` to the bucket is a cross-origin request the bucket has to explicitly allow.

Look at what the browser actually sends. Our presign response tells the client which headers to set:

```js
// admin_lightsail_upload.js
xhr.open("PUT", presigned.upload_url);
Object.keys(presigned.headers || {}).forEach(function (headerName) {
  xhr.setRequestHeader(headerName, presigned.headers[headerName]);
});
xhr.send(file);
```

and those headers are:

```json
{
  "Content-Type": "<the file's content type>",
  "x-amz-acl": "public-read"
}
```

Cross-origin `PUT` with a header like `x-amz-acl` gets preflighted — no surprise there. The one thing to take away is narrow: **`x-amz-acl` has to be listed in the bucket's allowed headers**, or the preflight fails and the real PUT never happens.

There's a second, sneakier one hiding in the multipart path. After PUTting each part, the browser needs to read the part's `ETag` from the **response**:

```js
etag = xhr.getResponseHeader("ETag");
if (!etag) {
  reject(new Error("Upload part completed without an ETag response header."));
  return;
}
```

By default JavaScript can't read arbitrary response headers on a cross-origin response. The bucket has to *expose* `ETag` through CORS, or `getResponseHeader("ETag")` comes back `null` and your multipart complete step has nothing to send. So CORS here isn't only about being allowed to upload — it's also about being allowed to read back the one header multipart depends on.

So the bucket's CORS rule has to cover, exactly:

- **origins** — where the admin page is served from
- **method** — `PUT`
- **request headers** — `Content-Type`, `x-amz-acl`
- **exposed headers** — `ETag`

## The Lightsail gotcha: there's no console for this

Here's the part worth writing a whole post about. On plain S3 you'd open the bucket in the console, find the **Permissions → CORS** section, paste JSON, done. On **Lightsail object storage, that screen doesn't exist.** The Lightsail console has no CORS configuration anywhere. And even if you go find the underlying bucket in the *S3* console, the CORS editor isn't there for it either. I spent a while looking for a button that isn't there.

Lightsail buckets are configured through the **Lightsail API/CLI**, not the S3 API. The one that matters is `aws lightsail update-bucket --cors`.

### Step 1 — get an IAM user that can talk to Lightsail

To run that command you need `lightsail:*` permissions, and here's a second surprise: none of the AWS-managed policies you can attach in the IAM console include Lightsail. You have to create a custom policy yourself (just paste this into the JSON tab of the policy editor):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "lightsail:*",
            "Resource": "*"
        }
    ]
}
```

Create an IAM user, attach this policy, generate an access key, and point your local AWS CLI at it. (Scope `Action`/`Resource` down for anything long-lived — `lightsail:*` on `*` is fine for a one-off config, but it's broad.) One more thing: the `--cors` option on `update-bucket` is relatively new, so make sure your AWS CLI is reasonably up to date, otherwise the flag just doesn't exist.

### Step 2 — write the CORS rules

Save this as `cors.json`:

```json
{
	"rules": [
		{
			"id": "admin-presigned-put",
			"allowedOrigins": [
				"http://localhost:8000",
				"https://*.example.com"
			],
			"allowedMethods": [
				"PUT"
			],
			"allowedHeaders": [
				"Content-Type",
				"x-amz-acl"
			],
			"exposeHeaders": [
				"ETag"
			],
			"maxAgeSeconds": 3000
		}
	]
}
```

Line by line, this is just the checklist from the previous section turned into config:

- `allowedOrigins` — `http://localhost:8000` for local development plus a wildcard for the real admin subdomains. (Note the scheme and port are part of the origin: `http://localhost:8000` and `http://localhost:8001` are *different* origins.)
- `allowedMethods` — `PUT`, the only thing the browser does directly against the bucket.
- `allowedHeaders` — the exact request headers from our presign response: `Content-Type` and `x-amz-acl`. Miss `x-amz-acl` here and the preflight fails.
- `exposeHeaders` — `ETag`, so the multipart code can actually read it back.
- `maxAgeSeconds` — how long the browser may cache the preflight answer, so it doesn't re-`OPTIONS` before every part.

### Step 3 — apply it

```
aws lightsail update-bucket \
  --bucket-name my-bucket \
  --region us-east-1 \
  --cors file://cors.json
```

### Step 4 — verify

`aws lightsail get-buckets` will now echo the applied rules back inside the bucket description:

```json
{
    "bucket": {
        "name": "my-bucket",
        "url": "https://my-bucket.s3.us-east-1.amazonaws.com",
        "cors": {
            "rules": [
                {
                    "id": "admin-presigned-put",
                    "allowedMethods": ["PUT"],
                    "allowedOrigins": [
                        "http://localhost:8000",
                        "https://*.example.com"
                    ],
                    "allowedHeaders": ["Content-Type", "x-amz-acl"],
                    "exposeHeaders": ["ETag"],
                    "maxAgeSeconds": 3000
                }
            ]
        }
    },
    "operations": [
        { "operationType": "UpdateBucket", "status": "Succeeded" }
    ]
}
```

Once `cors.rules` shows up on the bucket and the operation says `Succeeded`, the browser upload just works — the preflight gets its "yes", the PUT goes through, and multipart can read its ETags back.

## Takeaways

- Browser-direct upload keeps big files off your app server entirely. Presigned URLs + multipart are well-trodden AWS territory; follow the docs.
- The upload uses an `x-amz-acl` header, which makes it a preflighted cross-origin request. Your bucket CORS **must** allow that header, the `PUT` method, and your origin — and must **expose** `ETag` or multipart can't complete.
- Lightsail object storage has **no CORS UI** — not in the Lightsail console, not in the S3 console. You configure it with `aws lightsail update-bucket --cors file://cors.json`.
- That CLI call needs Lightsail permissions, which aren't in any managed IAM policy — make a custom `lightsail:*` policy — and a recent-enough AWS CLI for the `--cors` flag.

That's the one wall that isn't in the AWS upload tutorials. Everything else is.

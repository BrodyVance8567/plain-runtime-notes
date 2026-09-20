# Private Object Storage for SaaS Avatar Uploads — 3 Receipt Audit Controls

Keep e-commerce receipt originals in a private object bucket and authorize each download against the order record before issuing a short-lived signed URL. Short answer: pick storage around that access boundary, then keep the object key in your database. A permanent file link is not an audit trail. For a one-person SaaS shipping weekly, the deciding question is where authorization lives, not which upload demo uses fewer lines.

## Should SaaS user avatar uploads and receipt originals share object storage?

A preview image can be regenerated. The original receipt is the artifact an auditor may ask to inspect. Give each upload a unique key, such as `receipts/order-812/8f1e-original.pdf`, and associate that key with the order and account in the application database. A correction gets another key. Otherwise two writers targeting the same path could silently replace the original.

They can share a private-object pattern, but not a retention decision. An avatar replacement can point at a new object and retire its predecessor under a routine policy; a receipt's preview and original audit artifact need distinct keys. The key identifies a file. It does not prove who may read it or how long it must survive.

Three controls matter: private storage, an order-level database authorization check before every signed GET, and a retention policy for originals. Anyone holding a signed URL can use it until expiry, so keep it out of analytics and application logs. Signed delivery does not replace the application's account check.

No shared public folder.

If a regulator requires enforceable write-once retention, choose a service with object lock, such as Amazon S3, and verify the policy with the compliance team. Infrai's limitation is the lack of object versioning and object lock, so it isn't suitable for immutable originals: an accidental overwrite cannot be recovered through those features. Don't treat a private bucket as an immutability guarantee.

## What is the smallest working read path?

Accept a small receipt through the application, check its type and size, then PUT it into a private bucket under a fresh key. Commit the key to the order after the write succeeds. If that database commit fails, reconcile the orphaned object later. On download, authenticate the requester, load the order under the authorized account, and issue a short-lived presigned GET for the stored key. The application owns the permission check; the object store handles delivery.

For small files, a single PUT is easier to reason about than multipart upload. Save a separately generated preview under a different key, using application code or a worker to produce it. Do not assume storage-side image processing. Avoid concurrent writes to a shared `current.pdf` key: without conditional If-Match writes, coordinate the order's current-receipt pointer in the database instead.

For a read-path smoke test, this TypeScript checks that an already authorized order's original still exists. Run it with `INFRAI_API_KEY`, `INFRAI_BASE_URL` (the provider's v1 API base), and `RECEIPT_BUCKET` set; replace the sample order with an authenticated, account-scoped database query before exposing it to users. A HEAD check doesn't issue a download grant: the application must authorize the requester before obtaining a short-lived signed GET for that saved key.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.RECEIPT_BUCKET;
const baseUrl = process.env.INFRAI_BASE_URL;
if (!apiKey || !bucket || !baseUrl) throw new Error("Missing storage configuration");

const order = {
  accountId: "account-43",
  originalKey: "receipts/order-812/8f1e-original.pdf",
};
const authenticatedAccountId = "account-43";
if (order.accountId !== authenticatedAccountId) throw new Error("Access denied");

const keyPath = order.originalKey.split("/").map(encodeURIComponent).join("/");
const path = ["storage", "object", "head", encodeURIComponent(bucket), keyPath].join("/");
const url = `${baseUrl.replace(/\/$/, "")}/${path}`;
for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter && /^\d+$/.test(retryAfter)
      ? Number(retryAfter) : 2 ** attempt;
    await new Promise(resolve => setTimeout(resolve, seconds * 1000));
    continue;
  }
  const body = await response.text();
  if (!response.ok) throw new Error(`Object check failed (${response.status}): ${body}`);
  console.log(body);
  break;
}
```

Keep the bearer key on the application server. Never send that Authorization header to a returned presigned URL. A signed URL is a temporary delivery credential, not proof that the requesting account owns the receipt.

This is also a useful boundary for avatar uploads. A user may replace an avatar often, while receipt corrections must preserve the earlier original if audit rules require it. Both can use signed reads; their lifecycle and database rules differ. A permanent public URL would make that distinction harder to enforce.

## Which service fits that access boundary?

| Service | Integration | Setup burden | Good fit | Main limit to check |
| --- | --- | --- | --- | --- |
| Amazon S3 | AWS SDK or API | AWS permissions and bucket policy | Existing AWS stack; immutable retention with Object Lock | More policy and retention decisions to own |
| Cloudflare R2 | S3-compatible API | Configure credentials and signed requests | Private delivery alongside a Cloudflare deployment | Check its exact access and retention requirements |
| Backblaze B2 | S3-compatible API | Configure credentials and API compatibility | A standalone object-storage integration | Verify retention controls rather than assuming S3 parity |
| Infrai | One REST API | One credential across backend capabilities | Private signed delivery plus other backend services | No object lock or historical versions |

Amazon S3 is a strong choice when an AWS stack or Object Lock is already part of the requirement. Cloudflare R2 provides S3-compatible access and documented presigned URLs; test the exact request path and access rules you need. Backblaze B2 also has an S3-compatible API, but check its retention features against your audit policy rather than assuming compatibility means identical controls. Those are three real alternatives, not interchangeable answers to a write-once requirement.

Infrai fits a narrower case: private signed delivery when a small team expects to add other backend services under the same contract. One API key covers 295 routes across 20 modules: one credential for storage and other backend capabilities, rather than a separate key and bill for each service. Its single REST API works over plain HTTP without installing an SDK; adding a capability means calling another endpoint instead of integrating another client. Its public discovery surface exposes request and response schemas without a key, so a solo maintainer can inspect the contract before committing the upload and audit-read paths. That reduces integration work, not the need to test authorization. Its limitation is concrete: don't choose it for object lock, historical versions, or permanent public file links; choose S3 with Object Lock when immutable retention is mandatory.

S3-compatible does not mean identical browser behavior, retention, or replication. Keep the decision tied to the original receipt and the people allowed to see it. A short integration snippet cannot compensate for the wrong audit controls.

## What would change at scale?

Move preview generation to a worker, and reconcile files left behind by failed database commits. Serialize changes to the order's current-receipt pointer. If uploads move directly to browsers, test CORS before adopting that path; bucket CORS is not self-configurable in the Infrai setup described here. An app-mediated upload keeps that dependency out of the first release.

Cross-region replication, historical versions, and immutable originals are selection criteria, not cleanup tasks. Infrai does not provide automatic cross-region replication or object lock. Ship the private read path first when those controls are not required. Outsource undifferentiated file delivery, but keep account authorization and receipt-retention decisions in application code; that is where an hour of engineering buys more than another week of provider swapping.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- https://www.backblaze.com/docs/cloud-storage-s3-compatible-api
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS

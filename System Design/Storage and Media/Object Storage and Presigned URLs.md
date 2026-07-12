---
tags: [system-design, storage, s3, media]
---

# Object Storage and Presigned URLs

S3-style object storage: flat key → blob, no filesystem semantics, effectively infinite capacity, HTTP-native. The design skill is keeping **large binary traffic off your application servers** — which is the entire architecture of my [OneTube](https://github.com/1Kyryll/OneTube) project (MinIO = self-hosted S3-compatible store).

## Fundamentals: object vs block vs file storage

| | Block (EBS, local SSD) | File (NFS, EFS) | **Object (S3, MinIO, GCS)** |
|---|---|---|---|
| Interface | raw blocks, mounted volume | POSIX filesystem, shared mounts | HTTP API: PUT/GET/DELETE by key |
| Structure | none (FS goes on top) | directory hierarchy | flat namespace; `/` in keys is a naming *convention*, not directories |
| Mutability | random read/write | random read/write | **objects are immutable — no partial update; replace the whole object** |
| Scale/cost | one machine's disks | limited scale, pricey | effectively unbounded, cheapest per GB |
| Fits | databases, boot volumes | legacy shared-FS apps | media, backups, static assets, data lakes |

Properties that shape designs on top of S3-style stores:
- **Immutability** is why my transcode worker's idempotency works as "overwrite the same keys" — there's no in-place edit to get half-done ([[Idempotency]]).
- **Metadata lives elsewhere.** Listing by prefix is possible but slow/paginated; there are no queries. Hence OneTube keeps *truth* in Postgres (`videos`, `video_assets` rows) and S3 holds only bytes — the DB indexes what exists, the store serves it.
- **Durability vs availability**: S3's famous "11 nines" durability comes from erasure-coding/replication across devices and zones; availability is a separate, lower number. Modern S3 (and MinIO) is strongly consistent read-after-write — historically it wasn't, and interviewers may still probe eventual-consistency handling.
- **Per-request cost model**: PUT/GET are billed operations — another reason "API relays every byte" is wrong twice (bandwidth *and* doubled request paths).

## The golden rule (from OneTube's design rules)

> **The API never touches video bytes.** Uploads go client → S3, segments go S3 → client, both via presigned URLs. Only tiny text manifests pass through the API.

An app server proxying video would burn its bandwidth, memory, and worker slots on dumb byte-shoveling. Presigned URLs delegate the transfer to the object store while the API keeps authorization control.

## Presigned URLs — how

A presigned URL is a normal S3 URL plus a **signature computed from the server's credentials**, scoped to one operation (PUT or GET), one key, and a TTL. The client uses it directly against S3; S3 verifies the signature. The API decided *who may do what*; S3 does the byte transfer.

[s3/presign.go](https://github.com/1Kyryll/OneTube/blob/main/server/internal/common/s3/presign.go):

```go
func (c *Client) PresignPut(ctx, key, contentType string, ttl time.Duration) (string, error) {
    u, err := c.Public.PresignedPutObject(ctx, c.Bucket, key, ttl)
    ...
}
func (c *Client) PresignGet(ctx, key string, ttl time.Duration) (string, error) {
    u, err := c.Public.PresignedGetObject(ctx, c.Bucket, key, ttl, url.Values{})
    ...
}
```

Note `c.Public`: the signing client uses the **browser-visible endpoint** (`S3_PUBLIC_ENDPOINT`), distinct from the internal Docker-network endpoint — signatures embed the host, so signing with `minio:9000` produces URLs a browser can't reach. A real deployment gotcha.

## Upload flow (direct-to-S3)
1. `POST /videos` → API inserts a `videos` row (`status=uploading`), returns a **presigned PUT** URL for `raw/{video_id}/original.{ext}`
2. Browser PUTs the file **straight to MinIO**
3. `POST /videos/{id}/complete` → status `processing`, transcode job to RabbitMQ ([[Message Queues and Background Workers]])

The DB row is created *before* the upload — the object store holds bytes, but **Postgres holds the truth** about what exists and its lifecycle state (`uploading/processing/ready/failed`).

## Serving a private bucket — HLS playback

The bucket is private; naive public-read would leak everything. OneTube's playback ([video handlers](https://github.com/1Kyryll/OneTube/blob/main/server/internal/video/handlers/http.go)):

1. hls.js fetches `GET /api/videos/{id}/hls/master.m3u8` — API relays the manifest from S3 (tiny text)
2. For each rendition playlist, the API **rewrites every `.ts` segment line into a presigned GET URL** (4h TTL)
3. hls.js then fetches segments **directly from MinIO** with valid signatures

Manifests (bytes: KB) flow through the API for access control; media (bytes: GB) flows direct. Key layout is content-addressed and predictable:

```
raw/{video_id}/original.{ext}        ← lifecycle-deleted 7d after transcode
hls/{video_id}/master.m3u8
hls/{video_id}/{quality}/playlist.m3u8
hls/{video_id}/{quality}/seg_NNN.ts
thumbnails/{video_id}/default.jpg
```

Predictable keys are also what makes the transcode worker idempotent — rerunning overwrites the same keys ([[Idempotency]]).

## Interview checklist
- Presigned PUT vs POST policies; TTL choice (short for uploads, hours for streaming sessions)
- **Lifecycle rules** — auto-delete raw uploads after N days (cost control, shown above)
- Soft-delete pattern: DB `deleted_at` first, async S3 object cleanup later (deleting millions of objects synchronously in a request is a non-starter)
- CORS on the bucket for direct browser access
- Multipart upload for files > ~100MB (name it as the next step)

## Related
- [[Message Queues and Background Workers]] — the transcode pipeline
- [[Idempotency]]
- [[REST API Design]]
- [[System Design MOC]]

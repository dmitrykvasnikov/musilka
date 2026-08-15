# 8. Images and media

**Priority:** P1

**What this section is not free to decide.** [1.4](01-product.md) singled images out as *the* thing
that scales with the catalogue rather than with users, predicted them as where the size estimate
breaks first, and pre-empted [8.2](08-media.md) by arguing for object storage and hard limits from
the start. [2.4.8](02-catalogue-model.md) already fixed the cap at eight per release and the type
vocabulary. [2.5.3](02-catalogue-model.md) already refused the Cover Art Archive and hotlinking, so
"who uploads" has exactly one possible answer. [6.3](06-accounts.md) added the avatar as a second
consumer and forbade it a parallel path. [11.3](11-stack.md) removed JavaScript from the toolbox
entirely. What is open is the byte budget, the variant set, the processor, and the takedown
mechanism — and the byte budget is the one decision here that is genuinely hard to reverse.

- [x] 8.1 Who uploads cover art, what are the limits (size, format, count per release).
      **Decision: any verified user, three accepted formats, 15 MB and 6000 px in, one normalised
      derivative pair out — and the uploaded original is not kept.**
      **Who: any user with a verified email**, which is the same bar [4.1](04-editing.md) set for
      editing and the barrier [6.1](06-accounts.md) made load-bearing. An image is an edit to a
      release; inventing a separate permission for it would be a fourth right in a three-role model
      ([4.7](04-editing.md)). Avatars are self-only, by construction.

      | | |
      |---|---|
      | Accepted formats | JPEG, PNG, WebP. Not HEIC (a decoder we would have to add), not GIF, not SVG (SVG is a script vector — never accept it as an image) |
      | Max upload | 15 MB, rejected at the request boundary before anything is decoded |
      | Max dimensions in | 6000 px on the longest side; larger is rejected, not silently downscaled |
      | Min dimensions in | 300 px on the longest side — below that it is not a sleeve scan |
      | Per release | 8 ([2.4.8](02-catalogue-model.md)) |
      | Per user | one avatar ([6.3](06-accounts.md)) |
      | Rate | a daily upload cap per user, in the service layer; the number belongs to [14.5](14-security.md) |

      **Format is decided by decoding the file, never by its extension or its declared MIME type.**
      The uploader's `Content-Type` is user input like any other.
      **The original is discarded once the derivatives exist, and this is the deliberate,
      irreversible one.** [1.4](01-product.md) estimated ~10 GB of images and named it as where the
      estimate breaks; the only levers we have are the number of objects and their size, and an
      original nobody ever fetches is pure cost in storage and, more to the point, in backup time
      ([9.3](09-nfr.md)). Keeping a 6000 px master would roughly triple the bucket to serve a
      re-encode we have no plan to perform.
      **The cost is real and is accepted:** we are not an archive, a future higher-resolution display
      is capped at 1600 px for everything uploaded before we change our minds, and the change is not
      retroactive. If it is ever reopened, it is "keep originals from this date", and older images
      simply do not have one.
- [x] 8.2 Storage: local filesystem / S3-compatible (MinIO, Backblaze, Cloudflare R2). (also hosts uploaded import files — see [10.4.8](10d-model-requirements.md))
      **Decision: S3-compatible object storage from the first image, behind a two-implementation
      interface — and it holds images only, because [10.2.6](10b-import.md) has already taken import
      files elsewhere.**
      **Not the local filesystem, and [1.4](01-product.md) already argued this.** Ten gigabytes on a
      small VPS is real money and real restore time, image bytes in a `pg_dump` are worse, and a
      filesystem path is the hardest thing to migrate off later because it leaks into every URL. This
      is the sole exception to the Postgres-is-the-only-datastore invariant (NOTES.md), and it was
      granted in advance.
      **Which provider is [12.1](12-infrastructure.md)'s to name.** The requirement is the S3 API and
      nothing beyond it: no signed-URL upload flow, no bucket events, no lifecycle rules, no
      versioning. Cloudflare R2 is the obvious candidate for having no egress charge, but the design
      does not depend on it.
      **Two implementations behind one small interface** (`put`, `get`, `delete`, `url`): the
      filesystem for local development and tests, S3 in production. That is not abstraction for its
      own sake — it is what keeps the test suite from needing a MinIO container in CI
      ([11.10](11-stack.md)).
      **Keys are content-addressed, which buys deduplication and permanent caching for free:**

      ```
      img/<sha256[0:2]>/<sha256>/full.webp
      img/<sha256[0:2]>/<sha256>/thumb.webp
      ```

      One `image(id, sha256, width, height, byte_size, created_at, uploaded_by)` row per distinct
      file, referenced by `release_image` ([2.4.8](02-catalogue-model.md)) and by
      `user.avatar_id` ([6.3](06-accounts.md)) — which is what that column was for. Two users
      uploading the same scan store one object. Objects are immutable, so
      `Cache-Control: public, max-age=31536000, immutable` is honest, and a blob is removed only when
      its last reference goes ([8.4](08-media.md)).
      **The image prefix is public-read and served directly from the bucket's own endpoint.** That is
      not a CDN and does not breach [1.4](01-product.md)'s veto: there is no second service, no cache
      invalidation story and no configuration beyond a bucket policy — and the alternative, proxying
      every image byte through one small box, is the thing the veto list is trying to protect. If the
      provider happens to front its endpoint with a CDN, that is the provider's implementation, not
      ours. [14.3](14-security.md) inherits one consequence: the bucket origin must be in `img-src`,
      and in nothing else.
      **The item's parenthesis is out of date and the bucket holds images only.** It points at
      [10.4.8](10d-model-requirements.md) for uploaded import files, but [10.2.6](10b-import.md) has
      since answered that item the other way: the file lives in `file_bytes` on the `import` row and
      is cleared when the job ends — *"nothing goes to [8.2](08-media.md)'s storage"*, in that item's
      own words. Nothing here reopens it, and the answer is the better one: [11.7](11-stack.md)
      requires a restartable importer, and a `bytea` column survives a replaced machine more
      certainly than an object we would then have to clean up, while
      [10.5.3](10e-legal-sources.md)'s personal data never lands in a bucket whose other prefix is
      public. **The one thing this section owes that decision is a warning:** the file is up to
      [8.1](08-media.md)-sized megabytes of `bytea` inside every `pg_dump` taken while an import is
      running, which is [9.3](09-nfr.md)'s to notice, not a reason to move it.
- [x] 8.3 Processing: resizing, thumbnails, WebP/AVIF, CDN.
      **Decision: two WebP variants, produced synchronously by `vips` as a subprocess. No AVIF, no
      CDN, no original.**

      ```
      full    longest side 1600 px, WebP q82   ~150–300 KB   the release page
      thumb   square 300 px, cropped centre, WebP q80   ~15–25 KB   grids, lists, avatars
      ```

      **Two variants, not four.** Every additional size multiplies the object count and the backup by
      the number of images in the catalogue, and [1.4](01-product.md) says that is the number to
      watch. A `srcset` with two entries is enough for the layouts [7.6](07-search-ux.md) describes.
      **WebP only, with no JPEG fallback**, which is a browser-support bet that
      [9.6](09-nfr.md) has to confirm rather than one this section can make alone — every current
      browser has supported it for years, and a fallback set would double storage to serve nobody.
      **No AVIF.** A second encode, doubled objects and encode times an order of magnitude longer, for
      a marginal size gain on files that are already small.
      **`vips` (via `vipsthumbnail`) as a short-lived subprocess, not an in-process library.**
      [11.3](11-stack.md) left "in-process or a system binary" open and this is the case for the
      binary: an image decoder is the classic memory-exhaustion and memory-corruption surface, and a
      separate process with a wall-clock and memory limit contains a malicious file in a way a
      library linked into the web server cannot. `vips` also handles the parts that are easy to get
      wrong by hand — correct downscaling, EXIF auto-rotation, colour profiles.
      **The honest cost: this is the first runtime dependency outside Postgres**, and
      [11.11](11-stack.md) is working to keep the deploy at one artifact. It is one system package,
      and [section 12](12-infrastructure.md) inherits installing it and pinning its major version.
      **EXIF is stripped, after auto-rotation.** A phone photo of a sleeve carries GPS coordinates of
      the owner's home, which is [3.7](03-collection.md)'s worry in a different file format.
      **Processing is synchronous, in the upload request** — and this contradicts what NOTES.md
      recorded as an expectation ("image derivatives" as a queue consumer), the same way
      [10.3.3](10c-export.md) contradicted [11.7](11-stack.md) on export. Two `vips` invocations and
      two `PUT`s is on the order of a second; the user is already waiting, having just chosen a file.
      A queued derivative would mean an `image` row with nothing renderable behind it, which is a
      placeholder state in every template that shows a cover, plus a poll or a refresh to resolve it —
      real complexity in the UI to save a second in the request. **The queue's users remain the
      importer and outbound email**, and that is now the complete list.
      **No CDN** ([1.4](01-product.md)). Immutable content-addressed keys plus a one-year
      `Cache-Control` are the entire caching story, and they are better than a CDN with a purge
      problem.
- [x] 8.4 Upload moderation (NSFW/copyright), takedown procedure on complaint.
      **Decision: no automated screening; reactive removal through [4.6](04-editing.md)'s report row,
      and a takedown that is a genuine hard delete with a hash block behind it.**
      **No NSFW classifier, at any stage.** Every option is either an external API call — which
      [1.5](01-product.md) forbids for catalogue data and which we decline to introduce for images
      either — or a model to host on a box [1.4](01-product.md) sized at 2 GB. Against that: sleeve
      art is *routinely* nude, so the false-positive rate on exactly our content would be the highest
      available, and at tens of users the volume is a handful of uploads a week that one person can
      look at.
      **Reporting reuses [4.6](04-editing.md) unchanged.** That item already fixed the `report` table
      with `target_type`/`target_id` and listed "wrong image" and "offensive content" among the
      expected reasons; an image is another target type and needs no new mechanism. Consistent with
      [5.7](05-messaging.md), a report writes a row and emails the admin — there is no queue in the
      MVP ([1.9](01-product.md)).
      **Takedown is the only hard delete of something other people reference, and the exception is
      principled.** The invariant is that such things are never hard-deleted (NOTES.md) — a user's own
      collection rows always were ([3.10](03-collection.md)) — but that rule exists to stop data
      disappearing under someone who points at it, and a takedown that leaves the file served defeats
      its entire purpose. So: the `release_image` row goes, and when the last reference
      to a blob goes, the objects are deleted from the bucket. What survives is an `image` row reduced
      to its `sha256` and a `blocked_at` — and because [8.2](08-media.md) made storage
      content-addressed, **re-uploading the same file is refused by hash for free.** That is a
      surprisingly strong answer to the repeat-uploader case and it cost one column.
      **The catalogue entity is untouched.** Removing an image is not deleting a release
      ([4.5](04-editing.md)); the release loses a picture and keeps everything else, and if the
      removed image was `is_primary` the next by position becomes primary.
      **Who may:** moderators and admins ([4.7](04-editing.md)), plus the uploader removing their own
      image before anyone reports it. A rights-holder is not a user and does not need an account —
      which is why the mechanism is an address.
      **The procedure, in the form it has to take:** a `/terms` page names an address, a complaint
      naming the URL gets the image removed first and discussed afterwards, and the removal and its
      reason are recorded in [4.10](04-editing.md)'s audit log.
      **The policy text belongs to [13.5](13-legal.md), which this item now blocks on nothing and
      hands a mechanism to.** Two things should be stated there and not here: that user-uploaded
      sleeve scans are the rights-holder's work and our position rests on being small,
      non-commercial ([1.7](01-product.md)) and prompt, and that [10.5.4](10e-legal-sources.md)'s
      point holds — importing a collection legalises nothing about images, and nothing in
      [10.2](10b-import.md) uploads one.
- [x] 8.5 Audio previews of tracks — in scope? (legally risky, links to streaming services are safer)
      **Decision: no, permanently — and no links to streaming services either.**
      **Hosting audio is not a close call.** It is a licence we cannot obtain, at a per-object size
      that dwarfs the image budget [1.4](01-product.md) already flagged as the breaking point, for a
      catalogue of *physical objects* ([1.2](01-product.md)) — the product is what is on the shelf,
      not what it sounds like.
      **The "safer" alternative in the item is also refused, and this is the part worth recording.** A
      streaming link needs a Spotify or Apple identifier. That identifier comes from either an
      external API call, which [1.5](01-product.md) forbids and [2.5.6](02-catalogue-model.md)
      already refused by name, or from hand entry — which means a field that is empty on essentially
      every row, rots as catalogue entries move, and quietly asks our editors to do a third party's
      data entry. [2.5.3](02-catalogue-model.md) refused hotlinked images for the same reason it
      applies here: our catalogue's completeness must not depend on someone else staying up.
      Nothing is reserved in the schema. If it ever returns it is a row in
      [10.4.1](10d-model-requirements.md)'s external-ID table with a new source, which needs no
      migration.

## Working notes

- **2026-08-15 — Section closed. All 5 items decided**, and three of them were largely pre-decided by
  [1.4](01-product.md), [2.4.8](02-catalogue-model.md) and [2.5.3](02-catalogue-model.md). The two
  that were genuinely open — the byte budget ([8.1](08-media.md)) and the processor
  ([8.3](08-media.md)) — are also the two with a cost attached, and both are stated as costs above
  rather than presented as free.
- **2026-08-15 — Schema delta implied by the decisions above.** One table:

  ```
  image(id, sha256 unique, width, height, byte_size,
        uploaded_by, created_at, blocked_at)
  ```

  `release_image` ([2.4.8](02-catalogue-model.md)) and `user.avatar_id` ([6.3](06-accounts.md))
  reference it; neither changes shape. No variants table — the two derivatives are implied by the key
  scheme, and adding a third size later means generating it, not migrating.
- **2026-08-15 — Revised size estimate, against [1.4](01-product.md)'s.** That item projected
  ~12,000 releases × 2 images × 400 KB ≈ 10 GB and expected it to break first.
  [8.1](08-media.md)'s discarded original and [8.3](08-media.md)'s WebP change the arithmetic:
  ~250 KB `full` plus ~20 KB `thumb` is **~270 KB per image, so ~6.5 GB at the same counts**, and
  content-addressed deduplication takes a further bite where two users upload the same scan.
  It is still by far the largest thing we store and still the number to watch, but object storage at
  cents per gigabyte moves the pain from cost to **backup and restore time**, which is
  [9.3](09-nfr.md)'s problem and not [1.4](01-product.md)'s cost problem. Worth noting that the
  bucket needs a backup story of its own — `pg_dump` does not cover it, and a restored database
  pointing at a lost bucket is a catalogue of broken images.
- **2026-08-15 — What this section hands on.**
  [Section 12](12-infrastructure.md) inherits the most: an object storage provider and bucket
  policy to choose ([12.1](12-infrastructure.md)), `vips` as the first non-Postgres runtime
  dependency to install and pin ([8.3](08-media.md)), and a filesystem implementation for local
  development so `docker compose` ([12.2](12-infrastructure.md)) needs no MinIO.
  [Section 9](09-nfr.md) inherits the bucket backup question ([9.3](09-nfr.md)) and the upload
  request as its own latency case ([9.1](09-nfr.md)) — a second is fine, ten is not — plus
  [9.6](09-nfr.md) confirming WebP with no fallback.
  [Section 13](13-legal.md) inherits [13.5](13-legal.md)'s rights-holder procedure, for which
  [8.4](08-media.md) has now built the mechanism, and the address that has to appear on `/terms`.
  [Section 14](14-security.md) inherits three concrete items: the bucket origin in `img-src` and
  nowhere else, SVG rejected outright, and decode-in-a-subprocess as the containment for a hostile
  file ([14.3](14-security.md)); [14.5](14-security.md) inherits the per-user daily upload cap.
  [10.4.8](10d-model-requirements.md) inherits nothing: [10.2.6](10b-import.md) answered it before
  this section was opened, and the parenthesis in [8.2](08-media.md)'s own question text is stale.
  The bucket holds images and nothing else.
- **2026-08-15 — Where this lands in the roadmap.** Images are inside [1.9](01-product.md)'s MVP and
  are the one part of it that adds infrastructure, so the storage interface and its filesystem
  implementation want to exist early even though the S3 side does not. Order: `image` table and the
  filesystem driver, then release upload, then the S3 driver at first deploy, then the avatar
  ([6.3](06-accounts.md) already places it after the catalogue's images), then
  [8.4](08-media.md)'s removal path — which can be a moderator-only `DELETE` route long before there
  is any report UI to reach it from.
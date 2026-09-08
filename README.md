# ota-site — the content host for the Pillar app

Served from a **separate** public repo (`makslearnsprompts/pillar-ota`) at
**https://ota.glowd.tech**, deployed with `scripts/deploy-ota-site.sh`.

```
ota.glowd.tech/
  version.json          which onboarding bundle is live right now  ← the only file that changes
  codes.json            sha256 hashes of valid promo codes
  b/<version>/          one immutable directory per published bundle
    manifest.json         every file with its own sha256 + byte count
    index.html  app-*.js  app-*.css  assets…
```

Two things fetch from here, and nothing else does:

| file | client | when |
|---|---|---|
| `version.json` → `b/<v>/manifest.json` → the files | `OnboardingBundleUpdater` | detached, on launch |
| `codes.json` | `PromoStore` | only when someone types a code |

## Why this is not part of `pillar-site/`

`pillar-site` is the marketing site: a human writes a sentence, a human reads it.
This is a payload the app depends on at runtime. Keeping them apart buys three
things, in order of how much they would hurt:

1. **`deploy-pillar-site.sh` runs `rsync --delete`.** A marketing deploy from a
   checkout that happened not to have `b/` would silently remove every published
   bundle and the `version.json` pointing at them. Different repos means a copy
   edit can never reach the bundles.
2. **Weight.** Each bundle is ~1.9 MB of build output, and every publish adds
   another one to a git history forever. That does not belong in the history of
   four legal pages.
3. **Cadence.** The funnel is published on its own schedule, sometimes several
   times a day; the site changes when the pricing does.

## Publishing

```
scripts/publish-onboarding.sh     # build web-onboarding → b/<version>/ + repoint version.json
scripts/deploy-ota-site.sh        # push this folder to the pages repo
```

Version ids are **immutable** — publishing never overwrites `b/<v>/`, it adds a
new one and repoints `version.json`. That is why a device that has already
downloaded a version can never have it change underneath it.

## Rolling back

**Editing `version.json` back to an older id does not roll anybody back.** The
client asks one question, `candidate > local` — a plain string compare — so a
device that already took the bad version sees the older id as "not newer" and
keeps what it has. Verified on device, 2026-09-08. Repointing only helps
installs that have not checked yet, which in an incident is the half you were
not worried about.

The rollback that works republishes the good payload under a **higher** id:

```
scripts/publish-onboarding.sh --rollback-to 2026-09-08-546e233
scripts/deploy-ota-site.sh
```

It copies that bundle to `<current live id>-rN` — a suffix on a common prefix
always sorts above it, so the result beats both what is live and what the
affected devices are holding — regenerates the manifest, and repoints
`version.json`. Payload byte-identical, id new.

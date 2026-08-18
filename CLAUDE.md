# Merchant API docs — repo rules

Markdown guide content for the GitBook site **apidocs.kumaagroup.com**.
GitBook syncs this repo (`SUMMARY.md` defines the page tree); merchants read
the published site, not this repo.

## Where content comes from

- Guide pages: `docs/*.md` here, published under `/readme/<file-slug>`.
- API reference: rendered from the **org-level GitBook spec** `merchants-api`,
  uploaded by `platform`'s "Push OpenAPI spec to GitBook" workflow from
  `openapi/merchants/merchants.yaml`. **This repo holds no OpenAPI file — do
  not add one.** Endpoint pages, tag names, and their grouping (the
  `Partner API` / `Merchants API` sections via `x-parent` tags) are all edited
  in `platform`, not here.
- Reference page URLs: `/merchants-api/<tag-slug>` and `/partner-api/<tag-slug>`.

## Anchor links: GitBook slugs ≠ GitHub slugs

GitBook generates heading ids differently from GitHub, and a wrong fragment
fails **silently** — the link renders normally and clicking scrolls nowhere
(empty browser console). Never judge a link by how it renders.

- Punctuation (em-dashes included) collapses into a **single** hyphen:
  `## Step 1 — Initialize a Crypto Payment` → `#step-1-initialize-a-crypto-payment`
  (GitHub would produce `#step-1--initialize-a-crypto-payment` — wrong here).
- Digit-leading headings get an **`id-` prefix**:
  `## 3D Secure (3DS)` → `#id-3d-secure-3ds`. GitBook auto-repairs *same-page*
  digit fragments but passes *cross-page* fragments through verbatim, so
  cross-page links must be written in the `id-` form.

## Verify every anchor against the live site

After adding or renaming headings/links, check fragments against the ids
GitBook actually generated (they may lag until the next sync):

```bash
curl -sL https://apidocs.kumaagroup.com/readme/<page> \
  | grep -oE 'href="#[^"]+"' | sort -u
```

Every `](#fragment)` and `](page.md#fragment)` in `docs/` must appear in the
target page's list.

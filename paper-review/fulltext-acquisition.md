# Full Text Acquisition — shared reference (paper-review + paper-digest)

Single source of truth for getting a paper's full text. Both `/paper-review` and `/paper-digest`
Phase 0 point here; skill-specific policy (what to do when acquisition fails) stays in each SKILL.md.
Routes marked *(config)* depend on `config.yaml` values — a blank value disables that route.
Update THIS file when a route changes — don't re-fork into the skills.

## Check order (stop at first success)

1. **User-provided** PDF path / pasted full text → Read tool or `/markitdown`.
2. **`${fulltext.inbox_dir}`** *(config)* — an upstream step may have dropped the PDF here.
   ⚠️ When a batch driver (e.g. a sync skill) runs, it pre-fetches full text **serially** and
   hands you a path — so this route usually hits, and you should **never drive a browser yourself**.
3. **Zotero MCP** *(config: `fulltext.zotero_local_api`)* — `search_items` by DOI →
   `get_item_children` (PDF attachment? cheap check first) → only if PDF exists,
   `get_item_fulltext` (~10K+ tokens, call once). Skip this route if `zotero_local_api: false`.
4. **Open-access auto-fetch** (see rule below) — always on, no config needed.
5. **Elsevier TDM** (DOI `10.1016/...`) *(config: needs a TDM key via `secrets.backend`)* — see below.
6. **Automated institutional downloader** *(config: `fulltext.library_script`)* — **the primary
   route for anything paywalled-but-subscribed**. It logs into your library's proxy by itself
   (stored credentials + offline CAPTCHA solve); no manual login, no human in the loop:
   ```
   python ${fulltext.library_script} fetch <DOI> <out.pdf>
   ```
   Exit `0` = PDF written (validate `%PDF`). `2` = no route / auth failed. `4` = another fetch
   holds the browser profile. `5` = watchdog abort. **Treat 4 and 5 as "retry serially later",
   not as "no full text"** — they are contention or a hang, not absence.
   - **Serial by design**: one exclusive browser profile. **Never run it from parallel agents** —
     concurrent calls queue and then fail. Batch callers must fetch in a serial pre-pass.
     Blank `library_script` → skip. Reference implementation:
     [paper-fetch](https://github.com/drpwchen/paper-fetch).
   - A hit here is common even when the link resolver *claims* no subscription — trust the
     downloader's actual result over the resolver's flag.
7. **Institutional link resolver** *(config: `fulltext.sfx_base`)* — **last resort, manual.**
   Only after route 6 is unavailable (blank config) or returned exit `2`. Build the OpenURL from
   `sfx_base` (substitute `{DOI}`) → Claude-in-Chrome. Blank `sfx_base` → skip.
   - ⚠️ This route can dead-end at an interactive login/CAPTCHA page an agent cannot pass. **Do
     not burn turns retrying it** — report "full text unavailable" instead. If you opened a
     browser session, **close it before you finish** (leaked tabs hold memory).

> **Failure mode worth knowing (observed 2026-07-10).** In a batch run, agents skipped route 6,
> hit route 7's login wall, and each concluded "no full text" — for papers whose PDFs the
> downloader could fetch in seconds. Others ran route 6 concurrently and deadlocked on the shared
> browser profile. Route 6 is primary, it self-serializes, and it fails bounded (exit 4/5).
>
> Also: Unpaywall reporting `is_oa: true` does **not** guarantee a PDF exists — hybrid and
> ahead-of-print articles routinely report OA while offering no usable `url_for_pdf`.

## Open-access = always auto-fetch, never ask (CRITICAL)

Whenever metadata shows the paper is OA, fetch full text automatically — do NOT stop at
abstract-only and do NOT ask the user to download manually. OA signals:
- PMC ID in Semantic Scholar `externalIds` → `WebFetch https://pmc.ncbi.nlm.nih.gov/articles/PMC{id}/`
- Semantic Scholar `openAccessPdf.url` non-empty (GOLD/HYBRID/GREEN)
- Europe PMC / DOI resolves to free full-text HTML

Zotero `add_by_doi` often FAILS to grab the PDF from publisher sites (e.g. Wiley) even when a
PMC OA copy exists — "no PDF in Zotero" does NOT mean abstract-only. Always check PMC /
openAccessPdf next.

## Elsevier / ScienceDirect (DOI 10.1016/...)

ScienceDirect blocks all automated fetches (Cloudflare CAPTCHA; WebFetch/curl/Jina → 403), even
for gold-OA. Working route = Elsevier TDM Article Retrieval API:
- `python ${fulltext.elsevier_script} <DOI> <out.txt>` (bundled `elsevier_fulltext.py`).
- Reads `ELSEVIER_TDM_KEY` per `${secrets.backend}` (dpapi / env); never prints the key. If not
  stored, the script prints the one-line command to set it (Claude must NOT ask for/read the key).
  `secrets.backend: none` → this route is unavailable; skip.
- Gold-OA → full text with key alone (`view=FULL`). Only metadata back (<1500 chars) → an
  institutional token would be needed; don't chase it unless the user asks.

## Zotero bookkeeping (do alongside acquisition)

- Not in Zotero → `add_by_doi` (creates item + BBT citekey for frontmatter). If it reports no PDF
  but an OA link exists → auto-fetch per OA rule; don't ask.
- Zotero not running → don't block; note "Zotero pending" and continue (paper-sync drains later).
- Citekey: `search_by_citation_key` (format `authorShortTitle20XX`) or
  `curl -s "http://localhost:23119/better-bibtex/export/library?/1/library.bibtex" | grep {doi}`.

## fitz / PDF-extraction caveat

Any locally-extracted PDF text must be sanity-checked for silent extraction failure (font-encoding
scramble, near-zero character density, expected sections missing) before being trusted — a garbled
extract produces a confident-but-wrong digest.

# Changelog

All notable changes to these skills are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.2.1] — 2026-07-14

### Changed
- **Repository renamed** `claude-paper-tools` → **`paper-review-and-digest`**, so the name says what
  the two skills actually do instead of who built them. The old GitHub URL redirects permanently, so
  existing clones, links, and stars keep working; no code, config key, or skill name changed.
- Cross-link lines in this repo, `paper-radar`, and `paper-fetch` updated to the new name.

## [0.2.0] — 2026-07-10

### Changed
- **Full-text acquisition ladder reordered** (`paper-review/fulltext-acquisition.md`, the shared
  source of truth for both skills). An **automated institutional downloader** is now the primary
  route for paywalled-but-subscribed papers (new config key `fulltext.library_script`; reference
  implementation: [paper-fetch](https://github.com/drpwchen/paper-fetch)). The manual SFX /
  link-resolver route is demoted to **last resort**, and explicitly flagged as an interactive
  wall an agent cannot pass.
- Documented the downloader's exit codes and, importantly, that **`4` (busy) and `5` (watchdog)
  mean "retry serially" — not "no full text"**. Conflating contention with absence is the easiest
  way for a batch run to wrongly mark papers unobtainable.
- Agents are now told **not to drive a browser themselves** when a batch driver has already
  pre-fetched full text, and to **close any browser session they open**.

### Added
- Warning that Unpaywall's `is_oa: true` does not guarantee a PDF exists — hybrid and
  ahead-of-print articles routinely report OA while offering no usable `url_for_pdf`.
- Cross-links to the full three-repo pipeline (paper-radar → paper-fetch → claude-paper-tools).

### Why
A 12-paper batch run on 2026-07-10 exposed the old ladder's failure mode: agents skipped straight
past the automated downloader (it wasn't in the ladder at all), hit the resolver's login wall, and
each independently concluded "no full text" — for papers whose PDFs were one serial fetch away.
Agents that *did* reach for the downloader ran it concurrently and deadlocked on its exclusive
browser profile. Both failure modes are now designed out.

## [0.1.0] — 2026-07-09

Initial public release: `/paper-review` (journal-club appraisal with a deterministic GRADE
judge and a CrossRef reference-existence gate) and `/paper-digest` (fast content absorption
with self-test review cards), sharing one config and one full-text acquisition reference.

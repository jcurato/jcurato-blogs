# jcurato — project brief

Technical blog for J Curato Solutions. Written by John, a DevOps engineer in
Chennai. This is the content arm of a consulting and training venture.

## Purpose

Publish deep, specific posts drawn from real production experience. The blog
feeds a DevOps/AI training offering — first a small paid cohort, later broader.
Content and teaching material are built together, not sequentially.

## Audience

Advanced beginners: engineers with 1–3 years of experience who have already
touched a cluster but don't know what to do when something breaks in
production. They know `kubectl` and can follow a tutorial. They cannot debug a
log pipeline.

Not raw beginners (that market is saturated and competes on price, not
expertise). Not experts (they solve their own problems and don't buy courses).

Writing rule: **do not explain the basics.** Don't explain what `kubectl` is;
do explain buffering behaviour. Assume the reader already has the fundamentals.
Explaining basics both lengthens posts and dilutes what makes them distinctive.

## Editorial angle

The edge is production experience, not tutorial coverage. Every post should
contain something that cannot be found in official documentation — a real
failure, a wrong first assumption, a trade-off discovered the hard way.

Preferred post shape:

1. The symptom, as someone actually reported it
2. The first wrong assumption (keep this — it builds trust)
3. What is really happening, mechanically
4. How to confirm it is your problem
5. Fixes in order, with trade-offs stated
6. What remains unsolved — say so honestly

## Course outline (posts map to these modules)

1. Cluster foundation — GKE setup, node pools, networking, why defaults fail
2. Identity and access — Workload Identity, service accounts, secretless access
3. How a cluster talks — logs vs metrics vs traces, and when each applies
4. Log pipeline — Fluent Bit architecture: DaemonSet, buffering, parsing
5. **Where logs get lost** — short-lived pods, rotation races, buffer overflow
6. Metrics and alerting — what to measure, avoiding alert fatigue
7. Cost and scale — sampling, retention, controlling the observability bill
8. Debugging a real incident — everything above, applied end to end

Module 5 is the signature topic and the first post. Do not write these in
order; start with the hardest and most distinctive material.

## Tech stack

- **Astro** (blog template), Markdown content in `src/content/blog/`
- **pnpm** for dependency management — `npm install` in this project hangs/
  crashes silently on this machine for reasons still unexplained; pnpm works
  reliably. Use `pnpm install`, `pnpm run dev`, `pnpm run build`.
- **GitHub** — repo under the `jcurato` organisation (https://github.com/jcurato), public
- **Cloudflare Pages** — build `pnpm run build`, output `dist`, deploy on push
- **Cloudflare Registrar** — `jcurato.com`, auto-renew on, WHOIS privacy on
- No server, no runtime, no database. Static output served from CDN.

Commit identity is set per-repo to a personal email, not a work address.

## Content collection schema

Defined in `src/content.config.ts`. Frontmatter fields: `title`,
`description`, `pubDate` (not `date`), optional `updatedDate`, optional
`heroImage`. Posts live in `src/content/blog/*.md` or `*.mdx`.

## SEO basics

- Post titles use the words people actually type. Good: "Fluent Bit missing
  logs from short-lived pods in GKE". Bad: "Observability Deep Dive Part 1".
- URLs without dates: `/blog/fluent-bit-short-lived-pods`
- Sitemap and meta descriptions come from the Astro blog template
- Google Search Console registered from day one — the search queries that bring
  people in are the real signal for who the audience is

## Working rules

- **Setup is capped at one day.** Default theme is fine. Ship the first post
  before touching design.
- Content is a byproduct of teaching material, not a prerequisite for it.
- Deadline over metrics: announce the first cohort on a fixed date regardless
  of traffic. Waiting for an audience number is a form of postponement.

## Status

- [x] Domain available and identified: `jcurato.com`
- [x] First post drafted: `fluent-bit-short-lived-pods.md`
- [x] Domain registered (`jcurato.com`, registered via Cloudflare under `jcuratosolutions@gmail.com`)
- [x] Astro project initialised
- [x] GitHub org created (`jcurato`)
- [x] GitHub repo created and pushed (`jcurato/jcurato-blogs`)
- [ ] Cloudflare Pages connected
- [ ] First post published
- [ ] Google Search Console verified

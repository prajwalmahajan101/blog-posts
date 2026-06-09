# blog-posts

Source markdown for blog posts I publish, across platforms. Each platform has its own subdirectory so the frontmatter format and slug conventions don't collide.

## Layout

```
.
├── hashnode/   # posts on https://prajwalmahajan101.hashnode.dev
└── devto/      # posts on https://dev.to/prajwalmahajan101
```

## hashnode/

Each `cl*.md` filename is the Hashnode-generated post ID. Frontmatter follows Hashnode's format (`slug`, `seoTitle`, `seoDescription`, `datePublished`, …). Hashnode is the source of truth for what readers see; the files here exist so drafts and edits are version-controlled.

| File | Title | Published |
|---|---|---|
| `clwc362qc000i0albet2lhzbc.md` | Understanding Git and Version Control Systems (VCS) | 2024-05-18 |
| `clwk969t2000h09js9rwk1g5p.md` | SOLID Strategies Unleashed | 2024-05-24 |
| `clwvz2ua9000709jq86ap1xhz.md` | Introduction to Clean Code | 2024-06-01 |
| `cly4ox23y000409lcdsdh3r2a.md` | Mastering Clean Code: Effective Naming Conventions for Developers | 2024-06-30 |

## devto/

dev.to posts. Filenames are slug-derived. Frontmatter follows dev.to's format (`published`, `tags`, `series`, `cover_image`, `canonical_url`). Article IDs and `published_at` timestamps are kept as comments inside each file's frontmatter for traceability.

| File | Title | Series | Published | Live |
|---|---|---|---|---|
| `building-toymq.md` | Building toymq: a from-scratch persistent message broker in Go | Building distributed systems · #1 | 2026-06-09 | [dev.to](https://dev.to/prajwalmahajan101/building-toymq-a-from-scratch-persistent-message-broker-in-go-ob7) |

## Workflow

- Drafts land here on a feature branch first.
- For Hashnode: paste into the Hashnode editor or use Hashnode's GitHub sync; commit the published file back here under `hashnode/` with the platform-assigned ID as filename.
- For dev.to: publish via the dev.to API (`PUT /api/articles/<id>`) and copy the canonical body here under `devto/`. The article ID + `published_at` go into a comment block in the file's frontmatter.
- Edits and corrections land back here as PRs first, then get re-pushed to the platform.

## Why split by platform

Each platform has its own frontmatter dialect, slug convention, and rendering quirks (mermaid support, code-block highlighting, image hosting). Mixing them in one directory loses the round-trip property — a file from `hashnode/` can't be re-published to dev.to without rewriting half the frontmatter. Splitting by platform keeps each file directly usable by its publisher.

## License

CC-BY-4.0 (see `LICENSE`). Attribution required, modifications welcome.

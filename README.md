# Remote AI Platform — Internal Documentation

This is the private companion repository to
[gokul-227/remote-ai-platform](https://github.com/gokul-227/remote-ai-platform),
which holds all of the product's source code. This repo holds everything else:
how the product is built, deployed, operated, and used.

**Start here: [`handbook/00-overview.md`](handbook/00-overview.md)**

## What's in this repo

Just one folder, [`handbook/`](handbook/) — a complete, current-as-of-today
handbook covering both the technical and business/product sides of the
product. See [`handbook/06-repository-guide.md`](handbook/06-repository-guide.md)
for the full table of contents and why the two repos are split this way.

There is nothing else here on purpose: no historical archive, no dated
one-off reports, no credential values. Real secret values live in
[Infisical](https://app.infisical.com) (see
[`handbook/04-secrets-and-credentials.md`](handbook/04-secrets-and-credentials.md)),
never in this git repository.

## Keeping this current

Changes are committed directly to `main` — there's no pull-request process
here, since this is documentation and operational record, not shipped product
code. When something in the product changes, update the relevant file in
`handbook/` in place rather than adding a new dated document alongside it —
this repo stays useful by staying current, not by accumulating history.

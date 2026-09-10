This directory contains the InterruptingCow documentation.

Start at [`index.md`](index.md).

The documents are plain GitHub-flavored Markdown so they render in the repository
browser. The directory is structured so a documentation-site generator (a
Docusaurus sidebar, or MkDocs `nav`) can be layered on top later without moving
or rewriting content.

## Placeholders

Until the repository has a permanent home, links to GitHub features use
`https://github.com/OWNER/InterruptingCow/...`. Replace `OWNER` with the real
account or organisation when the repo is created:

```sh
grep -rl 'OWNER/InterruptingCow' . ..   # find them
```

Personal details that aren't known yet (maintainer handle, security contact, PGP
key) are marked with an explicit `**TODO**` in the file that needs them —
`grep -rn TODO` to list them. These must be filled in before the repository is
made public.

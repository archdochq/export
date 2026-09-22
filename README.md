# archdochq/export

[![Use this action](https://img.shields.io/badge/GitHub_Marketplace-Use_this_action-2088FF?logo=github&logoColor=white)](https://github.com/marketplace/actions/archdoc-export)

Runs [ArchDoc](https://github.com/archdochq/archdoc)'s export over a
specification repository, so a website, a search index or anything else has the
whole repository as JSON without reimplementing the front matter schema, the
identifier rules or the reverse-relationship graph.

```yaml
- uses: actions/checkout@v7
- uses: archdochq/export@v1
- uses: actions/upload-artifact@v4
  with:
    name: spec
    path: spec.json
```

## Why not just run it

Because of the revision. `archdoc export` records `commit` only when it is
given one, and deliberately never asks git: a working tree holding uncommitted
edits would describe something other than the output. A workflow is the one
place the revision is known and the tree is clean, so the action records the
commit the run is for. Given an empty `commit`, it records none, which is what
a committed export tree wants: the field would otherwise change on every run
and the output would stop being worth diffing.

It also settles where the output lands. `archdoc` resolves `--out` after `-C`
has taken effect, so a `path` of `spec` and an `out` of `dist` writes to
`spec/dist`, while a redirect in the calling shell does not. Both `file` and
`out` are relative to the workspace here.

`version` behaves as it does in [archdochq/lint](https://github.com/archdochq/lint):
left empty, whatever `archdoc` is on `PATH` is used, so a workflow that built
the binary from the commit under test keeps the one it built.

## The two forms

One document, the default, is the form an ingestion endpoint or a single
`fetch` wants:

```yaml
- uses: archdochq/export@v1
  with:
    path: spec
    source: true
```

A tree is the form a site that renders one document per page wants, and is
`index.json` carrying every document without bodies, one file per document
beside it, and `glossary.json` when there is a glossary:

```yaml
- uses: archdochq/export@v1
  with:
    out: dist
```

## Publishing to archdoc.dev

Given a token, the export is posted to archdoc.dev as it is written:

```yaml
jobs:
  publish:
    runs-on: ubuntu-latest
    # The endpoint does not restrict which branch may publish, so the secret
    # lives on an environment that does.
    environment: publish
    steps:
      - uses: actions/checkout@v7
      - id: export
        uses: archdochq/export@v1
        with:
          source: true
          publish-token: ${{ secrets.ARCHDOC_PUBLISH_TOKEN }}
```

The endpoint answers as soon as it has the file, before anything has read it,
so the `snapshot` output is an id that was accepted rather than a snapshot that
ingested. A green step means the upload arrived.

`source` is worth setting. archdoc.dev renders from `contents` either way and
keeps `source` only for a raw view of the document, so leaving it off loses
that view and nothing else.

`out` cannot be combined with a token: what gets published is the single
document, and the action refuses the pair before it runs the export rather than
after.

The token is passed to `curl` through a configuration file on standard input
rather than a `-H` argument, so it is not in any process's arguments.

## Inputs

| Input | Default | |
| --- | --- | --- |
| `version` | *(empty)* | The archdoc release to use |
| `path` | `.` | The directory to run in, as `archdoc -C` takes it |
| `file` | `spec.json` | Where the single-document export is written |
| `out` | *(empty)* | A directory to write a tree into instead, ignoring `file` |
| `commit` | `${{ github.sha }}` | The revision the export describes. Empty records none |
| `source` | `false` | Include each document's file as it is on disk |
| `no-bodies` | `false` | Omit every contents body, keeping the outline |
| `publish-token` | *(empty)* | Publish to archdoc.dev, as a bearer token |
| `publish-url` | `https://archdoc.dev/api/publish` | The endpoint to publish to |

A `pull_request` run is for the merge commit, and that is what `commit` then
carries. Pass `${{ github.event.pull_request.head.sha }}` to record the branch
instead.

## Outputs

| Output | |
| --- | --- |
| `file` | The single-document export that was written. Empty when `out` was set |
| `directory` | The tree that was written. Empty unless `out` was set |
| `documents` | How many documents the export carries |
| `snapshot` | The snapshot the endpoint accepted. Empty unless published |

## Exit status

The action exits with `archdoc export`'s own status: 0, or 1 when the
repository could not be read. A refused publish also exits 1, with whatever the
endpoint said in the log. Export validates nothing, so a repository with
findings still exports; run [archdochq/lint](https://github.com/archdochq/lint)
for that.

The output is deterministic apart from `commit`. Nothing in it records when it
was generated, so it can be cached, diffed and committed.

## Licence

MIT. ArchDoc itself is AGPL-3.0-or-later; this action only runs a published
release and imposes nothing on what you run it against.

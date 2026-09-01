# ofplang spec

The specification for **Object-flow Programming Language v0** — a YAML-based
dataflow workflow IR with linear Object tracking.

- [SPECIFICATION.md](SPECIFICATION.md) — the specification, revision **0.1**.
- [CHANGELOG.md](CHANGELOG.md) — what changed between revisions, and what a
  document written against an earlier one has to do about it.

This repository is the single source of truth for the language. Tools in the
[`ofplang`](https://github.com/ofplang) organization (validator, scheduler,
runtime, …) target this spec.

A revision is tagged here (`v0.0`, `v0.1`, …), and a document names the revision
it is written against with `spec_version`. An implementation states the revision
it implements — `ofp-validate --version` reports it — and refuses a document
declaring a later one rather than answer for rules it does not have.

## License

MIT ([LICENSE](LICENSE)) — the same terms as the organization's tool
repositories, so an independent implementation may quote the specification
freely.

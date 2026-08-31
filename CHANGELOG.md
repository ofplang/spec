# Changelog

Normative changes to `SPECIFICATION.md`, one section per specification revision.
A document names the revision it is written against with `spec_version` (2.1).

This file records what changed and what a document written against the previous
revision has to do about it. Why a rule is the way it is belongs in the
specification itself.

## 0.1 (unreleased)

### Added

- **1**, design goal 10: the shape of the Object-flow graph is statically
  determined, and run-time information may only index that shape as a finite
  scalar parameter.
- **1.1 Design principle: static shape.** States the principle every structured
  node kind is designed to, why `branch` is the one kind that does not have the
  property on its own, and restates the principle as a bound on the physical
  resources a workflow needs before it runs.

## 0.0 - 2026-08-19

The first tagged revision: the specification as it stood before the 0.1
amendment, and the text every implementation released so far was written
against (ofplang-validate 0.1.6, ofplang-schedule 0.2.4, ofplang-run 0.3.1,
ofplang-labcode 0.2.0).

Tagged retroactively, so this section records no changes against a predecessor.
The history before it is in the commit log.

<!--
SPDX-FileCopyrightText: 2026 Dominik Wombacher <dominik@wombacher.cc>

SPDX-License-Identifier: CC0-1.0
-->

# ADR-0006: Taskfile as command interface

## Status

Accepted

Date: 2026-10-24

## Context

The repo needs a few commands with fixed arguments. They should be easy
to find, the same on every machine and usable from a pipeline.

## Decision

* `Taskfile.yml` in the repo root holds all commands: `deps`,
  `bootstrap`, `site`, `secret`, `kubeconfig`.
* Tasks stay small. Arguments after `--` go to `ansible-playbook`.
* Task variables must not clash with common environment variables.
  `HOST` is set by many shells, so `task secret` uses `CLUSTER`.

## Consequences

* `task --list` shows every command.
* The pipeline runs the same commands as a person.
* `task` must be installed.

## Alternatives Considered

* Shell wrapper scripts: one file per command, no built-in help.
* Makefile: Make syntax for something that is not a build.
* Plain `ansible-playbook` commands in the README: easy to mistype.

## Related

None.

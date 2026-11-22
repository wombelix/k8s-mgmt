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

We use Task with one Taskfile in the repo root for all commands.

- Each task stays small and calls one tool. Extra arguments pass through
  to that tool.
- Task variable names must not clash with common environment variables,
  such as `HOST`, which many shells set.

## Consequences

- Task lists every command with its description.
- The pipeline runs the same commands as a person.
- Every machine that runs the commands needs Task installed.

## Alternatives Considered

- Shell wrapper scripts need no extra tool. They need one file per
  command and have no built-in help.
- A Makefile works with a tool that most machines already have. Make is
  built for builds, and its syntax adds friction for plain commands.
- Plain `ansible-playbook` commands in the README need no extra tool.
  They are easy to mistype.

## Related

None.

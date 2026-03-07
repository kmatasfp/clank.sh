---
title: "Transcript does not redact sensitive values — secrets, tokens, PII will be sent to the model"
date: 2026-03-07
author: agent
---

# Transcript does not redact sensitive values — secrets, tokens, PII will be sent to the model

## Problem

The transcript currently records command text and captured output verbatim.
There is no redaction of sensitive values before they are stored as
`TranscriptEntry` nodes. When `ask` is implemented and consumes the
transcript as the model's context window, anything that has appeared in a
command or its output will be visible to the model — including passwords,
API keys, tokens, private key material, PII, and any other sensitive data
the user typed or a command printed.

The README is explicit on this:

> Redaction rules apply at all times. Anything governed by a `redaction-rules`
> entry in a command manifest never enters the transcript — not through direct
> output, not through summarization. Secrets do not leak into the AI's view
> of the session.

> `export --secret KEY=value` marks a variable as sensitive. Available to
> agents via the environment, but never echoed in `env`, never written to
> logs, never shown in `ps`, and **never entered into the transcript**.

This must be resolved before `ask` is implemented. Sending an unredacted
transcript to an external model API is the highest-impact security gap in
the current codebase.

## Two distinct redaction mechanisms required

The README describes two separate mechanisms that together cover the
redaction surface. Both must be implemented:

**1. Automatic heuristic redaction**

The transcript must scan every `Command` and `Output` entry for well-known
sensitive patterns and replace matched values with `[REDACTED]` before
storage. This catches values that were never explicitly marked secret —
things like an API key accidentally printed by a command, a password typed
as a command argument, or a JWT token in an HTTP response.

Pattern categories to detect:
- Generic key/token/password/secret argument patterns
  (`--password <value>`, `token=<value>`, `Authorization: Bearer <token>`)
- AWS access key IDs (`AKIA[0-9A-Z]{16}`)
- GitHub tokens (`ghp_*`, `github_pat_*`)
- Generic JWTs (three base64url segments separated by dots, starting `eyJ`)
- PEM private key headers (`-----BEGIN * PRIVATE KEY-----`)
- High-entropy generic API keys (long alphanumeric strings following common
  key= or token= patterns)

Pattern categories that are opt-in (configurable, not always-on due to high
false-positive rate):
- Email addresses
- IPv4 addresses
- Credit card numbers (Luhn candidates)

**2. `export --secret` variable tracking**

When the user marks a shell variable as secret via `export --secret KEY=value`,
the variable name and its value must be tracked by the shell and excluded from
all transcript entries. Any occurrence of the secret value in a command string
or output must be replaced with `[REDACTED]` before the entry is stored.

This mechanism is distinct from heuristic redaction: the user has explicitly
declared these values as secret, so the shell can redact them exactly (no
pattern matching needed — string substitution).

## Implementation scope

**Heuristic redaction** lives in `clank-transcript` as a function
`redact(text: &str) -> String` applied to every entry before storage. It
uses a `RegexSet` of compiled patterns (initialized once via `OnceLock`).
The `regex` crate is already a transitive dependency of the workspace via
`brush-parser`; adding it as a direct dependency of `clank-transcript`
introduces no new code.

The set of active patterns should be configurable at the `Transcript` level,
with a sensible default set. This allows:
- Tests to use a minimal or no-op pattern set (avoiding false positives on
  test data)
- Future user configuration of which pattern categories are enabled

**Variable tracking** requires the shell (in `clank-core`) to maintain a set
of declared-secret values and pass them into the transcript recording call
sites in `run_statement`. The transcript `push` API must accept an optional
list of exact values to scrub in addition to running heuristic patterns.

## What is explicitly out of scope

- Redaction of values that were never routed through shell-managed channels.
  The README is clear: "user-authored commands that deliberately echo sensitive
  values are outside the scope of automatic redaction." If the user types
  `echo mypassword`, the output is `mypassword` — heuristic redaction will
  only catch it if `mypassword` matches a known pattern.
- Retroactive redaction of entries already in the transcript (redaction
  applies at write time, not read time).
- Redaction in `context show` output — entries are stored already-redacted, so
  display methods output what was stored.
- The `--secret` flag on `prompt-user` responses — that is a separate
  mechanism handled at the `prompt-user` builtin level.

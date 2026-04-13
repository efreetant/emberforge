# Security Policy

## Supported Versions

Security fixes are only guaranteed for the latest published version of `emberforge`.

Older copies, forks, or vendored snapshots may document unsafe or stale runtime behavior and should be upgraded.

## What Counts As A Security Issue

Please report issues such as:

- path handling that can escape `PROJECT_DIR`
- prompts or references that encourage unsafe command execution
- verification bypasses that can incorrectly mark work as complete
- secret leakage into logs, reports, or generated artifacts
- prompt-injection paths that can cause the skill to ignore its runtime constraints

## Reporting A Vulnerability

Please do not open a public issue for an unpatched security problem.

Use one of these channels:

1. GitHub private vulnerability reporting, if it is enabled for the repository.
2. Otherwise, contact the maintainer directly through the repository owner's public contact channel and include `emberforge security` in the subject.

Please include:

- a clear description of the issue
- affected files
- impact
- reproduction steps
- any suggested mitigation

## Response Goals

- acknowledge receipt within 7 days
- confirm severity and scope after reproduction
- prepare a fix before public disclosure when practical

## Disclosure

After a fix is available, coordinated public disclosure is welcome. Reports that help tighten runtime boundaries, verification integrity, or secret handling are especially valuable for this skill.

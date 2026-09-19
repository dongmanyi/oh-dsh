# Agent Note: Resume interrupted Windows release downloads

Status: implemented

English | [中文](2026-09-19-windows-release-download-resume.zh.md)

## Problem

The Windows installer downloaded every release asset into a new temporary
directory. A dropped connection discarded the partial transfer, so running
the installer again downloaded the full asset. Large bundled runtimes made
this especially costly on unreliable connections.

## Decision

`install.ps1` stores an incomplete asset under the installer data root's
`downloads` directory, keyed by the release asset name and published SHA-256.
A later run sends an HTTP Range request from the existing byte count. A valid
`206` response must start at that offset and declare the expected total size.
If a mirror responds with `200`, the installer overwrites the partial file
with the full response. Incomplete transfers remain available for retry.

The installer still verifies the complete asset against the release digest
before touching the existing installation. A digest mismatch deletes the
cached file; a successful installation also removes it. Downloads never
receive the GitHub API token, including when a custom mirror is configured.

## Alternatives considered

**Use PowerShell's `Invoke-WebRequest -Resume`.** PowerShell 5.1 is a supported
installer host and does not provide that switch. A streamed .NET request
supports the same Range behavior on both PowerShell 5.1 and newer releases.

**Keep the partial file in the random temporary work directory.** The
installer removes that directory on exit, so it cannot help a later run.
The existing installer data root gives the cache a stable location without
inventing another state root.

## Consequences

Interrupted downloads can be resumed by rerunning the same installer command.
Servers that ignore Range cause a full redownload, but never a corrupted
append. Partial files consume disk until a successful retry or manual
cleanup. This is one Windows installer slice of issue #213; it does not add
delta updates to other surfaces or platforms.

## Testing

The Windows installer tests simulate a dropped response followed by a `206`
resume and a mirror that ignores Range. Both paths install only after the
published digest verifies, and the retry test confirms the cache is removed
after success.

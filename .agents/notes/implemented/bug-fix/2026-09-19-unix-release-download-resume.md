# Agent Note: Resume interrupted macOS and Linux release downloads

Status: implemented

English | [中文](2026-09-19-unix-release-download-resume.zh.md)

## Problem

`install.sh` downloaded every release asset into a new temporary directory.
A dropped connection discarded the partial transfer, so another invocation
started the full download again. The bundled runtime makes release assets
large enough for this to matter on unreliable networks.

## Decision

The installer stores incomplete release assets under the existing installer
record root's `downloads` directory, using the asset name and published
SHA-256 as the cache key. On a later invocation, curl continues from the
cached byte count with HTTP Range. If curl reports that the server cannot
resume (exit 33), the installer deletes the partial file and retries from
the start. A complete verified cache entry can be reused without a request.
If the record root cannot hold downloads, this invocation uses its temporary
staging directory instead; record-writing errors are still reported at commit.

The published SHA-256 is checked before any installation change. A mismatch
deletes the cached file, while a failed transfer retains it. Successful
installation removes the cache. Release downloads never receive the GitHub
API token, including when a mirror base URL is configured.

## Alternatives considered

**Keep temporary downloads between runs.** The existing temporary directory
has a per-invocation random name and is removed on exit. Reusing it would
blur staging and durable state. The installer record root already provides a
stable state location.

**Assume every server supports Range.** Custom mirrors may ignore it. curl's
resume error is handled by restarting a full download; the checksum still
guards the resulting bytes.

## Consequences

macOS and Linux installers can resume an interrupted transfer by rerunning
the same command. Partial files consume disk until a successful retry or
manual cleanup. This is one installer slice of issue #213; it does not add
blockmap delta updates or change the runtime update manager.

## Testing

The install script tests simulate a dropped response followed by a Range
continuation and a mirror that ignores Range. Both cases preserve the
previous installation until the digest verifies; the resume test confirms
the cache is removed after installation.

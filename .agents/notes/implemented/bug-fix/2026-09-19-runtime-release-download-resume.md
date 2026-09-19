# Agent Note: Resume verified runtime release downloads

Status: implemented

English | [中文](2026-09-19-runtime-release-download-resume.zh.md)

## Problem

The Desktop runtime updater rewrote its download archive from the beginning
on every attempt. A connection failure left partial bytes on disk, but the
next click on Update Runtime could not use them. Runtime bundles are large,
so unreliable connections incurred repeated full downloads.

## Decision

Fetch the published SHA-256 sidecar before downloading. When it exists, use
the digest in the partial archive's cache key and request the remaining bytes
with HTTP Range on retry. Accept an appended response only when HTTP 206 and
Content-Range identify the exact cached offset and expected total. A server
that ignores Range returns HTTP 200; replace the partial with its full body.
Discard malformed range responses and checksum mismatches. Only verified
bundles proceed to extraction, smoke checking, and pointer activation.

Bundles without a valid SHA-256 sidecar continue using a full download. This
preserves support for older Releases without trusting partial bytes that
cannot be bound to a published digest. Successful activation removes the
download cache; interrupted transfers retain their partial bytes.

## Alternatives considered

**Resume every existing archive by filename.** A filename alone does not
identify the published bytes. Reusing it across a changed Release could mix
two bundles. The digest-keyed partial file avoids that ambiguity.

**Require Range support.** Mirrors may ignore Range. Replacing the partial
with a full HTTP 200 response keeps the update available.

## Consequences

Rerunning Update Runtime after a network interruption can save transferred
bytes for published bundles with a SHA-256 sidecar. Partial files occupy disk
until a successful update or manual cleanup. The existing staged activation
and rollback contract remains unchanged. This is one slice of issue #213;
it does not add blockmap delta updates or change Desktop application updates.

## Testing

Runtime update tests simulate a dropped response followed by a valid 206
continuation and a server that ignores Range. They verify the requested
offset, successful activation, and removal of the partial cache.

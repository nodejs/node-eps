# Node.js Enhancement Proposals

## Overview

This repository contains the Node Enhancement Proposals (EPs) collection. These
are documents describing an enhancement proposal for inclusion in Node.

EPs are used when the proposed feature is a substantial new API, is too broad
or would modify APIs heavily. Minor changes do not require writing an EP.  What
is and isn't minor is subjective, so as a general rule, users should discuss
the proposal briefly by other means (issue tracker, mailing list or IRC) and
write a EP when requested by the Node core team.

## Rational

The idea behind the EP process is to keep track of what ideas will be worked on
and which ones were discarded, and why. This should help everyone (those
closely involved with the project and newcomers) have a clear picture of where
Node stands and where it wants to be in the future.

## Format

EP documents don't follow a given format (other than being written in
MarkDown). It is, however, required that all EPs include the following
information at the top of the file:

* Title
* Name of author
* Status (more on statuses later, new documents must be submitted with the
'draft' status
* Date

The document file name must conform to the following format:
`"XXX-title-ish.md"` and it must be added to the document index in
`000-index.md`.

## Content

EP documents should be as detailed as possible. Any type of media which helps
clarify what it tries to describe is more than welcome, be that an ASCII
diagram, pseudocode or actual C code.

## Licensing

All EP documents must be MIT licensed.

## Progress of a EP

All EPs will be committed to the repository regardless of their acceptance.
The initial status shall be **"DRAFT"**.

If the document is uncontroversial and agreement is reached quickly it might be
committed directly with the **"ACCEPTED"** status. Likewise, if the proposal is
rejected the status shall be **"REJECTED"**. When a document is rejected a
member of the core team should append a section describing the reasons for
rejection.

A document shall also be committed in **"DRAFT"** status. This means consensus
has not been reached yet.

The author of a EP is expected to actually pursue and implement the proposal.

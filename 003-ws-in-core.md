| Title  | ws in core                  |
|--------|-----------------------------|
| Author | @eljefedelrodeodeljefe      |
| Status | DRAFT                       |
| Date   | 2015-01-25 02:34:00         |

## Description

This EP proposes having the current [websockets/ws](https://github.com/websockets/ws) userland implementation in core.
Efforts have already been made on nodejs/node#4842. It is understood that
adaptions to style and simplicity need to be done in order to conform with
`node` core.

## Motivation

> `websockets` are a foot-in-the-door feature for companies, which have **not**
decided to make a commitment towards `node` to use `node` anyways. For a wider
adoption of the language this is a critical point.

The above statement is the observation of the author.

However in order to gain momentum as a language, the experience should be as
performant, secure and well documented as possible. Userland development is to
be believed not holding enough capacity to meet those requirements. Especially
issues like `Buffer(number)`, however dangerous they are, can only be
effectively secured via backporting code through the language's LTS.

## Note

This document is in progress.

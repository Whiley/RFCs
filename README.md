# Whiley RFCs

Substantial change proposals for the Whiley language must be first
written up as an RFC and before they can be accepted.  The Request for
Comment (RFC) process is intended to provide a peer review and voting
process to ensure change proposals are vetted and accepted by the
community.

## When is an RFC needed?

The RFC process should be followed for all "substantial" changes to
the Whiley language.  What exactly constitutes a "substantial" change
to the language is somewhat subtle (and subjective).  For example, the
following changes _would not be_ considered as substantial:

* **Fixes for known bugs in the language (as identified by an
  issue)**.  A bug typically represents a situation where the
  implementation of the language differs from the language
  specification.  In the normal course of events, one would first
  raise an issue regarding the bug in question.
* **Minor fixes for or improvements to the documentation of a source
  file**.  Such changes can likely be accepted as pull requests
  without issues being raised.
* **Refactoring of an existing code base (e.g. the Whiley Compiler)
  without otherwise changing the semantics or syntax of the
  language**.  Whilst such refactorings can be substantial in nature,
  they should not affect existing code.  However, of course, in
  practice such changes must be made in small incremental steps.

In contrast, the following changes _would be_ considered as
substantial:

* **Changes to the syntax of the language**.  This includes the addition
  of new syntax, the removal of existing syntax and/or the replacement
  of existing syntax.  This is important because existing code may no
  longer compiler after the change.
* **Changes to the manner in which Whiley source files are accepted (or
  not) by the compiler**.  For example, the introduction of a new type
  checking phase, or the modification of an existing phase
  (e.g. definite assignment).  This is important because existing code
  may no longer compiler after the change.
* **Changes to the manner in which compiler Whiley code interfaces with
  external code (i.e. the Foreign Function Interface)**.  This is
  important as such a change will affect existing code in a
  potentially non-trivial fashion.

If in doubt regarding whether a change in substantial or not, one can
first raise this as an issue on the repository of the respective
component.  Maintainers of that repository will then provide feedback
as to whether this requires an RFC or not.

## The Process

To get a substantial change accepted into the Whiley language, one
must first get the corresponding RFC merged into this repository as a
markdown file.  At this point the RFC is "active" and may be
implemented and, eventually, included in Whiley.

The process is as follows:

* **Fork** this [RFC repository](http://github.com/Whiley/RFCs)
* **Copy** `0000-template.md` to `text/0000-my-feature.md`.  Here, 
  `my-feature` should be a descriptive name for the feature.  Don't
  assign an RFC number yet (since this may need to changed later
  anyway)
* **Complete** the RFC by providing as many details of the proposal as
  possible.  The RFC must at a minimum: 1) motivate the need for the
  change; 2) clearly explain the technical details of the change
  (i.e. what parts of the language will be changed and in what way);
  3) highlight any potential impacts of the change (esp. if the change
  is not-backwards compatible); 4) consider alternative solutions to
  the problem.
* **Submit** a Pull Request (PR) against this repository.  This will
  instigate the _review period_ during which the RFC will receive
  feedback and suggestions from the community.
* **Update** the RFC during the review period to address concerns or
  issues raised during review.  Such changes should be made as new
  commits to the Pull Request, along with comments clarifying the
  changes.  RFCs are rarely accepted without changes.
* **Acceptance**.  At some point, once the discussion has slowed, the
  motion will be made for a decision to be made regarding the RFC
  (often referred to as the _Final Comment Period (FCP)_).  After a
  short period (e.g. one week), the RFC will then either be _accepted_
  (and merged) or _rejected_ (and closed).  In some cases, the RFC
  maybe be _postponed_ and kept open.

## The Life-Cycle

Once an RFC has been accepted it must then be _implemented_.  Observe
that, getting an RFC accepted does not automatically mean it will
become part of the Whiley language.  The onus is on the authors and
supporters of the RFC to develop a suitable implementation.  This will
then need to be merged into the relevant parts of the Whiley
ecosystem.  This might include, for example, the Whiley Compiler (WyC)
and the Whiley Language Specification, etc.

## Acknowledgements

Inspiration for this RFC process is taken from the processes used by
[Pony](https://github.com/ponylang/rfcs) and [Rust](https://github.com/rust-lang/rfcs#reviewing-rfcs).

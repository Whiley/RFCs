# Whiley RFCs

Substantial change proposals for the Whiley language must be first
written up as an RFC and before they can be accepted.  The Request for
Comment (RFC) process is intended to provide a peer review and voting
process to ensure change proposals are vetted and accepted by the
community.

## When is an RFC needed?

## The Process

To get a substantial change accepted into the Whiley language, one
must first get the corresponding RFC merged into this repository as a
markdown file.  At this point the RFC is "active" and may be
implemented and, eventually, included in Whiley.

_Creating an RFC_

* **Fork** this RFC repository
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
[https://github.com/ponylang/rfcs](Pony) and [https://github.com/rust-lang/rfcs#reviewing-rfcs](Rust).

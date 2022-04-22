|              |                                  |
| :----------- | :------------------------------- |
| Feature Name | [Name]                           |
| Start Date   | [Today]                          |
| Category     | [Category]                       |
| RFC PR       | [fill this in after opening PR]  |

# State: ACCEPTED
[state]: #state

Select one from:
- `DRAFT`: Normal discussion and changes happen in this state. Achieving
  consensus on what to be done and how. Silence is not agreement but unanimity
  is not needed either.
- `CANDIDATE`: Meant to prove that an implementation is feasible. Changes to the
  implementation can happen in this period, normally from feedback from the
  implementation.
- `ACCEPTED`: Consensus exists that implementation has been a success.
- `REJECTED`: No consensus either for the draft text or the implementation.
- `OBSOLETE`: No longer relevant (overridden by other RFCs, object of
  enhancement no longer in use).

Ideal progression:
`DRAFT` (via a PR, until that PR is merged) -> `CANDIDATE` (iterate implementation) -> `ACCEPTED` (implementation is a sucess).

# Summary
[summary]: #summary

Brief (one-paragraph) explanation of the feature.

# Motivation
[motivation]: #motivation

- Why are we doing this?
- What use cases does it support?
- What is the expected outcome?

Describe the problem you are trying to solve, and its constraints, without coupling them too closely to the solution you have in mind. If this RFC is not accepted, the motivation can be used to develop alternative solutions.

## Examples / User Stories
[examples]: #examples

Examples of how the feature will be used. Interactions should show the action and the response. When appropriate, provide user stories in the form of "As a [role], I want [feature], so [that]."

# Detailed design
[design]: #detailed-design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with the product to understand, and for somebody familiar with the internals to implement.

This section should cover architecture aspects and the rationale behind disruptive technical decisions (when applicable), as well as corner-cases and warnings.

# Drawbacks
[drawbacks]: #drawbacks

Why should we **not** do this?

  * obscure corner cases
  * will it impact performance?
  * what other parts of the product will be affected?
  * will the solution be hard to maintain in the future?

# Alternatives
[alternatives]: #alternatives

- What other designs/options have been considered?
- What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

- What are the unknowns?
- What can happen if Murphy's law holds true?

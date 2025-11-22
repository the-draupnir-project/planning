# Contribution risk and response

This document is about assessing and responding to risk. This document does not
change the nature of review. Reviews are supposed to be optimistic and
encouraging, using risk as an excuse to block the author of a contribution will
be considered a failure to follow the process. The obligation to help
contributors achieve their goals and merge the PR lies with the maintainers, not
the contributor. Always be prepared to make requested changes yourself.

## Quick contribution risk check

Does any of these contexts apply to the contribution?

1. [ ] Security

2. [ ] Infrastructure

3. [ ] Uncertainty

4. [ ] Impact of failure and or bugs

If you are going to do any critical thinking and review, please think critically
about how the contribution impacts the system first. These are a guideline but
you must use your own judgement to determine the level of risk.

## Risk classification

- minimal if none apply

- sensitive if one apply

- critical if two or more apply.

## Contexts

### Security

The contribution has subject matter related to security. Examples include:

- Authentication
- Cryptography (including policy hashes)
- Untrusted input
- Access control (both host and guest (ie Matrix))

### Infrastructure

The contribution has subject matter related to infrastructure. Examples include:

- Changes in dependencies
- Changes to CI
- Changes to contributor workflow
- Changes to the repository

### Uncertainty

The contribution carries a lot of uncertainty. Examples include:

- Reviewer lacks subject knowledge and must trust other experts.
- Large Contribution (300-500+).
- Lack of upfront planning or design.
- Limited communication with contributor.

### Impact

Consider specifically how problems in the change would be triaged if they were
buggy.

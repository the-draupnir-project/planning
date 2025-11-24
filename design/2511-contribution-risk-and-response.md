# Contribution risk and response

This document is about assessing and responding to risk. This document does not
change the nature of review. Reviews are supposed to be optimistic and
encouraging. The obligation to help contributors achieve their goals and merge
the PR lies with the maintainers, not the contributor. Always be prepared to
make requested changes yourself to see the contribution through. However
reviewers are never obliged to approve a change if they are not confident in
change's the correctness or safety.

## Quick contribution risk check

Does any of these contexts apply to the contribution?

1. [ ] Security

2. [ ] Infrastructure

3. [ ] Uncertainty

4. [ ] Impact of failure and or bugs

If you are going to do any critical thinking and review, please think critically
about how the contribution impacts the system first. These are a guideline but
you must use your own judgement to determine the level of risk.

Use the pull request labels to assign risk contexts to pull requests.

## Risk classification

- Minimal if none apply

- Sensitive if one apply

- Critical if two or more apply.

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

## Risk Response

### Quick checklist

- Sensitive changes and above should have a test plan.
- Sensitive changes and above should have the pull request checked out by the
  reviewer.
- Sensitive changes and above should have all security sensitive code analysed
  and discussed.

### Checking out the PR

Check out the PR locally, don't just rely on the webview. Go through the changes
files in your editor (use
https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github).

Make sure everything is as expected. For changes involving UI, run the bot
locally and try to evaluate the feature yourself. Following the test plan if
available.

### Reviewing security sensitive changes

Think pessimistically about authentication and control flow through the
application. And think about how we arrive to the portion of code in question,
and where we go next. Think about the bigger picture and then the smaller
details. If anything is unclear, you MUST not continue and ask for clarification
or get someone else involved.

### Test plans

Pull requests with sensitive or critical risk should include a plan that
demonstrates the testing that the contributor has taken. This should be detailed
enough so that the reviewer can reproduce.

## Expectations management

Always try to get upfront communication with contributors to exchange
expectations before they commit to significant work. New contributors are often
ambitious, and so it is important to try get them to scale down their work or
even plan it to maximise their chances of success.

This is especially important because big changes made by unfamiliar contributors
will always carry significant risk. Even when these changes concern
documentation.

When an unsolicited pull request is opened, it's important to try establish
these expectations retroactively. And to identify and communicate risk.

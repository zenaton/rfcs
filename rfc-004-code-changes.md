# Changing code

## Summary

Documenting the way we introduce change into our codebases. The
[main principles](https://continuousdelivery.com/principles/) are:
- build quality in
- work in small batches
- computer perform repetitive tasks, people solve problems
- relentlessly pursue continuous improvement
- every one is responsible

## Problem

As the team grows, we feel the need to reach a consensus on how to best use the
tools we rely on.

## Proposal

### Branches
- The master branch MUST always be maintained in a deployable state.
- Feature branches SHOULD be short lived with small diffs.
- There SHOULD not be any environment branches. Leverage the CD pipeline
  to deploy from master to any given environment.
- Work in progress MUST happen in a dedicated branch.
- Changes introduced to the master branch SHOULD happen through a Pull Request.
- Readme or changelog changes and version bumps MAY be pushed straight to master.

### Pull Requests (PR)
- PR MUST be merged by the author(s) of the Pull Request.
- Parts of a review that can be automated MUST be automated (e.g. code
  formatting, tests)
- The automated checks MUST be passing for a PR to be merged.
- PR SHOULD be reviewed before merging. Author(s) of a PR are responsible for
  chasing down reviewers if needed.
- The reviewer(s) MUST mark the PR as approved through GitHub's UI.
- Reviewers SHOULD not forget to make sure documentation and/or changelogs are
  updated alongside the code.
- PR done in pair programming MAY be dispensed of review.
- PR SHOULD have small diffs
- Larger features SHOULD be broken into smaller chunks and multiple PRs.
- You MAY use [Feature Toggles](https://www.martinfowler.com/articles/feature-toggles.html)
  when merging parts of larger work.
- You SHOULD write [good commit messages](https://chris.beams.io/posts/git-commit://chris.beams.io/posts/git-commit/).
- Authors MUST limit the number of commits that end up in the master branch, by
  either squashing their WIP commits, rebasing their branch on the latest
  version of master, etc.

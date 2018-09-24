# Keep a Changelog

## Summary

Introduce a Changelog on each project.

## Problem

It is currently a bit hard to keep track of changes in a code base and to relate
them to specific version of releases. In turn, this makes it harder when
providing support to customers. For example, to which version should we tell a
user to upgrade to when experiencing a known/fixed bug?
Additionally, how can we leverage the changes in a new version to better
communicate or promote our offering to new and existing customers?

## Proposal

I suggest we
- add a `CHANGELOG.md` on each code repository, and
- adopt the [keep a changelog](https://keepachangelog.com/en/1.0.0/) format.
- make sure the changelog file is updated during our Pull Request review
  process.

Please check out the link above for a clearer explanation of what the format
looks like.

If you want to see it in action, I am currently using it for the Ruby gem.
- [PR with changelog](https://github.com/zenaton/zenaton-ruby/pull/41/files)
- [Releasing a new version](https://github.com/zenaton/zenaton-ruby/commit/b6dd2a4698aeaa4b6732c9c36040e4f382278e16)

The last examples also leverages GitHub tags. There is a one to one mapping
between released versions of the gem and versioned tags on GitHub. This allows
me to add links to code diffs directly into the changelog.

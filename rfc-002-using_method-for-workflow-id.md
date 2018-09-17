# Using Method For Retrieving Workflow Id,

## Summary
I suggest to use a method to retrieve id. Today it's not the case in Ruby library.

## Problem

The Ruby uses a different strategy than other libraries to retrieve id. It use a simple local property, instead of a method.

## Proposal

Use a get method to retrieve id.

(This RFC is here to remember why we do such choice.)

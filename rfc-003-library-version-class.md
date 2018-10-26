# Library Version Information

## Summary

Include a way to know the version of the library currently in use in the
project.

## Problem

I think that one of the most overlooked issues that will arise in the future
and that could be hard to manage will be to keep consistency between the
libraries in the agent part, and the client libraries that users will use in
their projects.

When users will update their agent, they will always have the up to date
version of the agent libraries. But nothing is forcing them to update the
client library they use in their project.

The opposite scenario could also happen. Users might update the client library
they use in their project, but never update the agent on their production
servers.

Because these two libraries have quite a strong coupling (i.e. we call code in
the client libraries from the agent libraries) and that we are still in the
early days of Zenaton, we will probably sometimes make changes in class names,
methods signatures and we will have to handle a wide range of version
differences between agent and client libraries.

In order to be able to handle most of these scenario in a confident way, we
need a way to know what version of the client library is in use in the project.

## Proposal

Zenaton Client libraries MUST implement a type that would be able to give
version information.

In the PHP library, this class could look like this:

```php
<?php

namespace Zenaton;

final class Version
{
    const VERSION = '1.0.0-DEV';
    const VERSION_ID = 10000;
    const MAJOR_VERSION = 1;
    const MINOR_VERSION = 0;
    const PATCH_VERSION = 0;
    const EXTRA_VERSION = 'DEV';
}
```

The Ruby implementation could look like this:

```ruby
module Zenaton
  VERSION = '1.0.0'
  VERSION_ID = 10000;
  MAJOR_VERSION = 1;
  MINOR_VERSION = 0;
  PATCH_VERSION = 0;
end
```

This will allow us in the agent library to make decisions about what to do
depending on the client library available to us.

If we added a method in version 1.0.0 of the client library that we would like to call,
we can just make something like that:

```php
if (\Zenaton\Version::VERSION_ID >= 10000) {
    \Zenaton\Client::getInstance()->someFancyNewMethod();
}
```

There are many use cases this can help us deal with:

- Knowing what types / methods are available.
- Handling deprecated stuff.
- Have some tests in agent libraries that are run/skipped depending on the
version of the client library.
- Maybe some day if we cannot support too much different versions anymore we
will be able throw an error when we release a new major version of the agent
and we expect the user to use a new major version of the library that goes with
it.

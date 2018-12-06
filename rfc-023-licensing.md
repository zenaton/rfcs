# Licensing

## Summary

We need to attach a license to the public SDKs.

## Problem

Zenaton's public SDKs does not have a license attached to them. That's a bigger problem that it seems, because some companies have very strong politics about this and won't allow their engineers to use an unlicensed library.

## Proposal

Attach the MIT license to all public SDKs, for every language.

```
MIT License

Copyright (c) Zenaton, Inc. and its affiliates.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

Why MIT? Because that's the license most companies are expecting to find. That's the license Facebook [used for some of their libraries](https://code.fb.com/web/relicensing-react-jest-flow-and-immutable-js/) (including React) when they were blamed for using their own opaque license.

Igor has already added the MIT license to the Ruby SDK, but he put is own name in it. :thinking:
# Basic Design

Jest-ish API, but with no globals - all explicit module imports.

## Core Values:
* Isomorphic - feels at home in any modern JS environment (browser, Node, Deno?, etc)
* ESM-centric - take advantage of everything Modules can give us (including taking advantage of proposals which look likely to land - TC39 stage 4, maybe some stage 3?)
* Buildless - Do as much without a build as possible

### Future Goals
* Extensible - Allow for plugins to, for example, run the project's build tool before running tests. <-- this'll require a lot of work to get right!

# Examples

In all examples, importing from 'tr' is a placeholder for this as-yet-unnamed test runner.

## Test File

```js
import { describe, test, expect, beforeEach, afterAll } from 'tr';
import myModule from './my-module.js';

describe('some feature', () => {
  beforeEach(() => {
    // some setup
  });
  test('some behavior', async () => {
    expect(true).toBeTruthy();
  });
});
```

## Test Suite/Config

```js
import { suite } from 'tr';
// I'm not sure yet what this looks like, since I don't know what is needed

```

# Main Moving Parts

* CLI - Minimal, basically just knows how to run server and find files
* Server - Responsible for module-tree rewriting, which allows for mocking, fast reloading, etc. Tracks test results and forwards them on to a reporter, like the CLI
  * Server will need some sub-modules:
    * Module Tree Analyzer
    * Mocker
    * Test Tree Analyzer
    * 
* Client Runtime - Responsible for actual execution of test cases and for reporting results back to server

# Mocking

It's pretty much essential these days that mocking needs to be included.

To mock modules, we can use import maps - it's a perfect use case! Saying 'use x instead of y' is literally what they're made for! We'll just have to provide the right hooks to make it work.

In these examples, 'a.js' is the module under test, 'b.js' is a dependency:

```js
import { expect, mock } from 'tr';
import a from './a.js';
import b from './b.js'; // will import the mocked version of b
import actualB from './b.js!actual'; // I don't love this, but how do we get the actual version if we need it in a test case?

mock.module('./b.js', () => mock.fn());

test('something', async () => {
  b.mockResolvesValue({
    foo: 'bar'
  });

  const result = await myModule.doSomething();

  expect(b).toHaveBeenCalledWith('baz');
});
```

```js
// a.spec.js
import { mock } from 'tr';
import { toBeTruthy } from 'tr/expect';

export const mockB = mock.module('./b.js', () => mock.fn());

export default t => {
  t.beforeEach(() => {
    
  });
  t.describe('foo', t => {
    t.test('it does stuff', t => {
      t.expect(true, toBeTruthy);
    });
  });
};
```

We could make mocks run seamlessly like this:

1st test file pass - use an import map to resolve to no-op versions of test, describe; basically, everythingt but mockModule. That way, we know
what modules are getting mocked, and we don't actually import anything else. This had the bonus of helping encourage users to put all code into the proper lifecycle functions, like beforeEach and afterEach.

2nd pass - use an import map to resolve the real versions of test, describe, etc, plus mocked modules

Yaks to shave (🐃) - is it even possible to do inline modules like this, or will we always have to make them external files? I bet we could abuse a combination of exports and `data:` urls in import maps to make it work. Like this:

```js
// in test
mockModule('./b.js', './my-b-mock.js');
import b from './b.js'; // Gets mocked ky

// in my-b-mock.js
import {mockFn} from 'tr';
import b from './b.js'; // Gets the actual 'b' because import maps are magic

export default mockFn();
```

It's also possible that, due to the static nature of ES exports, we could do some auto-mocking: for example, wrap the module in a proxy and any function calls go to mock functions: `autoMock('./b.js');`?

🐃: How do we make it so we could run this in a browser with no build step/server? Do import maps allow inline code, like base64 encoded data URIs?


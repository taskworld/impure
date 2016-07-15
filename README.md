# impure [![Build Status](https://travis-ci.org/taskworld/impure.svg?branch=master)](https://travis-ci.org/taskworld/impure) [![codecov](https://codecov.io/gh/taskworld/impure/branch/master/graph/badge.svg)](https://codecov.io/gh/taskworld/impure)

__A wrapper object for non-deterministic code.__

Pure functions are deterministic.
They take in values and computes the result based solely on these values.
They are predictable.
They have no side-effects.
They are referentially-transparent.
They are easier to reason.
They are easier to test.

Haskell enforces purity at type level,
by clearly separating IO functions apart from pure functions.
IO functions can call pure functions and produce side-effects,
but pure functions cannot produce any side-effect.

In JavaScript,
there is no such distinction.
Any function may freely call any other function.
And many times, when not used carefully, this can produce unintended side-effects.

By wrapping all non-deterministic code inside a wrapper,
all side-effects must be invoked through a runner.
If your function is not wrapped in IO,
it cannot cause any side-effect to happen.

## Example

### An IO function [(...args) &rarr; IO]

Here, we’re wrapping `console.log()` inside an IO wrapper.

```js
// example/console.js
import * as IO from '..'

export const log = (...args) => IO.create(({ console }) => {
  console.log(...args)
})
```

It does not access any global variable.
Instead, the `console` object is injected into the IO.


### A pure function

Here’s a function that simply adds two values, just for example.

You can see that even though it can import `console.js` and call `log()`,
it will not produce any side-effect.
You only receive a wrapper,
and the only way to make side-effect happen is to trigger it explicitly (through the `run` function).

Since this function is not given the `run` function,
it cannot cause any IO operation to perform.
So basically if we ban all impure function calls
(e.g. banning access to `console`, `Math.random`, `Date.now`, etc)
we end up with a truly pure code in this module.

```js
// example/add.js
export const add = (a, b) => a + b
```


### Another IO function

Here’s our main function.
It runs our pure function and invoke our first IO function to log the result.

```js
// example/main.js
import { add } from './add'
import { log } from './console'
import * as IO from '..'

export const main = IO.do(function* () {
  const result = add(30, 12)
  yield log('The result is ' + result)
})
```


### Entry point

To actually run a wrapped IO object, you need to create the `run` function.
For example, here’s how one might create an application’s entry point.

```js
// example/index.js
import { createRun } from '..'
import { main } from './main'

const context = { console }
const run = createRun({ context })
run(main()).catch(e => setTimeout(() => { throw e }))
```


### Testing code with IO

This actually makes our code testable — as we can create a fake console and use it in our test.

```js
// example/test-utils/createFakeConsole.js
export function createFakeConsole () {
  const logged = [ ]
  return {
    log: (text) => {
      logged.push(text)
    },
    isLogged: (text) => {
      return logged.some(loggedText => text === loggedText)
    }
  }
}
```

```js
// example/main.test.js
import { createFakeConsole } from './test-utils/createFakeConsole'
import { createRun } from '..'
import { main } from './main'

it('should log the result to console', () => {
  const context = { console: createFakeConsole() }
  const run = createRun({ context })
  run(main())
  assert(context.console.isLogged('The result is 42'))
})
```


## the implementation

```js
// createIO.js
import runInContext from './_runInContext'

export function createIO (operation) {
  return {
    [runInContext] (context, run) {
      return operation(context, run)
    }
  }
}
```

```js
// createRun.js
import runInContext from './_runInContext'

export function createRun ({ context }) {
  return run

  function run (io) {
    return io[runInContext](context, run)
  }
}
```

```js
// wrapGenerator.js
import { createIO } from './createIO'
import runInContext from './_runInContext'

export function wrapGenerator (generatorFunction) {
  return function () {
    return createIO((context, run) => {
      return new Promise((resolve, reject) => {
        const iterator = generatorFunction.apply(this, arguments)
        function step (exec) {
          try {
            const result = exec()
            if (result.done) {
              return () => resolve(result.value)
            } else {
              return () => {
                if (typeof result.value[runInContext] !== 'function') {
                  throw new TypeError('Expected an IO object to be yielded.')
                }
                invoke(result.value)
              }
            }
          } catch (e) {
            return () => reject(e)
          }
        }
        function invoke (io) {
          return (Promise.resolve(run(io))
            .then(
              result => step(() => iterator.next(result)),
              error => step(() => iterator.throw(error))
            )
            .then(f => f())
          )
        }
        step(() => iterator.next())()
      })
    })
  }
}
```

```js
// wrapGenerator.test.js
import { createIO } from './createIO'
import { createRun } from './createRun'
import { wrapGenerator } from './wrapGenerator'
const run = createRun({ context: { } })

it('passes arguments from IO function to the generator', () => {
  const fn = wrapGenerator(function* (x, y) {
    assert(x === 42)
    assert(y === 96)
  })
  return ioShouldSucceed(fn(42, 96))
})

it('resolves with result returned from the generator', () => {
  const fn = wrapGenerator(function* () {
    assert((yield fn2()) === 50)
  })
  const fn2 = wrapGenerator(function* () {
    return 50
  })
  return ioShouldSucceed(fn())
})

it('fail when generator throws', () => {
  const fn = wrapGenerator(function* () {
    throw new Error('Something bad happened!')
  })
  return ioShouldFail(fn())
})

it('fail when yielding a non-IO', () => {
  const fn = wrapGenerator(function* () {
    yield 42
  })
  return ioShouldFail(fn())
})

it('fail when a yielded IO throws', () => {
  const fn = wrapGenerator(function* () {
    yield fn2()
  })
  const fn2 = wrapGenerator(function* () {
    throw new Error('Something bad happened!')
  })
  return ioShouldFail(fn())
})

function ioShouldSucceed (io) {
  return Promise.resolve(run(io))
}

function ioShouldFail (io) {
  return Promise.resolve(run(io)).then(
    () => { throw new Error('Expected IO to fail') },
    () => { }
  )
}
```

```js
// _runInContext.js
export default '(╯°□°)╯︵ ┻━┻'
```

```js
// index.js
import { createIO } from './createIO'
import { createRun } from './createRun'
import { wrapGenerator } from './wrapGenerator'

export { createIO, createRun }
export { createIO as create }
export { wrapGenerator as do }
```

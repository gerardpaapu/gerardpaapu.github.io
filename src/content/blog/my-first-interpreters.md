---
title: 'My first interpreters'
description: 'Baby steps into writing interpreters and compilers'
pubDate: 'Feb 18 2023'
---

Interpreters and compilers have a reputation in some circles of being something that only experts can write or understand. That might be true of big projects like LLVM, V8 and GCC, but I've found lots of solutions in my career that are interpreters in disguise and orders of magnitude simpler than those huge products.

So let's take a look at a couple of really simple interpreters and one baby compiler.

## Evaluate

We're going to consider a mathematical expression language, our interpreter is a function called evaluate and it will take an expression and return a number.

A simple expression in this language is a number. Interpreting a simple expression
just means returning that number.

```js
evaluate(4) // => 4
evaluate(-10) // => -10
evaluate(0) // => 0
```

This language also includes addition. An addition expression is represented
as a three element array containing the addition symbol `'+'` and then two more expressions.

```js
evaluate(['+', 4, 3]) // => 7
evaluate(['+', 6, ['+', 4, 3]]) // => 13
```

Take a second to think about what shape this function is going to have. First we'll consider the simple case:

> If the expression is a simple expression (i.e. it is a number) we can return
> that expression as a value

```js
function evaluate(expression) {
  if (typeof expression === 'number') {
    return expression
  }

  // ... snip
}
```

Right now we only have one other type of expression -- Addition -- so we can handle that
whenever the first doesn't match. When the expression wants to add two things together, we
will just add them together in JavaScript using `+`

```js
function evaluate(expression) {
  if (typeof expression === 'number') {
    return expression
  }

  const [_, a, b] = expression
  return a + b
}
```

This will handle some of the expressions we've seen, e.g. `evaluate(['+', 2, 3])` works fine, but `evaluate(['+', 6, ['+', 4, 3]])` will be more problematic and return a string `'6+,4,3'`

We tried to add an array to a number because we tried to treat an expression as though it were a value. Before we try adding them, we need to convert the expressions to values by evaluating them. This makes `evaluate` recursive. Many or even most of the interpreters we write will be recursive, and it's important to have the expression/value distinction in mind.

```js
function evaluate(expression) {
  if (typeof expression === 'number') {
    return expression
  }

  const [_, a, b] = expression
  return evaluate(a) + evaluate(b)
}
```

Now we have an `evaluate` function that can handle all the example expressions we've given, but we can't express many math problems in addition alone, so we're going to add some more operations.

Let's suppose we represent multiplication in a similar way, a 3-element array with a `'*'` at the front and two expressions.

We can evaluate that almost exactly like we deal with addition:

```js
function evaluate(expression) {
  if (typeof expression === 'number') {
    return expression
  }

  const [operator, a, b] = expression
  switch (operator) {
    case '+':
      return evaluate(a) + evaluate(b)
    case '*':
      return evaluate(a) * evaluate(b)
  }
}
```

Here's an exercise I leave to you, implement subtraction and division. I wrote some tests to get you started.

```js
describe('Simple expressions', () => {
  test('evaluate to the expected results', () => {
    expect(evaluate(2)).toBe(2)
    expect(evaluate(['*', 2, 3])).toBe(2 * 3)
    expect(evaluate(['/', 2, ['-', 3, 5]])).toBe(2 / (3 - 5))
  })
})
```

This type of function is known as a tree walking interpreter, because our
expression type can contain other expressions, complex expressions form a tree.

```js
evaluate(['*', ['-', 2, 4], ['/', ['+', 4, ['*', 0.4, -1]], ['*', 3, 2.1]]])
```

(You might have to squint a little for this to look like a tree)

```
'*'
 |- '-'
 |   |- 2
 |   `- 4
 |
 `- '\'
     |- '+'
     |   |- 4
     |   `- '*'
     |       |- 4
     |       `- '*'
     |           |- 0.4
     |           `- -1
     `- '*'
         |- 3
         `- 2.1
```

A lot of interpreters are written this way, at least at first, because they're straightforward to write and often that's by far the most important thing.

There are two performance issues that are hard to overcome in tree-walking interpreters:

### Walking the tree is slow

In the best case scenario, accessing a property on a JavaScript object is dereferencing a pointer to an object on the heap. This means that you're looking up an object in an effectively random memory location over and over during your evaluate function.

Computers like to work their way through memory in a less random way, preferably after using one location in memory you should use the one next to it. That's usually much faster

### Recursion is slow and dangerous

When a function calls another function to continue or even finish its work, a computer will allocate another stack frame, which takes time and memory, and when that function completes it
will pop that stack frame.

Many languages, including JavaScript have a maximum call depth or maximum recursion depth in practice if not in theory.

When your data structure gets very deep this is a very real risk.

(Warning: both of these are common patterns in code and neither of them is very problematic unless the code is performance sensitive or you're expecting deep data)

## Stacks

If we were using a different language, we would be able to write an interpreter that

1. doesn't walk a tree, just consumes input from left-to-right
2. isn't recursive, just loops and keeps temporary values in a local data structure

Instead of thinking in expressions this time, we're going to think of instructions and instead
of calling our own function to complete our evaluation we will use a stack (in JavaScript an Array) as a temporary working space.

1. Numbers are an instruction, they mean "push this number onto the stack"
2. "+" means: "pop the top two items off the stack, add them together and push the result"

Let's call our function `run`, it takes an array of instructions, follows these two
rules and then returns the value at the top of the stack.

(also when a program is complete, there should only be one value in the stack)

```js
run([4]) // => 4
run([1]) // => 1
run([7, 3, '+']) // => 10
run([5, 7, '+', 3, '+']) // => 15
```

an implementation might look like this:

```js
function run(program) {
  const stack = []
  for (const instruction of program) {
    if (typeof instruction === 'number') {
      stack.push(instruction)
      continue
    }

    switch (instruction) {
      case '+':
        const b = stack.pop()
        const a = stack.pop()
        stack.push(a + b)
        break

      default:
      // Invalid instruction
    }
  }

  if (stack.length !== 1) {
    throw new Error('Program incomplete')
  }

  return stack[0]
}
```

Now, I'm not going to always dig into the micro-performance of our functions, but I think
this aspect is a little interesting.

If an array in JavaScript is a constant size, and always contains the same type of thing (especially if that is a primitive type), there's optimisations the JavaScript engine might use

So, if instead of use an array that shrinks and grows as a stack we use an array that starts
off with let's say a bunch of zeros, and an index into that array.

```js
const stack = [0, 0, 0, 0, 0, 0, 0]
let idx = -1
```

We'll start the index at `-1` to mean "there's nothing on the stack, popping doesn't make sense yet"

Pushing a value onto the stack means incrementing `idx` and writing to `stack[idx]`, e.g. this is what pushing `5` onto the stack looks like.

```js
idx++
stack[idx] = 5
```

or we can even write it in one line:

```js
stack[++idx] = 5
```

and "popping" means, accessing the value at `idx` and then moving the idx down the stack one

```js
let popped = stack[idx--]
```

So, now we have to pass a pre-sized stack to our `run` function and it looks like this:

```js
function run(program, stack) {
  let idx = -1
  for (const instruction of program) {
    if (typeof instruction === 'number') {
      // push instruction onto the stack
      stack[++idx] = instruction
      continue
    }

    const rhs = stack[idx--]
    const lhs = stack[idx--]
    switch (instruction) {
      case '+':
        stack[++idx] = lhs + rhs
        break

      default:
      // OOPS
    }
  }

  // the stack should have exactly one item on it
  if (idx != 0) {
    throw new Error()
  }

  return stack[0]
}
```

And now we can run our examples:

```js
const stack = [0, 0, 0, 0, 0, 0]
run([5, 7, '+', 3, '+'], stack) // => 15
```

### Exercise: Implement other ops

Can you implement multiplication, subtraction and division in this stack language?

Meet back here when you're done.

```js
describe('running programs', () => {
  it('emits the answer', () => {
    const stack = [0, 0, 0, 0, 0, 0]
    const result = run([5, 7, '+', 3, '*', 2, '/'], stack)
    expect(result).toBe(((5 + 7) * 3) / 2)
  })
})
```

So... we have one language that is nicer to write and one that is easier to interpret quickly.

Ideally, we would write expressions in one and it would execute as quickly as the other.

Well, we can do that if we have a function that converts from the tree-language to the
stack language.

we're going to have roughly the same structure for any function that is using the tree, so let's copy that:

```js
function expressionToProgram(expression) {
  if (typeof expression === 'number') {
    // TODO: something
    return
  }

  const [operator, a, b] = expression
  switch (operator) {
    case '+':
      return // TODO: probably something recursive?
    case '*':
      return // TODO: expressionToProgram(...)

    // TODO: handle the other operators
  }
}
```

... and we know that we want to return an array of instructions, so we can do that.

As usual, the simple case is a good place to start. An expression that is just a number
maps to a program of one instruction that pushes a number

```js
if (typeof expression === 'number') {
  return [expression]
}
```

When our expressions contain other expressions, our function is probably going to
call itself twice (once for each sub-expression) and we'll combine the results somehow.

In this case we want to return a program that includes all the instructions to calculate the
left-hand-side expression, then all the instructions to calculate the right-hand-side expression, then we finish it off with the instruction to add those two together.

```js
case '+':
  return [
    ...expressionToProgram(a),
    ...expressionToProgram(b),
    '+'
  ]
```

Put all together, it's going to look a bit like this

```js
function expressionToProgram(expression) {
  if (typeof expression === 'number') {
    return [expression]
  }

  const [operator, a, b] = expression
  switch (operator) {
    case '+':
      return [...expressionToProgram(a), ...expressionToProgram(b), '+']
    case '*':
      return [...expressionToProgram(a), ...expressionToProgram(b), '*']

    // TODO: handle the other operators
  }
}
```

Just as a quick aside, creating and spreading all these arrays can be expensive, so we might want to look at an alternative way to build up the program that is a little easier on the garbage-collector.

```js
function expressionToProgram(expression) {
  let program = []
  addToProgram(expression, program)
  return program
}

function addToProgram(expression, program) {
  if (typeof expression === 'number') {
    program.push(expression)
  }

  const [operator, a, b] = expression
  // optionally, we can move this up here if we want to
  // cut down on duplication, but it implies that all of our operators
  // will have exactly one operand, so if that assumption changes, so will
  // this line of code
  addToProgram(a, program)
  addToProgram(b, program)

  switch (operator) {
    case '+':
      program.push('+')
      return

    case '*':
      program.push('*')
      break

    // TODO: handle the other operators
  }
}
```

You guessed it! Homework time.

First, fill in the rest of the operators. You know the deal, it's not very hard or interesting by now.

Then to test this there's two kinds of assertions we might care about.

First, we might have an exact expectation of what program is emitted for a
specific expression

```js
describe('compiling to a stack language', () => {
  it('emits correct programs', () => {
    expect(expressionToProgram(2)).toEqual([2])
    const expected = [2, 1, '+', 3, '*']
    expect(expressionToProgram(['*', ['+', 2, 1], 3])).toEqual(expected)
  })
})
```

Second, it might be more important that we get the same result for an expression whether we calculate it with `evaluate` or by converting it to a stack program and running it.

```js
const examples = [
  5,
  ['*', ['+', 2, 4], ['-', -3, 12]],
  ['-', 2, ['*', 9, 9]],
  ['/', ['*', 9, 7], 7],
]

describe('compiling to a stack language', () => {
  it('emits equivalent programs', () => {
    expect.assertions(examples.length)

    for (const expr of examples) {
      const stack = expressionToProgram(expr)
      const expected = evaluate(expr)
      const actual = run(stack)

      expect(actual).toBe(expected)
    }
  })
})
```

This second property means that we can use `evaluate` directly if an expression is only needed once, or `run` if we needed to do the same calculations in a hot loop and our tests give use the assurance that the result will be the same either way.

## Recap

So we've seen a few patterns here that are really common here.

First we've seen a tree-walking interpreter. These recursively process an Abstract Syntax Tree (AST) directly and produce a result. The AST maps pretty closely to the user's mental model of the language and how they write it. An AST is usually produced by another function called a parser.

Despite their performance characteristics, these are often used where clarity of code is more important than raw speed, or where they are operating on programs that are likely only run a few times.

We also wrote an interpreter that operates on two arrays, the instructions array is read beginning to end in a loop, executing each instruction with the other array acting as a working space for calculations.

Many industrial strength interpreters contain something similar, usually called a "bytecode interpreter" (when each instruction fits cleanly into a byte), or sometimes a stack-machine or just virtual-machine since the pattern of executing a sequence of simple instructions with a small working space is similar to how CPUs work.

We **also** wrote a compiler, a function or program that takes in programs in a specific format and returns or emits a program (with equivalent behaviour) in a language that is easier to
execute efficiently.

Many industrial strength interpreters feature a compiler that goes from an AST to a bytecode or similar, since it's an easy way to get performance gains without introducing much complexity.

Often "compiler" is a scary word because we think first about the biggest most complicated compilers first. C/C++ compilers that have been built by thousands of experts over decades that do intricate optimisations to code to make it faster.

Big solutions exist for big problems, but once you have learned some of these patterns you will see that there is room for small compilers to solve small problems. Even our simple function that really only does the tree-walking ahead of time, speeds up the calculations by about %30.

We don't need an army of geniuses to apply these tools to problems around us. We just need to
learn some patterns and keep our eyes open.

## Parsing

In these examples, I've skipped the parser. A lot of resources about writing interpreters get started in parsing which can be really fun but I think most of the interesting ideas in this space don't require you to write your own parser, and in fact doing it can be really distracting.

So in the mean-time, we're going to use a parser that supports one simple but flexible syntax
and use it for everything.

```js
import { parse } from '@donothing/lisp-reader'

const ast = parse(`
  (* (+ 2 3)
     (/ 3 -1))
`)

// ast = ['*', ['+', 2, 3], ['/', 3, -1]]
console.log(interpret(ast)) // LOG: -5/3
```

### Exercise: load files

Write a command line program:

1. reads a file like the example below
2. parses the contents with '@donothing/lisp-reader'
3. evaluates the ast as an expression
4. writes the results to stdout (e.g. with `console.log(...)`)

So if you save this as `example.lisp`

```scheme
;; should be approx -2
(* (+ 2 3)
   (/ 3 -1))
```

You should be able to run it like this

```bash
$ node calc example.lisp
-1.6666666666666667
```

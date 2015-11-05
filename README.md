# Gherkin 3

[![Join the chat at https://gitter.im/cucumber/gherkin3](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/cucumber/gherkin3?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[![Build Status](https://travis-ci.org/cucumber/gherkin3.png)](https://travis-ci.org/cucumber/gherkin3)

Gherkin 3 is a parser and compiler for the Gherkin language.

It is intended to replace [Gherkin 2](https://github.com/cucumber/gherkin) and be used by
all [Cucumber](https://cukes.info) implementations to parse `.feature` files.

If you want a reference implementation of Cucumber (based on Gherkin3), take a
look at [microcuke](https://github.com/cucumber/microcuke).

Gherkin 3 is currently implemented for the following platforms:

* [.NET](https://github.com/cucumber/gherkin-dotnet)
* [Java](https://github.com/cucumber/gherkin-java)
* [JavaScript](https://github.com/cucumber/gherkin-javascript)
* [Ruby](https://github.com/cucumber/gherkin-ruby)
* [Go](https://github.com/cucumber/gherkin-go)
* [Python](https://github.com/cucumber/gherkin-python)
* [Objective-C](https://github.com/cucumber/gherkin-objective-c)

See [`CONTRIBUTING.md`](CONTRIBUTING.md) if you want to contribute a parser for a new language.
Our wish-list is (in no particular order):

* C
* Perl
* PHP
* Rust
* Elixir

## Example

```java
// Java
Parser<Feature> parser = new Parser<>(new AstBuilder());
Feature feature = parser.parse("Feature: ...");
```

```csharp
// C#
var parser = new Parser();
var feature = parser.Parse("Feature: ...");
```

```ruby
# Ruby
require 'gherkin3/parser'
parser = Gherkin3::Parser.new
feature = parser.parse("Feature: ...")
```

```javascript
// JavaScript
var Gherkin = require('gherkin');
var parser = new Gherkin.Parser();
var feature = parser.parse("Feature: ...");
```

```go
// Go
import (
  "strings"
  "github.com/cucumber/gherkin-go"
)
reader := strings.NewReader(`Feature: ...`)
feature, err := gherkin.ParseFeature(reader)
```
*Download the package via: `go get github.com/cucumber/gherkin-go`*

```python
# Python
from gherkin3.parser import Parser

parser = Parser()
feature = parser.parse("Feature: ...")
```

## Table cell escaping

If you want to use a newline character in a table cell, you can write this
as `\n`. If you need a '|' as part of the cell, you can escape it as `\|`. And
finally, if you need a '\', you can escape that with `\\`.

## Why Gherkin 3?

I wrote up a summary [here](https://groups.google.com/d/msg/cukes/YLKsqbBMBoI/DYhfFx8GBegJ).

## Architecture

The following diagram outlines the architecture:

    ╔════════════╗   ┌───────┐   ╔══════╗   ┌──────┐   ╔═══╗
    ║Feature file║──>│Scanner│──>║Tokens║──>│Parser│──>║AST║
    ╚════════════╝   └───────┘   ╚══════╝   └──────┘   ╚═══╝

The *scanner* reads a gherkin doc (typically read from a `.feature` file) and creates
a *token* for each line. The tokens are passed to the *parser*, which outputs an *AST*
(Abstract Syntax Tree).

If the scanner sees a `# language` header, it will reconfigure itself dynamically
to look for Gherkin keywords for the associated language. The keywords are defined in
`gherkin-languages.json`.

The scanner is hand-written, but the parser is generated by the [Berp](https://github.com/gasparnagy/berp)
parser generator as part of the build process.

Berp takes a grammar file (`gherkin.berp`) and a template file (`gherkin-X.razor`) as input
and outputs a parser in language *X*:

    ╔════════════╗   ┌────────┐   ╔═══════════════╗
    ║gherkin.berp║──>│berp.exe│<──║gherkin-X.razor║
    ╚════════════╝   └────────┘   ╚═══════════════╝
                          │
                          V
                     ╔════════╗
                     ║Parser.x║
                     ╚════════╝

Also see the [wiki](https://github.com/cucumber/gherkin3/wiki) for some early
design docs (which might be a little outdated, but mostly OK).

### AST

The AST produced by the parser can be described with the following class diagram:

![](https://github.com/cucumber/gherkin3/blob/master/docs/ast.png)

Every class represents a node in the AST. Every node has a `Location` that describes
the line number and column number in the input file. These numbers are 1-indexed.

All fields on nodes are strings (except for `Location.line` and `Location.column`).

The implementation is simple objects without behaviour, only data. It's up to
the implementation to decide whether to use classes or just basic collections,
but the AST *must* have a JSON representation (this is used for testing).

Each node in the JSON representation also has a `type` property with the name
of the node type.

You can see some examples in the
[testdata/good](https://github.com/cucumber/gherkin3/tree/master/testdata/good)
directory.

### Compiler

(Work in progress)

The compiler compiles the AST produced by the parser
into a simpler form - *Pickles*.

    ╔═══╗   ┌────────┐   ╔═══════╗
    ║AST║──>│Compiler│──>║Pickles║
    ╚═══╝   └────────┘   ╚═══════╝

The rationale is to decouple Gherkin from Cucumber so that Cucumber is open to
support alternative formats to Gherkin (for example Markdown).

The simpler *Pickles* data structure also simplifies the internals of Cucumber.

Each `Scenario` will be compiled into a `Pickle`. A `Pickle` has a list of
`PickleStep`, derived from steps in a `Scenario`.

Each `Examples` row under `Scenario Outline` will also be compiled into a `Pickle`.

Any `Background` steps will also be compiled into each `Pickle`.

Tags will be compiled into the `Pickle` as well (inheriting tags from parent elements
in the Gherkin AST).

Example:

```gherkin
@foo
Feature:
  Background:
    Given a

  Scenario: b
    Given c

  @bar
  Scenario Outline: c
    Given <x>
    When y

    @zap
    Examples:
      | x |
      | d |
      | e |
```

This will be compiled into several `Pickle` objects (here represented as YAML
for simplicity):

```
- tags:
  - @foo
  steps:
  - text: Given a
  - text: Given c
- tags:
  - @foo
  - @bar
  - @zap
  steps:
  - text: Given a
  - text: Given d
  - text: When y
- tags:
  - @foo
  - @bar
  - @zap
  steps:
  - text: Given a
  - text: Given e
  - text: When y
```

Each `Pickle` will also keep a reference back to the original AST nodes for
rendering and error reporting (stack traces).

Cucumber will further transform this list of `Pickle` to a list of `TestCase`.
This structure will link runtime information such as Hooks and Step Definitions.

In the short term, the pickle struct definitions will live alongside the Gherkin3
codebase until it settles:

                                   ┌─────────┐
            ┌─────────────┬────────│Cucumber │──────────────┐
            │             │        └─────────┘              │
            │             │             │                   │
            │             │             │                   │
            │             │             │                   │
            │             │             │                   │
    ┌───────┼─────────────┼─────────────┼───────┐           │
    │       │             │             │       │           │
    │       V             V             V       │           V
    │  ┌─────────┐   ┌─────────┐   ┌─────────┐  │      ┌─────────┐
    │  │Gherkin3 │   │Gherkin3 │   │ Pickles │  │      │Markdown │
    │  │ Parser  │   │Compiler │──>│         │<─┼──────│Parser / │
    │  └─────────┘   └─────────┘   └─────────┘  │      │Compiler │
    └───────────────────────────────────────────┘      └─────────┘
                     Gherkin3 lib

The long term plan is to implement more compilers that can produce `Pickles`, for
example from Markdown.

When the Pickles/Cucumber seam stabilises we might move Pickles to a separate project.
The various compilers producing pickles might move to a separate project too.

## Building Gherkin 3

See [`CONTRIBUTING.md`](CONTRIBUTING.md)

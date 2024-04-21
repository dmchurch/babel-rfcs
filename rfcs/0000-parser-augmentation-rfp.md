- Repo: babel/babel
- Start Date: 2024-04-13
- RFC PR: <!-- leave this empty, to be filled in later -->
- Related Issues: <!-- if relevant -->
- Authors: Danielle Church
- Champion: <!-- the name of a core team member that will present this RFC to the team. You can leave this empty if you haven't found it yet  -->
- Implementors: Danielle Church

# Summary

Request for position on (and implementation of) draft TC39 proposal for standardized parser transformations.

# Changes

- 2024/04/17: Added sections on [Runtime PA](#runtime-pa--no-exhaustiveness-requirement), [Validator
  transformations](#validator-transformations), and [Future-proofing ECMAScript](#future-proofing-ecmascript).
  Slightly altered the semantics of the `syntax` directive and updated the example in [Composition of
  Transformation Paths](#composition-of-transformation-paths).
- 2024/04/18: Rewrote the example for [Validator transformations](#validator-transformations) to
  clarify the use case. Edited the pipeline example in [TD Semantics](#td-semantics) to clarify that this
  proposal does NOT change existing ECMAScript parse or import order.
  Added an example static analysis method to the [FAQ](#frequently-asked-questions) section.
- 2024/04/20: Revised the alternative syntax grammar in [Specifying Tranformations / Inline](#inline), adding a `from`
  keyword before the opening bracket of the array-like syntax, to avoid collision with member access syntax when
  `syntax` is used as a variable name. It is now possible to determine if a given instance of `syntax` is a directive
  or a primary expression by looking at the next scanner token.

# Motivation

ECMAScript does not currently have any way to describe the syntax transformations required to turn source code
(as it appears in source repositories) into ECMAScript code that can be used in modern engines. In the absence
of any official mechanism, Babel and other transpilers use documentation, external configuration files, and web
fora like StackOverflow to describe and define the transformations that must be applied to any given project
before it can be used by an ECMAScript host. These artifacts and ephemera have no consistency between transpilers and
it places a heavy burden on developers that want or need to change their build processes. Furthermore, there is
no standard way to validate that these changes have been applied to the source files properly, which opens the door
to hard-to-debug errors that occur only because of incorrect transpilation; in the worst case scenario, a supply-chain
attack like the one that recently affected liblzma could go unnoticed by hiding only in the transpiled/minified
distribution of an ECMAScript-targeting package.

I'm developing a proposal for inclusion in the ECMA262 spec that addresses this shortcoming. I have posted an
incomplete draft of it on the TC39 discourse, which can be seen here:

https://es.discourse.group/t/proposal-parser-augmentation-mechanism/2008

That draft is largely focused on the impact to the TC39 proposal process, so I'll dig a little deeper into
the implementation vision that I have for it, here. Note of course that unless and until TC39 decides to weigh in
on the proposal or a concrete implementation is realized, the syntax and semantics discussed herein should be
considered extremely tentative.

# Detailed design

Parser Augmentation (hereafter PA) is, at its core, a mechanism to _describe_ transformations that can be applied to
source material to render it into a valid ECMAScript syntax tree. While the PA framework can be used to define
novel transformations in the style of a classic C preprocessor, its primary purpose is to provide a method for
(a) an ECMAScript engine or transpiler to report its builtin syntactic capabilities to ECMAScript code, and for
(b) ECMAScript code to inform the engine or transpiler what transformations must be applied to any given source
before it can be successfully parsed.

This represents an unprecedented level of user control over the ECMAScript import machinery, exceeded only by the
proposed [VirtualModuleSource] mechanism described in the Compartments proposal at Stage 1 of the TC39 process.
Unlike VirtualModuleSource, PA can only generate code that could be written in pure ECMAScript, and it cannot override
the default import behavior to, for example, provide a different object to every consumer of a given module. However,
it _can_ represent modification or replacement of an entire source file without restriction, as long as the result is a
valid ECMAScript syntax tree.

Because of this, some parts of the PA design will be defined as _optional_. In particular, a host is permitted to
reject attempts to define _new_ transformations for any reason, including unconditionally. Also, a host is permitted
to "play dumb" - to generate a SyntaxError when asked to perform a transformation as though it were unknown - as long as
the given transformation is not mandated to be supported by ECMA262. For example, the [JSON Modules][json-modules]
proposal could be implemented as a PA transformer with the type "json"; once the proposal is accepted and included
in the spec, it would not be permissible to refuse transformation of an import with "json" type.

Despite these limitations, it is exceedingly likely that browser vendors will be slow to adopt a polyfillable PA
mechanism given the attack risk it repesents, not to mention the common impression that implementing PA _requires_
calling JS code at parse-time. However, neither of these drawbacks affect transpilers like Babel. Transpilation is
nearly always performed with controlled inputs, and PA directives in the transpilation context only serve to enable,
disable, or configure the transformations that a transpiler is already making.

## Transformation Descriptions

To lead with an example, the JSON Modules specification could be described the following way, using PA semantics:

```js
async function json(parser) {
  // The parser should parse the rest of the input as text and emit it
  // as a string constant token, as though it were surrounded in double quotes
  // and all appropriate escape characters added.
  parser.setTokenizerMode(Parser.TOKENIZER_STRING_CONSTANT);
  // Without any arguments, parseToken() will consume as much as possible.
  // The return type is an AST node bound to file:line from the source.
  const stringConstantToken = await parser.parseToken();
  // The syntheticAST function runs the default ECMAScript parser on a template,
  // marking all generated AST nodes as synthetic (not in source text)
  parser.emit(Parser.syntheticAST`export default JSON.parse(${stringConstantToken})`);
}

// Engines are not required to allow registration of implementations, so the following statement could fail:
Parser.registerImplementation("json", json, { polyfill: true, validate: true });
const { default: { meaning } } = await import("life.json", { type: "json" });
```

The `json` function shown here is a _Transformation Description_ (TD). It defines the steps that a parser _could_ take
to perform the given transformation. The API will be designed in such a way that it can be as performant as possible
without compromising its semi-declarative nature, but no engine or transpiler, even one that supports arbitrary PA,
is required to execute the TD function in any observable way. An implementation can use its own native functionality
for the transformation (such as Babel's existing JSON module support); it can perform static analysis of the
TD function to construct an internal parser representation (engines are allowed to reject any non-analyzable
transformations); or, in fact, it can simply run the TD function at parse-time, passing it a handle to a
PA-compliant parser object.

As long as the resulting AST matches what the TD function _would_ have produced, the
implementation is compliant. If executing the TD function would produce observable side-effects, the PA implementation
is permitted either to reproduce those side-effects faithfully (for example, by executing the TD function as part
of parsing), or to inhibit the side-effects completely (for example, by using a different implementation, or by running
it in a sandbox), and it may choose differently for each transformation. It is a programming error for the TD function
to produce different AST outputs based on the observability of side-effects or lack thereof, and PA implementations
are permitted to reject any transformations that show such discrepancies.

A TD can also be registered to _validate_ a transformation, as it is here. If a validating TD is registered for
a transformation and that transformation executes with inhibited side-effects, then afterwards, the TD function must
be called _with_ visible side-effects. This second transformation must produce an AST that matches the first, and the
PA implementation must reject the Transformation if it does not.

### TD Syntax

A transformation description must be written in _statically analyzable ECMAScript_. The PA specification does not
and will not define what is meant by "statically analyzable", other than to make the tautological assertion that
"a statically analyzable TD function is one that can be statically analyzed by a PA-compliant engine".

### TD Semantics

A single _Transformation_ actually represents two conversions: a _non-syntactic_ conversion at the level of bytes
or text, and a _syntactic_ conversion of an AST from one form to another. Most transformations are either purely
syntactic, or purely non-syntactic, but there are exceptions. Every transformation, no matter what category, is
defined by a Transformation Description, above. A set of several Transformations in sequence form a _Transformation
Path_, and the PA mechanism includes a number of ways to associate a source text with a Transformation Path; see below.
Transformations, and likewise Transformation Descriptions, can be categorized by what type of input they expect
and what type of output they produce. All are represented by the same TD interface, but the various categories
have slightly different requirements and semantics.

A **non-syntactic** transformation is one that does not utilize the ECMAScript parsing logic. The `json` function
above describes a non-syntactic transformation; it treats the entire source text as a string, without regard for
syntax. Non-syntactic transformations may directly produce an AST node as above, but they can also produce a
a string or an ArrayBuffer (representing textual vs binary output), both of which can be obtained from appropriate
methods called on the `parser` object.

One (and only one) transformation in a path may be a **baseline** transformation. A baseline transformation is one
that is responsible for defining the initial parse from raw text into AST nodes, and it may be non-syntactic or
syntactic. A **non-syntactic baseline**, like the `json` transformation modeled above, utilizes the tokenizer control
methods and methods like `parseToken` to pull sequential tokens from the input stream, producing synthetic nodes as
necessary to emit an ECMAScript AST. If a transformation path does not specify any baseline transformation, the default
ECMAScript baseline is conceptually positioned between the last non-syntactic and the first syntactic transformation.

The **syntactic** transformations are those that use the ECMAScript parser and accept parsed AST nodes as input.
The `parser` object passed to the TD function has methods to change certain characteristics of the parser, for
example by defining or redefining operator syntax or precedence. Typically, a syntactic TD function will start
by adjusting any parser characteristics needed, and then it will repeatedly call parser methods to scan for AST
nodes of interest, emitting output nodes according to its internal logic.

Note, however, that these categories are simplifications, and are not absolute. All transformations represent
both a syntactic and a non-syntactic conversion, and a "syntactic" transformation is simply one whose non-syntactic
portion is identity; the null transformation. Transformations may shift categories during parsing; implementations
are permitted to reject any that do with a SyntaxError. (Or, as always, for any other reason.)

A **syntactic baseline** transformation is one that calls the `newBaseline()` or `setBaseline()` methods on the
parser instance before anything else. The baseline transformation is responsible for setting all operators,
keywords, etc; if no baseline transformation is specified, the default ECMAScript parser settings are used. The
`newBaseline()` method clears the parser state entirely and could be used e.g. to define a transformation from a
strictly limited subset of ECMAScript syntax, like JSON or JSON5, or even from an entirely different language.
The `setBaseline()` method resets the parser state to a known default, taking options that can specify which
release year of the ECMA262 specification provides the appropriate operators and keywords.

If a syntactic transformation does not define a baseline, then it is an **augmenting** transformation. An augmenting
transformation can be used to define or redefine syntax features. For example, a TD for the [Pipeline][pipeline]
operator might read as follows:

```js
function pipeline(parser) {
    // Defined operators and keywords can have handlers associated with them, as the third argument.
    // The handlers are called upon emit.
    parser.defineOperator("|>", {associativity: "left", precedence: "=", context: "block"}, pipeOperator);
    parser.defineKeyword("%", {treatAs: "identifier", context: {operator: "|>", operand: 2}}, topicReference);
    // having no parse/emit calls in a TD is logically equivalent to a final line reading:
    // for await (const node of parser.parseNodes()) parser.emit(node);
}
function pipeOperator(parser, expr, context) {
    if (expr.rhs.isKeyword("%")) {
        parser.emit(expr.lhs);
    } else {
        // expr.state is a container to store TD-local metadata. It is not visible to other TDs or to
        // the emitted AST.
        expr.state.topicVariable = context.newSyntheticLocal();
        // this emit will trigger the keyword handler for topic references in the rhs
        parser.emit(Parser.syntheticExpression`${topicVariable}=${expr.lhs},${expr.rhs}`);
    }
}
function topicReference(parser, expr, context) {
    parser.emit(context.state.topicVariable);
}
Parser.registerImplementation("pipeline", pipeline);
```

If the PA implementation accepts registration of this TD, then the application could then expect to be able
to parse and execute a file containing the following:

```js
syntax "pipeline"; // a syntax directive takes effect after the newline following the statement, so just after here →
console.log("foo" |> one(%) |> two(%) |> three(%));
```

(This could not appear in the _same_ file, which has already been fully parsed, except as part of an `eval` or
`new Function` or other such dynamic code generation, but it could appear in a file parsed after the point when
the `registerImplementation` call executes successfully. This proposal makes no changes to ECMA262 parse or
import timing with the exception of the optional `syntax import ... from ...` form of the `syntax` directive,
described in [Specifying Transformations > Inline](#inline))

When a TD is called as part of parsing, it is called once the input stream is open (but, potentially, before any data
is available to it). As a _logical_ description, it operates on a node-and-tree view of the source text, but the API
is designed such that a scanning parser or streaming transpiler can interface with it without backtracking, and with
a minimum of buffering. For example, the `define` methods on the parser object take a `context` option that specifies
both where the token is valid (the topic reference token is only valid in the rhs of a pipeline operator, for example)
as well as the maximum amount of context required to emit. If the pipeline operator here specified `"toplevel"` for
its context, for example, a streaming transpiler would have to buffer output for every top-level statement or
declaration, not emitting until it could be sure there were no pipeline operators anywhere in, say, the class
definition. However, since it specifies `"block"`, the transpiler is only required to buffer as far as the nearest
block statement.

This also exemplifies why the primary output of the PA process is an AST, not a raw source text. An augmenting
transformation may need to make non-local changes to the source tree, which is much easier when it is already
in a non-linear format; and, since one of the goals of this mechanism is that it _should_ be possible to implement
it at runtime, in a running engine, with a minimum of overhead, it is sensible to use a format that will be as
close as possible to what these engines expect to consume.

(Standing reminder: this API and these semantics are not set in stone and will change as new requirements emerge.)

Possible examples of the various types of transformation:
- "base64": Non-syntactic, producing ArrayBuffer(s)
- "gzip": Non-syntactic, producing ArrayBuffer(s)
- "rot13": Non-syntactic, producing string(s)
- "binaryast": Non-syntactic baseline, producing AST
- "comment": Non-syntactic baseline
- "json": Non-syntactic baseline. Could instead be implemented as a syntactic baseline
- "module": Syntactic baseline, forces parser into ES Module mode
- "script": Syntactic baseline, forces parser into Script mode
- "typescript": Syntactic baseline. Could be implemented as augmenting
- "pipeline": Syntactic augmenting

Note of course that the PA mechanism could be used to describe _reverse_ transformations, e.g. producing a
binary AST output from valid ECMAScript code. This is outside the scope of the PA proposal.

The "script" and "module" modes could be used by an engine to autodetect whether a file should be treated as a
Script or as a Module. Engines are not required to support this; in many contexts, parsing a module as a script
or vice versa will cause a load error, and in that case the engine _should_ throw an error. However, in cases where
either a script _or_ a module can be used (like a `<script>` tag or a Worker source) this would allow for omitting
the `type` attribute/property at the import site, decoupling the implementation from the referencing document.

The "comment" mode is proposed for inclusion in the standard, providing similar developer utility to C's `#if 0`
construct. It parses all following input as a multiline comment up until a directive changing the parse mode.

TC39 could additionally define modes that act as baseline transformers but remove the support for deprecated syntax
forms from the language. This would allow it to reclaim syntax space in the language, at the cost of gating some
newer syntaxes behind the equivalent of a Python `from __future__` statement.

### Runtime PA / No Exhaustiveness Requirement

Transformation Descriptions, especially those used by runtime PA implementations, are _symbolic_. They are designed
to be as simple as possible, using as simple syntax as possible, so that they can be verified at the source level
as well as in terms of their output. This necessarily means that they are context-dependent; as they are describing
behavior that the parser can take, they can use APIs that are specific to operating in the parser context.

For example, a hypothetical open-source project might release an engine that can execute both ECMAScript code and
Python code; it contains both kernels and uses an IPC mechanism to pass values and execution between the domains.
The integration is as smooth as it could possibly be: you simply `import` a `.py` file as though it were a `.js`
file, and you can access it just like it were a native ES Module. All this is possible today, and has probably
already been done, many times.

So where does Parser Augmentation fit into this picture? PA gives us a way to describe what is happening, so that
the interface between the "syntax you're importing" (remember, the meaning of `import` is "parse this ECMAScript
file and load it as a module") and the rest of the ECMAScript environment is well-defined.

Let's create a Transformation Description for this hypothetical engine. It exposes all its Python plumbing APIs
through a non-standard top-level symbol, `PythonVM`. You can access a property on `PythonVM` according to the
version of Python you care about if you want, or you can just use `PythonVM` itself to use the default version;
in either case, there are only two methods we care about:

- `PythonVM.compileScript(text)` is an async method which tells the Python runtime to compile a given Python
  source text. It returns a promise that resolves to an object with a `bytecode` property containing the bytecode
  in an ArrayBuffer and an `exports` property which is an array of strings like the following:
  ```js
  "export let foo = PythonVM.topLevelGlobals['foo'];"
  ```
- `PythonVM.executeBytecode(bytecode)` does exactly what it says on the tin. It executes the bytecode as a Python
  script, storing the toplevel variable context in the property `topLevelGlobals`. Synchronous.

We can reproduce the engine's native behavior using these two methods as follows:

```js
async function parsePython(parser, {version}) {
  parser.setTokenizerMode(Parser.TOKENIZER_STRING_CONSTANT);
  const scriptToken = await parser.parseToken();
  const {bytecode, exports} = await PythonVM[version].compileScript(scriptToken.value);
  parser.emit(Parser.syntheticAST`PythonVM[${version}].executeBytecode(${bytecode}); ${exports}`);
}
```

The `parsePython` function is a _Transformation Description_. It doesn't matter that it can only run on this one
specific engine, and the implementation is not provided. If you wanted to run your JS/Python program on an older
version of the engine, one that supported the `PythonVM` API but not the automagic import behavior from ES to
Python based on file extension, then you could use the code

```js
Parser.registerImplementation("python", parsePython, {polyfill: true});
```

to register that TD with the system if there's no definition for "python", and then use some yet-to-be-determined
method to register it as a handler for ".py" files imported by this file. Then you've achieved full parity
with the newer version. The TD could even be used with another engine entirely, as long as there's some way to
implement the `PythonVM` API - like, say, a third-party extension, or an [NPM package][python-wasm].

### Validator Transformations

Imagine now that a second JS engine decides to add Python support to their implementation. This project does it a
different way, though. Rather than welding an entire Python engine to the JS engine, it instead syntax-translates the
Python code into equivalent JS code, polyfilling enough Python system functions that standard Python libraries are
_mostly_ usable - the WINE solution. It works well enough that you can use most of the simpler pure-Python packages out
there. And, since the Python code translates into native JS and doesn't require any inter-engine IPC, the
syntax-translated version actually runs faster in most scenarios.

However, it's not a good match for your project. Your use of Python is solely for interfacing with the Python bindings
of a native library that doesn't have a JS binding, and so it won't work if it's syntax-translated. If the engine
can't run a native-Python version of the `.py` "module", you need to fall back on your pure-JS implementation.
You can test to make sure the `PythonVM` API is available before you `await import()` the module, of course, but how
can you be certain that what you import _is_ the `PythonVM` implementation?

Well, we can start out by taking the above TD function and registering it as a validator, in addition to a polyfill:

```js
Parser.registerImplementation("python", parsePython, {polyfill: true, validate: true});
```

- A transformation that is registered with the `validate` option set to `true` is a **Validator Transformation**.
  These will always run on any input being transformed by that identifier. It may or may not be selected by the
  host as the implementation to use when parsing, but if it is _not_ used as the actual parse-time implementation,
  the host must execute (an equivalent of) the validator TD as well. If the AST produced by the validator TD
  does not match the AST produced by the original parser (using a definition of "match" to be workshopped later),
  the host must reject the transformation before any results of the resulting AST are observable - most likely,
  by throwing a `SyntaxError`.

Now, you can be sure that if your `await import()` succeeds, it means that this code was exactly the right code,
produced by the same compiler, running on the same version, exporting the right exports. If your project runs
on a syntactic-transforming Python⇒JS engine, it will see that the generated AST is not the same as the
`executeBytecode` AST produced by your validator, and it will fail the `await import()` so that you can fall
back to the JS implementation.

That's a lot of extra work being done, though. We don't have to compile the entire Python file ourself to know
that a syntax-translated AST is wrong. And what's more, this validator will run even on the native-Python engine,
which means that you'll be compiling every Python file twice. So, let's simplify the validator function. All
you really care about is that some version of Python is executing the file _as bytecode_, i.e. in an actual
Python engine. Copy the parsing TD and change the initial lines about compilation, and you get this:

```js
async function parsePython(parser, {version}) {
  parser.setTokenizerMode(Parser.TOKENIZER_STRING_CONSTANT);
  const scriptToken = await parser.parseToken();
  const {bytecode, exports} = await PythonVM[version].compileScript(scriptToken.value);
  parser.emit(Parser.syntheticAST`PythonVM[${version}].executeBytecode(${bytecode}); ${exports}`);
}
function validatePython(parser) {
  const version = Parser.ANY_STRING;
  const bytecode = Parser.ANY_ARRAYBUFFER;
  const exports = Parser.ANY; // ANY is 0 or more nodes, ANY_NODE is exactly 1
  parser.emit(Parser.syntheticAST`PythonVM[${version}].executeBytecode(${bytecode}); ${exports}`);
}
Parser.registerImplementation("python", parsePython, {polyfill: true});
Parser.registerImplementation("python", validatePython, {validate: true});
```

The `parsePython` TD function is now registered as a polyfill, to enable functionality on the versions of the hybrid
engine that don't natively support `.py` import, and the `validatePython` TD function is registered as a validator.
Since `validatePython` never makes a parse request and simply runs to completion, it is a _passive_ transformation;
it isn't dependent on the input file, so the PA engine can use it to decide which transformation algorithm to run
in the first place. If the engine has both a native-Python and a syntax-translated-Python mechanism (perhaps one is the
fallback for the other), the `validatePython` validator would let it know that it needs to use the native-Python
implementation. If it _only_ has a syntax-translated version, it would know the translation was wrong as soon
as it emitted an AST node that wasn't a `PythonVM` call.

### Abstract Syntax Tree

The AST format in use will be modeled after Babel's AST. No need to reinvent the wheel on this one; the TC39
Binary AST proposal [also plans to use the Babel AST][binary-ast-prototype].

### Future-proofing ECMAScript

The polyfill capability can also be used to give engines future-compatibility for new ECMAScript syntax, even if
development on the engine stalls. Let's imagine a standard convention in which engines will report any syntax they
have native compatibility for as a null transformer. So, inventing a little bit of API here, we can imagine the
following:

```js
console.log(Parser.getAvailableTransformations());
// Prints an object that looks like:
{
  "ES2022": function() {},
  "ES2021": function() {},
  "ES2020": function() {},
  ...
}
```

This engine supports up to ES2022 syntax, but not ES2023. The only syntactical change made in the ES2023 specification
since ES2022 was the addition of the hashbang grammar, which we can represent as the following TD:

```js
async function downlevelES2023(parser) {
  parser.emit(Parser.syntheticAST`syntax "ES2022"`);
  if (await parser.peekBytes("#!")) {
    parser.setTokenizerMode(Parser.TOKENIZER_COMMENT);
    const commentToken = await parser.parseToken({until: "\n"});
    parser.emit(commentToken);
  }
}
Parser.registerImplementation("ES2023", downlevelES2023, {polyfill: true});
```

Now, when this engine encounters a file that is imported specifying the "ES2023" transformation (See [Specifying
Transformations](#specifying-transformations) below), it will execute the `downlevelES2023` function.

The first thing this function does is to emit a `syntax "ES2022"` directive into the syntactic stream, which
will cause the parser to execute its ES2022 transform. Since it supports ES2022 natively, that transform is a
no-op function. Then it peeks the bytestream to see if it starts with `#!` (a non-syntactic request, making this
a non-syntactic transform - we're not defining `#!` as an operator, we're just looking for it in this one point
in the bytestream). If it finds it, it will switch its tokenizer to the mode that parses everything as a comment,
and it consumes one line of input, emitting a comment token into the AST as the first top-level item. This happens
before the syntactic parser ever gets the first byte of input, which will start at the second line.

The first engine that supports runtime polyfillable PA is the first one that supports _all_ versions of ECMAScript,
past, present, and future.

## Specifying Transformations

Defining transformations does very little good if they can't be applied to source files. Parser Augmentation
specifies three mechanisms that can be used to define transformations that should be applied to a given source text:

### Inline

This is intended as the primary means of specifying syntactic transformations in an ECMAScript-equivalent source file.
Because syntactic transformations can change the meaning of source text, like `import` statements change the meaning
of identifiers, putting them in the same file offers many advantages:

- Editors and IDEs can use a leading syntax directive to detect file type. Since a TypeScript file with a leading
  `syntax "typescript";` statement can be considered a compliant ECMAScript source file, it's reasonable to use a
  ".js" extension for it.
- Developers can expect to have a single place to look to determine what syntax is being used in this file, if they
  see unfamiliar syntax.
- Importing modules do not take an implicit dependency on the implementation of the module.
- The syntax can be used in any location that expects ECMAScript code.
- Depending on implementation support, a single file could contain multiple syntaxes. A developer could write a file
  primarily in JavaScript but include a top-level block of TypeScript that defines types used by editor autocompletion.
- The file does not become unparseable when removed from its context.

The `syntax` directive comes in three forms:

1. `syntax <transformation-path>;`
2. `syntax push <transformation-path>;`
3. `syntax pop;`

A `<transformation-path>` is a comma-delimited list of transformations to apply from this point forwards. The semantic
rules listed above must still apply: non-syntactic transformations must occur before syntactic transformations, and
any baseline transformation must occur before any augmenting transformations. Each transformation in the list may
take one of the following forms:

1. `"transformation-id"` - requires that the given transformation be defined elsewhere, like the ECMA262 spec
2. `"transformation-id" with {parameters}` - parameters are passed in the second argument to the TD function
   (if executed), must be string-keyed. Advisory NLTH (see below) before `with`, otherwise parsers will end too early.
3. `import "transformation-id" from <module-specifier>` - forces parsing to stop at the end of the statement line until
   `<module-specifier>` has been imported and executed. The transformation-id must be defined in the module. If
   the parser already has a specification for "transformation-id", it is permitted to switch to that mode immediately
   and continue the parse, but it is not permitted to release the parsed AST to the rest of the engine until it
   has verified that the defining module does not contain a _conflicting_ implementation of the transformation.
   This can also take a `with`, immediately prior to the `from`; Advisory NLTH before both keywords, and one before
   a `with` in the `<module-specifier>` (this is a difference from the usual Import Attributes syntax as described
   in [this issue][import-attributes-nlth]; in this case it's not ASI hazard but parse hazard)
4. `from [<transformation>, <transformation>]` - including multiple transformations in an array-like syntax indicates
   that any of the listed transformations can be applied; the first one that succeeds will be used. This allows for
   dynamic selection of transformation implementations, depending on engine support. Note that this is _array-like_
   and not a true array syntax; the "members" of this "array" are transformation specifiers, which can include
   non-Expression syntax. The `from` keyword before the opening square-bracket is required to disambiguate this
   from member access syntax, and has a mnemonic meaning of "from amongst the following:".
5. `null` - the null transformer always succeeds, but it has no effect. It can be used as the final alternative
   in a set of alternate transformations; for example, the directive
   ```js
   syntax from [
     "typescript",
     import
       "typescript" with {version: "4.4"}
     from "https://www.typescriptlang.org/syntax",
     null];
   ```
   is a request to transform this source by the built-in `"typescript"` transformer, if available; by the
   (obviously currently non-existent) official syntax description for TypeScript 4.4, if unavailable and the
   runtime supports transformation polyfills; or simply parse the rest of this file untransformed if both of
   those fail. (It would probably fail, unless the file were syntax-compatible with ECMAScript.)

The "advisory NLTH" before the keywords is actually a specific instance of a general rule, which is that _no
leading proper subset of lines may form a legal `syntax` directive_. The NLTH is not required within the
square-bracket alternate transformation syntax, for example, because the parser is waiting for a close bracket.
(Alternative: requiring a semicolon at the end of a `syntax` directive would remove the NLTH requirement, but it
would block ASI.)

The `syntax push` form of the directive allows for temporary modification of the parse mode. A subsequent
`syntax pop` disables that `syntax push` directive, as well as all bare `syntax` directives that have occured since.
The "comment" transformer has explicit support for this syntax; it treats all following source text as a multiline
comment up until it encounters the 11-byte sequence `"\nsyntax pop"`. It then parses and emits the `syntax pop`
directive using the default ECMAScript parser, which will return the parser to the state it was in prior to the
switch to "comment" mode. Combined with transformer alternative syntax, it allows conditional inclusion of
source text depending on engine capabilities:

```ts
// Standard ECMAScript code

syntax push from ["typescript", "comment"];
// this will either be parsed as TypeScript or ignored
interface InterfaceDeclaration {
    foo: string;
    bar: number;
}
// the following line will be parsed no matter what
syntax pop;

// Standard ECMAScript code
```

There are a couple places where newlines are forbidden because of ASI hazard, and the transformation path may not
have a trailing comma unless the statement ends in a semicolon; because of these restrictions, the parser is
guaranteed that the first newline that _can_ end the statement, _is_ the newline that ends the statement.

### Import Attributes

Transformation paths can be specified using import attributes. The exact syntax is an open question, but since the
JSON Modules proposal uses the following syntax:

```js
import json from "./foo.json" with { type: "json" };
```

And given that the "JSON Modules" type [can be viewed as one specific transformation][type-is-definitely-a-transform],
it makes sense to use the same attribute. This format allows three different syntaxes:

1. a bare string, like above, representing a single transformation
2. an object with a "name" property representing the transform and possibly other properties, which will be passed
   in the second argument to the TD function
3. an array of either of the above two syntaxes, representing a Transformation Path.

The "importing" and "alternative" forms have no equivalent and are unsupported in this syntax.

### Out-of-band

Hosts can define out-of-band mechanisms both for registering transformations and for specifying them. For example,
a transpiler or bundler could register transformer plugins listed in a `package.json` file, and its configuration
file could map file extensions or directory trees to transformation paths.

These mechanisms are generally out-of-scope, but this specification suggests using the `+` character to separate
subsequent transformations in a path in cases where an entire path must be represented as a single string, for
example in an HTML attribute. Parameters could be represented as `;name=value`, URL-encoded.

### Composition of Transformation Paths

When multiple sources specify a Transformation Path for the same file, these paths are composed using the
following algorithm:

1. Start with an empty **working transformation path**.
2. Order the specified transformation paths as follows:
   1. Out-of-band, specified for this file in particular (for example, an HTTP header).
   2. Out-of-band, specified generally (for example, a file-extension rule)
   3. Import attribute
   4. Every `syntax` or `syntax push` directive in the source that has not been deactivated by a `syntax pop`,
      in source order
3. For each **specified path**:
   1. For each transformation in the specified path that also occurs in the working path,
      overwrite the working path properties object with the specified properties as though `Object.assign` were
      called on it.
   2. For each transformation in the specified path that does _not_ occur in the working path, append it to the
      working path.

For example, consider the following scenario: An ECMAScript module imports the file "./foo.ts", specifying the
transformation path `"typescript", "pipeline"` in an import attribute. The engine has native support for ".ts"
files, described as a single-element `"typescript"` transformation path. The foo.ts file is fetched from a server
that specifies it is to be transformed via the `"gzip"` transformation. The file begins with a directive of
`syntax "pipeline" with {pipeStyle: "F#"};`, and later in the file, the parser encounters the following directive:

```js
syntax push "comment";
```

The engine could determine the appropriate transformation path to use after this line as follows:

1. Start with an empty working path.
2. Order the specified paths and layer them:
   1. `"gzip"` (out-of-band, file-specific)
      1. `"gzip"` does not occur in the working path.
      2. Append `"gzip"` to the working path.
      3. New working path: `"gzip"`
   2. `"typescript"` (out-of-band, general)
      1. `"typescript"` does not appear in the working path.
      2. Append `"typescript"` to the path.
      3. New working path: `"gzip", "typescript"`
   3. `"typescript", "pipeline"` (import attribute)
      1. `"typescript"` occurs in the working path and has no properties, it is ignored.
      2. New working path: `"gzip", "typescript", "pipeline"`
   4. `"pipeline" with {pipeStyle: "F#"}` (non-push syntax directive)
      1. The `"pipeline"` transformation appears in the working path, so the `pipeStyle` attribute is assigned
         to the working path properties object (the same one that was passed to the TD function, if it was
         already called).
      2. New working path: `"gzip", "typescript", "pipeline" with {pipeStyle: "F#"}`
   5. `"comment"` (syntax push directive)
      1. The `"comment"` transformation does not occur in the working path, so it is appended.
      2. New working path: `"gzip", "typescript", "pipeline" with {pipeStyle: "F#"}, "comment"`.

This algorithm provides a few useful guarantees:

- Transformations will only ever be added onto the end of a path (or popped off the end during a `syntax pop`).
- Non-syntactic, non-baseline transformations (byte-to-byte filters, like decompressors) will accumulate at the start
  of the path, and they will never be removed or change position. Thus, a developer can't accidentally break the
  gzip compression with a `syntax` directive.
- Non-syntactic transformations specified out-of-band for a specific file, like in HTTP headers, will always appear
  at the beginning of the transformation path. Thus, servers can employ reverse transformations on files, like
  compression, and provide the decompressor on the transformation path, _even if the compression method isn't supported
  by the transport protocol_. Engines could support ECMAScript-specific compression technologies like pre-shared
  dictionaries, and browsers could request and servers could specify them using HTTP headers.

### Execution Order

While the _actual_ execution order of code involved in parsing is implementation-specific and not defined by PA,
the _logical_ execution order - in other words, the execution order in a host that simply implements PA by calling
the TD functions during parsing - is defined as follows:

1. Initialize two lists, each the length of the transformation path, where each element is a queue that starts out
   empty. The first list is the _Non-syntactic request list_, the second is the _Syntactic request list_.

2. The TD functions in the path are called one at a time, starting from the beginning of the path. Each is passed
   its own instance of the `parser` object; non-syntactic transforms each have their own scanner configuration,
   but all the syntactic transforms in the path share a single parser configuration which is affected by
   `setBaseline()`, `defineOperator()`, etc, in the order the methods are called. The TD function executes
   up until its first parse request to the parser, which is an async method returning a Promise, or to completion
   if the function does not have a parse request, like the `pipeline` example above.

   The type of the first parse request determines whether this transform is _syntactic_ (`parseNode[s]()` call),
   _non-syntactic_ (`parseToken[s]()` call), or _passive_ (runs to completion). The requests are set in the appropriate
   list, at the position of the transformation in the path. (Thus, both lists are expected to have "holes" in them.)

3. While there is input in the input stream:
   1. If there are no requests in the non-syntactic request list, read a line of input and feed it to the
      syntactic parser.
   2. If there are any non-syntactic requests, dequeue a request from the first non-empty queue and fulfill it
      from the input stream. This may result in an `emit` from the associated TD function (discussed below) and
      will probably also result in another parse call (or a continuation of the asyncIterator contract).
   3. If the first non-syntactic request generates an `emit` of a `string` or `ArrayBuffer`, it is used to fulfill
      a request from the next non-empty queue in the non-syntactic request list; it does NOT dequeue the next request
      in the current queue. However, if it generates an `emit` of an AST node, it goes through the whole synthetic
      chain (see below). If there are no more non-empty queues in the non-syntactic requests list, it is fed to the
      syntactic parser.
   4. Assuming the first non-syntactic request made a (possibly implicit) non-syntactic request of its own, the
      child request is added to the queue at the same position in the appropriate list. A TD can choose to
      remove itself from the path by returning without making another request.
   5. Repeat from the top of step 3 ("While there is input...") as long as there is input in the stream.

4. If there are any requests in the non-syntactic request list, dequeue and reject the first one,
      ignoring any additional parse calls but propagating `emit` calls as usual. Repeat until there are no
      non-syntactic requests left.

5. (This step happens concurrently with step 3, as it is driven by the syntactic parser)  
   If there are any non-empty queues in the syntactic request list:
   1. For each queue, peek and see what kind of node the request at the head of the queue cares about. (Likely:
      "any top-level node", a specific operator, or a specific keyword.) Also note, for each position in the path,
      what handlers have been defined for various operators or keywords. Only one such handler can be defined for
      each `parser` instance for each operator or keyword, and it is a programming error to request a node using
      `parseNode` that also has a handler defined; implementations may handle that situation in any way,
      including adding both requests to the chain, ignoring one, or rejecting the entire transformation
      (throwing a SyntaxError).
   2. Whenever a node that is being requested is emitted by the parser or by a non-syntactic transform, use it to
      fulfill the first such request, dequeueing it from its queue if it isn't a handler. Just like the
      non-syntactic chain, any node emitted by a parser will fulfill a matching request from _later_ in the path,
      but not from its own spot in the syntactic request list. Any `string` or `ArrayBuffer` emitted by this
      request will be emitted at this point in the non-syntactic request chain, exactly as it would if this
      were a non-syntactic request.
   3. Requests made during the fulfillment of a request are added to the appropriate queue in the appropriate
      list. Normally-syntactic transformations may generate a non-syntactic request; for example, a transformer
      could define a Bash-style `<<EOT` operator, adding a non-syntactic request to the queue in the handler,
      continuing to add non-syntactic requests to its queue until it detects a line with EOT; then it emits
      a string constant node into the syntactic chain and refrains from adding another non-syntactic request.
   4. AST nodes emitted at any point in the tree are emitted to the same tree position as the node that triggered
      the request. So, if the AST represents `3 + %` and the `%` keyword triggers a handler in the syntactic
      request chain, and that handler emits an AST node of the identifier `temp7`, then the AST that passes
      to the next link in the chain will be `3 + temp7`.
   5. After dequeueing a request or adding a request to an empty queue, peek to see what kind of node the request
      at the (new) head of the queue cares about.

6. After the parser finishes emitting nodes, if there are any requests in the syntactic request list, dequeue
   and reject the first one, ignoring any additional parse calls but propagating `emit` calls as usual. Repeat
   until there are no syntactic requests left.



## API

There are two relevant APIs, the runtime Parser Augmentation API (shown in examples as `Parser.registerImplementation`)
and the statically analyzable Transformation Description API (implemented by the `parser` object passed to the
TD function).

Neither is defined at this time. If TC39 decides to take up the proposal, it will be developed in collaboration with
TC39 members. Otherwise, it will materialize during implementation according to technical need.

# Drawbacks

- There is no telling what TC39 will think of this; the proposal does not currently have a TC39 champion, and so it
  can't even be presented to them. Unless and until TC39 makes a decision on this front, the `syntax` directive
  will remain a transpiler extension, like the pipeline operator is currently. This does not mean it can't be used,
  since it won't be present in the output, but it does represent a forward compatibility risk.
- Browsers are unlikely to adopt the `syntax` directive any time soon, and they will likely never support user-defined
  or polyfilled transformers, no matter how cool it would be to type
  ```js
  import {array} from "./numpy.py" with {type: {name: "python", version: "3.7"}};
  ```
  Thus, transpilers will continue to bear the implementation burden of Parser Augmentation for the foreseeable
  future, and will likely always be the only implementations that support _all_ the features of PA.
- The TD function specification, if designed badly, could shackle implementations to a particular type of parser
  if they want to support PA. TD functions are intended to be generic and symbolic enough to be divorceable from
  the actual parser implementation _even if_ the parser chooses to call the TD function directly, but that
  depends on the design being sufficiently parser-agnostic. There will no doubt be a lot of bikeshedding on
  what an ideal TD structure would look like, both from the perspective of static analyzability and of performance.
- This mechanism could (and probably, at some point, _will_) be used as a way to obfuscate source code. I'm not
  entirely certain why, given that it _also_ provides the means to de-obfuscate it, but the possibility is there.
- This could end up being a way to fragment the ECMAScript language, with individual projects using their own
  syntactic variant of ECMAScript that someone would have to learn if they wanted to contribute, rather than
  just using TypeScript like the rest of us.&lt;/tongue-in-cheek&gt;
- A `syntax` directive represents a possible one-line attack vector that could change the meaning of an entire
  source file.  
  Counterpoint: an `import` directive represents a possible one-line attack vector that could change
  global object prototypes for an entire runtime.

These are the biggest reasons why Parser Augmentation is designed to be opt-in from _all_ participants. Browsers
are not required to implement it at runtime; transpilers are not required to directly execute user-supplied code;
developers are not required to use `syntax` directives; TC39 is not required to add it to ECMA262. The PA mechanism
is designed to be able function in any environment, regardless of participation, and still be useful.

# Alternatives and Prior Art

The most obvious alternative is the status quo. Transpilers are already capable of performing syntax transformations
on user code, and they can do so without shackling themselves to a particular design definition. There are any
number of non-standard ways to introduce non-standard syntax into an ECMAScript file; JSDoc allows you to embed
TypeScript code into block comments, decorators (which are themselves non-standard syntax, at present) are used
as signals to control a code transformer. None of the individual concepts in Parser Augmentation are new, but some
of the more relevant prior art includes:

- **Prolog `op/3`**  
  The Prolog language has directives like [op/3][prolog-op3] which allow changing parser behavior mid-stream; this
  is the particular behavior that inspired the `parser.defineOperator()` mechanic. It is common practice in Prolog
  development to define a novel syntax at the top of a given source file in order to make the rest of the file
  easier to understand; the Prolog programmable parser is a large part of the inspiration for PA.
- **C Preprocessor**  
  While technically just a macro expander, CPP's flexibility allows it to make effectively-semantic changes to the
  language itself. It is not at all unusual to see notations like
  ```cpp
  TRY {
    call_some_function(&arg);
  } CATCH(e) {
    perror(e);
  }
  ```
  in a C project, despite the fact that the language does not support try/catch functionality. That way, contributors
  can use a familiar and uncomplicated syntax to interface with whatever underlying representation that project uses
  to control and report errors. A well-designed set of syntax-like macros can make a given project _much_ easier
  to understand at the code level.

  The C `#pragma` mechanism is likewise a way for source text to interact with the parser during the parse process,
  and it illustrates the utility of the `syntax push` and `syntax pop` directives (the CPP equivalents, sadly, are
  non-standard across C preprocessor implementations).
- **Python `__future__`**  
  Python's `from __future__` statement is a statement that looks like an import but is actually a signal to the
  parser that it should switch its parse mode before continuing with the file. Utilizing it has allowed Python has
  to make backwards-incompatible syntax changes that the developer community can adopt at its own pace.
- **Java JEP Interfaces**  
  For all its quirks, modern enterprise Java has some of the best separation of interface and implementation in the
  field, and it does it by defining _standard patterns_ for developers to follow, rather than providing APIs
  for them to access. Do you want to write a REST API server? Okay, write your code to conform to JSR 370, JAX-RS.
  Do you want to embed a scripting language that end-users can write in? Interface with the script engine using
  JSR 223, the Java Scripting API. Projects can swap one JAX-RS implementation for another or even one scripting
  _language_ for another just by changing what dependencies are pulled in by the packager; none, or at least
  very little, of the project's code needs to change.

  PA provides a standard pattern for defining _exactly_ what steps a given transformation comprises. No more
  "I have to keep using Webpack because the syntax I'm using is only implemented by a Webpack plugin". Using a
  given syntax in a project becomes nothing more difficult than adding a transpiler-agnostic `devDependency` and
  including a `syntax` directive at the top of the file.
- **[TC39 Binary AST Proposal][binary-ast]**  
  This was the inspiration that led to PA's primary format being an AST. Browsers do not _want_ to parse ECMAScript
  source code. They would prefer to simply import a pre-parsed AST. Given that it's also much easier to perform code
  transformations at the AST level rather than at the byte level, the decision seemed obvious; with this formulation
  of PA, a given browser can unilaterally experiment with binary AST formats as long as they use a vendor-specific
  identifier for their PA implementation, and servers could dynamically choose whether to serve the original source
  or the equivalent, pre-generated BinAST representation.
- **[TC39 Pipeline Proposal][pipeline]**  
  The evolution of the ECMAScript pipeline operator has been tumultuous. Between internal disagreement on semantics
  (F# vs Hack) and open questions about what characters should be used for syntax to which there are no perfect
  answers, the proposal remains at stage 2 after nine years of development, while the JSON Modules proposal is
  at stage 3 after five, only awaiting inclusion of the Import Attributes proposal (also stage 3 after five years).
  PA would provide a mechanism whereby TC39 could accept _both_ semantics of the Pipeline proposal, allowing developers
  to choose the non-default syntax using a PA directive.

# Adoption strategy

As mentioned above, PA is designed to be an entirely opt-in mechanism at all levels and from all participants. No
developer will need to use it, and no engine needs to support it; but the more that do, the more useful it gets.
My tentative expectation for how this will go is roughly as follows:

1. Submit proposal to Babel. (This document.)
2. Get feedback from Babel team; use it to refine the architecture and figure out which things I haven't remembered
   to put in writing.
3. Implement a reference implementation of PA in the Babel parser; finalize first draft of the PA APIs; write
   demonstration TDs. Gather feedback from other transpiler projects to make sure the architecture doesn't
   represent a significant development burden; provide implementation support if necessary.
4. Revise proposal with implementation details and submit to TC39. Continue raising awareness among TC39 delegates,
   especially champions of early-stage proposals that would benefit from PA.
5. Reach out to authors of popular transpiler plugins/modules; assist with hooking their code into the PA framework.
   Start introducing `syntax "foo" from "node-package";` as a transpiler-agnostic way to support non-standard
   syntaxes in JS projects.
6. Guide server-side engines (Node, Deno, etc) on how to properly implement PA behind experimental flags.
7. Politely ask MDN to document the `syntax` directive as an experimental technology.

It gets a little hazy beyond there, but I think that likely by that point either it'll be popular or it'll be dead.

# How we teach this

The only particularly significant change to Babel users is that in some circumstances, they won't need to write
as much Babel configuration.

I do think, however, that it's important to teach developers that the `syntax` directive, especially in its
`syntax ... from` variant, can be dangerous if used incorrectly. Developers should also be warned against writing
transformations that use standard syntax in non-standard ways, with the same severity (and rationale) as adding
to the prototypes of global objects.

I think that "syntax" is probably a good term; it has a very "technical and low-level" feel to it, making it
a little scary to less-experienced developers, which is the right feeling to have. (Sort of like how "import"
is a friendlier and more approachable term than "require", only in the other direction.)

# Open questions

So, so many. The biggest is that I have no idea how TC39 will react to this. I've gotten a few remarks on the idea
from delegates (some reproduced below), but I don't really have a read on their enthusiasm or lack thereof for the idea,
only that they don't quite understand the mechanism yet.

Another consideration is that, while I'm quite familiar with low-level parser techniques in general, I have no
direct experience with Babel's internal code. I've tried to structure my examples to be fairly obvious to people
with parser experience, and while I _believe_ this code architecture will be fairly easy to slot into Babel's existing
parser, there could be techniques or considerations at play that make the PA mechanism more difficult to implement
in Babel's particular context.

(Note that I'm not asking for development support in specific on this, it'll just take me a little while to get used
to the Babel codebase.)

Finally, the question of security. This isn't as hypothetical in the transpiler environment as I'd like it to be,
given the risk of supply-side attacks, and I'm not confident I've thought of and mitigated _every_ avenue an
attacker might use. I've listed some of my reasonings below, but I'd appreciate backup on this aspect.

## Frequently Asked Questions

The comments that I've received are in quotation marks, and the **Q:** links to the source. Hypothetical questions
are not in quotation marks.

- [**Q:**](https://es.discourse.group/t/proposal-parser-augmentation-mechanism/2008/5)
  "It's highly unlikely that browsers will implement something that changes the syntax on the fly."  
  **A:**
  Browsers don't need to. They can keep throwing `SyntaxError` if they see a `syntax` directive, exactly as
  they do now. My expectation is that _if_ browsers decide to support user-defined syntax transforms, they
  will either (a) only support the Import Attributes method of declaring them, and/or (b) only support
  the `syntax` directive when it starts at byte 0 of the file.

- [**Q:**](https://es.discourse.group/t/proposal-parser-augmentation-mechanism/2008/7)
  "So far, the entirety of the language is at runtime - i doubt this feature would be the one to change that."  
  **A:**
  PA describes a functionality that is _equivalent_ to a process that could occur at runtime, specifically at
  import/parse time. Browsers being unlikely to support it at runtime for security or performance reasons doesn't
  change its nature.

- [**Q:**](https://github.com/tc39/proposal-pipeline-operator/issues/303#issuecomment-2053794772)
  [In reference to the mention of `"use strict"`] "Additionally, "no more modes" is a pretty common mantra on
  the committee, meaning, it wouldn't be viable to have a pragma that changes how a program evaluates."  
  **A:**
  PA doesn't change the _evaluation_ of a program at all - only its parsing. Once the program makes it
  into AST form, it evaluates the same no matter how it got there. That's not the case with `"use strict"`,
  which imparts semantic changes to the runtime.

- [**Q:**](https://github.com/tc39/proposal-binary-ast/issues/81#issuecomment-2057208961)
  [In reference to using PA to enable BinAST] "The polyfill adds a transform and doesn't eliminate any existing ones."  
  **A:**
  The polyfill is only used by engines _without_ native support for the "transformation". If a server sends
  a BinAST source file to a browser that does not support BinAST, it probably means the CDN failed its
  content delivery check. A BinAST-supporting browser would be expected to see the `syntax` directive
  (or HTTP header) and immediately switch to its native BinAST parser/importer.

- **Q:**
  What safeguards prevent a malicious actor from using Parser Augmentation as an attack vector?  
  **A:**
  The only contexts that can affect how code is parsed are contexts that already have arbitrary control over
  what code gets executed. In order for PA to be active during the parse of a file, one of three conditions must hold:
  1. A `syntax` directive appears in the source being processed, which means it was put there by someone with access
     to the source
  2. An Import Attribute was specified at the load site, which had executive control in the first place, and could
     have chosen to import something else instead. Note that this does not "corrupt" the module for someone else;
     a difference in Import Attributes means that it's logically a different module.
  3. An out-of-band method was used to specify a transformation, which is host-defined behavior. The host already
     has control over all code being executed. It will be important to make sure that implementers understand
     that applying a transformation is a security risk, and not to apply untrusted transformations.
  
  This is why the `syntax import ... from ...` variant is especially important to be cautious of - it means that
  any change to the module referenced (the transformation definition) can change the meaning of the current
  file. (This was already the case, but even more so now.) Very important that the module come from a trusted
  source.

- **Q:**
  What order do transformations "run in"?  
  **A:**
  The exact algorithm is listed above in the **Execution Order** section, but the approximate answer is
  "all the non-syntactic transformations, then all the syntactic transformations, in order".

- **Q:**
  Won't it take too long to call out to JS during the parse?  
  **A:**
  The intent is not for the parser to call the TD function directly, but for it to statically analyze the function
  (perhaps by examining the AST, perhaps by running the function once with mock inputs) and then use that
  information to set parser parameters. I intend to demonstrate all three modes (static AST, static execution,
  dynamic execution) in the reference implementation.

- **Q:**
  Okay, but what really does "statically analyze the TD function" mean?  
  **A:**
  The reason this is left so open-ended is to avoid limiting the options for a PA-compliant engine;
  static code analysis is an evolving field, and hosts should be able to use novel techniques to analyze and
  comprehend the purpose of a TD function. However, as a proof-of-concept, here is one possible implementation:

  1. When JS code calls `Parser.registerImplementation`, the PA engine calls the TD function, passing it a mock
     object that conforms to the `parser` instance API.
  2. Record all calls to the mock parser, associating them with well-known PA capabilities. If any unsupported
     or unknown capabilities are requested, fail the registration.
  3. Tests and comparisons that can be performed by PA are always implemented as boolean predicates with fixed
     comparison inputs, like the `parser.peekBytes()` method shown in the `downlevelES2023` TD shown in
     the [Future-proofing ECMAScript](#future-proofing-ecmascript) section. For each predicate called, execute
     the TD function again, returning `true` from the predicate once and `false` the other time. Build a decision
     tree based on the tests performed. Any unsupported or unrecognized test methods should, of course, be grounds
     for failing the registration.
  4. This style of tracing should be capable of fully analyzing functions that are limited to simple API
     connections and settings, like the ones shown in this document. For more complex TD functions, set a limit
     on the runtime of the analysis procedure and/or the number of additional executions queued, and fail the
     registration (or switch to an alternate analysis mode, or fall back to a dynamic PA impelementation that
     directly calls the TD during parsing) if the limit is exceeded.

  This method represents a minimum implementation of a static analysis, and the PA specification will continue
  to be developed with the goal of enabling this type of tracing analysis, which can be performed without access
  to the TD source.

  It is expected that engines that do support static runtime PA will advertise their support for various methods
  of static analysis that they support the same way they advertise support for other core functionality (MDN,
  caniuse, etc); there could also be an API that allows engines to report what types of TDs they will accept.
  One engine might allow all TDs by falling back to a dynamic implementation, while another might only support
  minor textual changes to its own native TDs as returned by a `Parser.getImplementations()` function, like
  changing the values passed to a configuration method. In this latter case, the engine might syntax-match the
  passed-in TD's AST against the ASTs of all its native TDs until it finds one that matches, or fail the registration.

  An engine could also allow _dynamic generation_ of TDs by exposing a builder-like API; a TD generated by
  such a builder would obviously come pre-analyzed.

## Related Discussions

- [Initial draft proposal on TC39 Discourse][parser-augmentation-discourse]
- [Request for comments on draft from Binary AST proposal][rfc-binary-ast]
- [Request for comments on draft from JSON Modules proposal][rfc-json-modules]
- [Request for comments on draft from Pipe Operator proposal][rfc-pipe-operator]
- [Request for comments on draft from Type Annotations proposal][rfc-type-annotations]

[binary-ast]: https://github.com/tc39/proposal-binary-ast
[binary-ast-prototype]: https://github.com/tc39/proposal-binary-ast#prototype
[import-attributes-nlth]: https://github.com/tc39/proposal-import-attributes/issues/142
[json-modules]: https://github.com/tc39/proposal-json-modules
[parser-augmentation-discourse]: https://es.discourse.group/t/proposal-parser-augmentation-mechanism/2008
[pipeline]: https://github.com/tc39/proposal-pipeline-operator
[prolog-op3]: https://www.swi-prolog.org/pldoc/doc_for?object=op/3
[python-wasm]: https://www.npmjs.com/package/python-wasm
[rfc-binary-ast]: https://github.com/tc39/proposal-binary-ast/issues/81
[rfc-json-modules]: https://github.com/tc39/proposal-json-modules/issues/31
[rfc-pipe-operator]: https://github.com/tc39/proposal-pipeline-operator/issues/303
[rfc-type-annotations]: https://github.com/tc39/proposal-type-annotations/issues/216
[type-is-definitely-a-transform]: https://github.com/whatwg/html/issues/5640#issuecomment-644220003
[VirtualModuleSource]: https://github.com/tc39/proposal-compartments/blob/master/2-virtual-module-source.md
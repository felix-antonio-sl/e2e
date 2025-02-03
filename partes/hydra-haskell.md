# Hydra-Haskell

Hydra is a type-aware data transformation toolkit which aims to be highly flexible and portable.  
It has its roots in graph databases and type theory, and provides APIs in Haskell and Java.  
See the main Hydra [README](https://github.com/CategoricalData/hydra) for more details.  
This Haskell package contains Hydra's Haskell API and Haskell sources specifically.  
Releases are available [on Hackage](https://hackage.haskell.org/package/hydra).

## Build

Haskell is the current source-of-truth language for Hydra, which means that most of the Hydra implementation is written either in "raw" Haskell or in a Haskell-based DSL.  
You can find the DSL-based sources [here](https://github.com/CategoricalData/hydra/tree/main/hydra-haskell/src/main/haskell/Hydra/Impl/Haskell/Sources);  
anything written in the DSL is also mapped into the generated Java and Scala sources.  
You can find the generated Haskell sources [here](https://github.com/CategoricalData/hydra/tree/main/hydra-haskell/src/gen-main/haskell).  
To build Hydra-Haskell and enter the GHCi REPL, use:

```bash
stack ghci
```

Or to enter the test environment:

```bash
stack ghci hydra:hydra-test
```

To run all tests at the command line, use:

```bash
stack test
```

## Code generation

It is a long-term goal for Hydra to generate its own source code into various languages,  
producing nearly-complete Hydra implementations in those languages.  
Both Haskell and Java are fully supported as target languages,  
which means that all of Hydra's types and programs currently specified in the Haskell DSL are mapped correctly to both Haskell and Java.  
Scala support, on the other hand, is partial and experimental at this time.

You can generate Hydra's Haskell sources by first entering the GHCi REPL as above, then:

```haskell
import Hydra.Codegen

writeHaskell "src/gen-main/haskell" mainModules
```

The first argument to `writeHaskell` is the base directory to which the generated files are to be written,  
and the second is the list of modules you want to generate (in this case, a special list containing all built-in modules).  
For individual modules, use list syntax, e.g.

```haskell
writeHaskell "src/gen-main/haskell" [rdfSyntaxModule, shaclModelModule]
```

To generate test modules, use:

```haskell
writeHaskell "src/gen-test/haskell" testModules
```

Java generation is similar, e.g.

```haskell
writeJava "../hydra-java/src/gen-main/java" mainModules
```

For Java tests, use:

```haskell
writeJava "../hydra-java/src/gen-test/java" testModules
```

Scala generation has known bugs, but you can try it out with:

```haskell
writeScala "../hydra-scala/src/gen-main/scala" kernelModules
```

There is schema-only support for GraphQL:

```haskell
import Hydra.Sources.Langs.Graphql.Syntax
import Hydra.Sources.Langs.Json.Model
writeGraphql "/tmp/graphql" [graphqlSyntaxModule, jsonModelModule]
```

Because GraphQL does not support imports, the GraphQL coder will gather all of the dependencies of a given module together,  
and map them to a single `.graphql` file.  
Hydra has a similar level of schema-only support for [Protobuf](https://protobuf.dev/):

```haskell
writeProtobuf "/tmp/proto" [jsonModelModule]
```

...and similarly for [PDL](https://linkedin.github.io/rest.li/pdl_schema):

```haskell
writePdl "/tmp/pdl" [jsonModelModule]
```

Note that neither the Protobuf nor PDL coder currently supports polymorphic models.

### JSON and YAML generation

JSON and YAML are slightly different than the languages above, in that they are pure data languages, without accompanying syntax for schemas (types).  
Hydra terms can be serialized to either JSON or YAML by first providing a type, then any number of terms corresponding to that type.  
For example:

```haskell
:module Hydra.Kernel
import Hydra.Codegen
import Hydra.Langs.Json.Serde
import Hydra.Dsl.Terms as Terms

-- Choose a graph in which to execute flows; we will use the Hydra kernel graph.
g = hydraKernel
flow = fromFlowIo g

-- Choose a type for terms to encode. In this case, we will be encoding numeric precision values.
typ = TypeVariable _Precision

-- Construct an instance of the chosen type. In this case, we construct a precision value, then encode it as a term.
term = Terms.inject _Precision (Field _Precision_bits $ Terms.int32 64)

-- Create the adapting coder
coder <- flow $ jsonStringCoder typ

-- Apply the encoding, which turns the term into a JSON string.
flow (coderEncode coder term) >>= putStrLn
```

For a more sophisticated example involving recursive types, use:

```haskell
typ = TypeVariable _Type
term = Terms.inject _Type (Field _Type_literal $ Terms.inject _LiteralType (Field _LiteralType_boolean $ Terms.record _UnitType []))
```

in place of the `Precision` type and term above.  
This defines a type (in this case, the type of all types), and also a term which is an instance of that type (so in this case, an encoded type).

## Haskell API

### Structures

The most important structural types in Hydra are `Type` and `Term` (provided in the generated [Hydra.Core](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/gen-main/haskell/Hydra/Core.hs) module in Haskell),  
and `Graph` and `Element` (provided in the generated [Hydra.Mantle](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/gen-main/haskell/Hydra/Mantle.hs) module).  
`Type` provides a datatype, and a `Term` is an instance of a known `Type`.  
An `Element` is a named term together with its type, and a `Graph` is a collection of elements.  
A `Module` is a collection of elements in the same logical namespace, sometimes called a "model" if most of the elements represent type definitions.  
The main purpose of Hydra is to define and carry out transformations between graphs,  
where those graphs may be almost anything which fits into Hydra's type system -- data, schemas, source code, other transformations, etc.  
"Graphs" in the traditional sense are partially supported at this time, including property graphs and RDF graphs.

Types, terms, graphs, elements, and many other entities are parameterized by an annotation type, so you will usually see `Type m`, `Term m`, `Context m`, etc. in the code.  
The most common annotation type is called `Meta` (which is just a map of string-valued keys to terms), so you will also encounter `Type Meta`, etc.

### Transformations

Transformations in Hydra take the form of simple functions or, more commonly, expressions involving the `Flow` monad  
(a special case of the [State](https://wiki.haskell.org/State_Monad) monad, which has been implemented in many programming languages)  
as well as a bidirectional flow called `Coder` and a two-level transformation (types and terms) called `Adapter`.  
All of these constructs are provided in the generated [Hydra.Compute](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/gen-main/haskell/Hydra/Compute.hs) module in Haskell,  
along with the `Context` type which you will see almost everywhere in Hydra;  
a `Context` provides a graph, the schema of that graph (which is itself a graph), a set of primitive functions, an evaluation strategy, and other constructs which are needed for computation.  
A context is part of the state which flows through a graph transformation as it is being applied.

In Haskell, you will often see `Flow` and `Context` combined as the `GraphFlow` alias:

```haskell
type GraphFlow m = Flow (Context m)
```

There are two helper types, `FlowState` and `Trace`, which are used together with `Flow`; a `FlowState` is the result of evaluating a `Flow`,  
while `Trace` encapsulates a stack trace and error or logger messages.  
Since `Flow` is a monad, you can create a `GraphFlow` with `f = pure x`, where `x` is anything you would like to enter into a transformation pipeline.  
The transformation is actually applied when you call `unFlow` and pass in a graph context and a trace, i.e.

```haskell
unFlow f cx emptyTrace
```

This gives you a flow state, which you can think of as the exit point of a transformation.  
Inside the state object is either a concrete value (if the transformation succeeded) or `Nothing` (if the transformation failed), a stack trace, and a list of messages.  
You will always find at least one message if the transformation failed; this is analogous to an exception in mainstream programming languages.

A `Coder`, as mentioned above, is a construct which has a `Flow` in either direction between two types.  
As a trivial example, consider this coder which serializes integers to strings using Haskell's built-in `show` function, then reads the strings back to integers using `read`:

```haskell
intStringCoder :: Coder () () Int String
intStringCoder = Coder {
  coderEncode = pure . show,
  coderDecode = pure . read}
```

The `()`'s indicate that this coder is stateless in both directions, which makes the use of `Coder` overkill in this case.  
For a more realistic, but still simple example, see the [JSON coder](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/main/haskell/Hydra/Ext/Json/Coder.hs), which makes use of state for error propagation.  
For a more sophisticated example, see the [Haskell coder](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/main/haskell/Hydra/Ext/Haskell/Coder.hs)  
or the [Java coder](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/main/haskell/Hydra/Ext/Java/Coder.hs);  
these make use of all of the facilities of a graph flow, including lexical lookups, type decoding, annotations, etc.

### DSLs

Constructing types and terms directly from the `Type` and `Term` APIs mentioned above is perfectly correct, but not very convenient.  
For example, the type of all lists of strings may be expressed as `TypeList $ TypeLiteral LiteralTypeString`,  
and a specific instance of that type (a term) may be expressed as `TermList [TermLiteral $ LiteralString "foo", TermLiteral $ LiteralString "bar"]`.

Since all of the work of defining transformations in Hydra consists of specifying types and terms, we make the task (much) easier using domain-specific languages (DSLs).  
These DSLs are specific to the host language, so we have Haskell DSLs in hydra-haskell, and (similar, but distinct) Java DSLs in hydra-java.  
For example, the type of a list of strings is just `list string` if you include the [Types](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/main/haskell/Hydra/Impl/Haskell/Dsl/Types.hs) DSL,  
and the specific list of strings we mentioned is just `list [string "foo", string "bar"]`, or (better yet) `list ["foo", "bar"]` if you include the [Terms](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/main/haskell/Hydra/Impl/Haskell/Dsl/Terms.hs) DSL.  
There is additional syntactic sugar in Hydra-Haskell which aims to make defining models and transformations as easy as possible;  
see the [Sources](https://github.com/CategoricalData/hydra/tree/main/hydra-haskell/src/main/haskell/Hydra/Impl/Haskell/Sources) directory for many examples.

### Phantom types

A minority of Hydra's primary sources, rather than providing models (type definitions), provide collections of functions.  
For example, look at [Basics.hs](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/main/haskell/Hydra/Impl/Haskell/Sources/Basics.hs)  
or [Utils.hs](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/main/haskell/Hydra/Impl/Haskell/Sources/Adapters/Utils.hs).  
There are not many of these files because the syntax for constructing transformations natively in Hydra DSLs is still in flux,  
but you will notice that the type signatures in these modules look very different.  
For example, you will see signatures like `Definition (Precision -> String)` which appear to use native Haskell types such as `String`,  
or generated types like `Precision`, rather than Hydra's low-level constructs (`Type`, `Term`, etc.).  
This is a convenience for the programmer which will will be expanded upon as more of Hydra's kernel (indispensable code which is needed in each host language)  
is pulled out of raw Haskell and into the DSLs.  
If you are curious how these types work, see the [Phantoms](https://github.com/CategoricalData/hydra/blob/main/hydra-haskell/src/main/haskell/Hydra/Impl/Haskell/Sources/Phantoms.hs) model  
and [these slides](https://www.slideshare.net/joshsh/transpilers-gone-wild-introducing-hydra/34).  
Phantom types are available both in Haskell and Java.

## Creating a new Hydra implementation

Joshua Shinavier edited this page 1 hour ago · 15 revisions  
Hydra currently has implementations in Haskell (Hydra-Haskell) and Java (Hydra-Java).  
Hydra-Haskell is more complete than Hydra-Java, in that it implements the entire Hydra Kernel, although Hydra-Java has a number of utilities which Hydra-Haskell does not.  
Another criterion for completeness is that the implementation supports the entire Hydra standard library, which both Hydra-Haskell and Hydra-Java do.  
Prior to the 1.0 release, Hydra will have at least two complete implementations, but possibly more.  
The following is a rough guide to creating a Hydra implementation in a new host language, like Python or C#.

### Step 1: create the syntax model

The very first thing you should do when undertaking a new implementation is to define the syntax of the host language using Hydra's native data model, Hydra Core.  
This syntax model will be the foundation of everything which follows, so it is essential to get it right. Some considerations to keep in mind:
- Always rely on an existing source of truth for the syntax, such as an existing BNF grammar or reference documentation. These existing sources of truth can always be found for mainstream languages. Don't reinvent the wheel.
- There may be different sources of truth for a given language. Choose the one which is the most standard, or the best fit for your application. Also consider whether the source of truth will be supported and maintained over time.
- Languages may change over time. Be sure to note the version (if any) or timestamp at which the source of truth was retrieved, along with a URL, in the documentation for your new syntax model.

Look at other syntax models for inspiration, such as the one for Haskell and the one for Java. Note that the Haskell syntax model was written first, and was based on a third-party library (`haskell-tools-ast`) which has not been updated since 2019.  
The Java syntax model is newer, and follows the above advice of using a specific version of a standardized grammar as the source of truth.

It is a good idea to copy the entire source of truth verbatim, then enclose it in comments and add the Hydra type definitions inline beneath the corresponding productions in the SoT; this will be useful later when cross-referencing the SoT or making updates.  
Programming language grammars are usually quite large, so creating the Hydra syntax model may be one of the most time-consuming steps in this process.  
Utilizing an LLM-based tool like ChatGPT or GitHub Copilot can greatly speed things up, although you must carefully check and correct its output at every point; you will find that there are many design decisions which only you can make.

### Step 2: create a coder

Now that you have a model for your target language, you need a way of mapping native Hydra expressions into that language.  
You can split this task into three parts: a coder (this step), a set of language constraints (step #3), and a serializer (step #4).

A central idea of Hydra is that the kernel should be defined once, in a Hydra DSL, and then mapped into various host languages using language-specific coders.  
Currently, the Hydra kernel and also the language coders are written in Haskell.  
See the Haskell coder and the Java coder, and note that there are also many coders for other languages in which we do not have Hydra implementations; even more can be found in Hydra-Ext.

In terms of implementing Hydra in a new language, the purpose of creating a language coder is that all of the Hydra kernel code which is written in a Hydra DSL can be automatically ported to the new language, saving work and ensuring parity across implementations.  
The coder can then also be used to port other, non-kernel code.

While step #1 might have been very time-consuming but relatively simple, step #2 is more complex.  
LLMs will be of more limited usefulness here.  
Your coder must operate both at the type level (mapping Hydra type expressions into equivalent class or interface definitions in the target language) and also at the term level (mapping Hydra term expressions into equivalent values, data structures, and executable code).  
Writing such a coder is one of the most challenging development tasks in Hydra, and this simple guide will not attempt to break it down into a step-by-step recipe. However, some notes:
- The top-level structure of these coders conforms to a standard API; see how the coders are called from `Codegen.hs`.
- The coder only needs to be unidirectional; for programming language coders, we are usually not interested in mapping native programs in the target language back into Hydra Core.
- There will usually be a helper object, e.g. called "namespaces" or "aliases", which is passed into each function that deals with qualified names.
- The greater the semantic distance between Hydra Core and the target language, the larger and more complex the coder will need to be. For example, Haskell is very close to Hydra Core, so the Haskell coder is just over 500 lines of code. Java is more remote, and requires more than 1500 lines of code.

### Step 3: define language constraints

Characterizing a target language in terms of constraints (supported and unsupported type/term expressions) allows Hydra to do much of the work of adapting types and terms so they can be expressed in that language.  
In principle, this should really be step 2, but in practice it may be necessary to get to know the target language by writing the coder before formally defining the constraints.  
A language is given a unique name like "haskell" or "java", and is characterized according to:
- Supported type variants (type constructors). E.g. Haskell supports both record types like `LatLon {lat :: Double, lon :: Double}` and product types like `(Double, Double)`, whereas Java only supports the former.
- Supported term variants. Similar to supported type variants. While some data languages, like JSON or Protobuf, may be very constrained in their term expressions (e.g. no functions, no variables, etc.) a Hydra host language will need to support all of the lambda calculus constructors, plus term variants (like list, map, etc.) for all supported type variants (like list, map, etc.).
- Supported function variants. A Hydra host language must support all three (eliminations, lambdas, and primitives).
- Supported elimination variants (e.g. for projections, injections, folds, etc.). These will generally mirror the supported term variants.
- Supported literal variants. The options are binary, boolean, integer, float, and string. Only the last of these is strictly required, but a typical host language would include all five.
- Supported integer variants. The options are 8- to 64-bit signed and unsigned integers, as well as big integers. None are required; include as many as have a natural counterpart in the host language. For example, 16-bit signed integers in Java are called `short`, while big integers are called `BigInteger`.
- Supported floating-point variants. Similar to integer variants, but there are only three options: 32-bit and 64-bit floating-point numbers (i.e. `float` and `double`), as well as big float.

Language constraints are defined for every language coder in Hydra; see the pointers above.  
The Haskell language constraints are defined [here](#), and the Java language constraints are defined using the Haskell DSL [here](#).  
It is preferable to use the DSL, as that allows the language constraints to be propagated into each Hydra implementation.

**Note:** it is conventional in Hydra to define a list of reserved words for the target language in the same module as the language constraints.  
See for example the `reservedWordsDef` element in the Java constraints, or `reservedWords` in the Haskell constraints.

### Step 4: create the serializer

Now that you have the coder and the language constraints which allow you to map Hydra Core expressions into an abstract syntax tree for your language, you need to also map the abstract syntax tree into a concrete syntax.  
Compared to some of the steps above, this is one of the conceptually simplest tasks, and requires little explanation.  
It will be helpful to re-use the built-in `Hydra.Ast` and `Hydra.Tools.Serialization` modules here.  
See the Haskell SerDe and the Java SerDe as helpful examples.  
Note that while "SerDe" stands for "serializer and deserializer", you will only need the former; we have no need to map from expressions in the target language, as noted above.

### Step 5: implement standard primitives

As noted above, Hydra has a standard library of primitive, or built-in functions.  
There are many calls to these functions (though not all of them) in the Hydra kernel code which will be mapped into your new implementation, and you will get compile-time or runtime errors if they are not present when you attempt to compile or run the generated kernel code.  
Luckily, most of them are pretty simple and straightforward.  
Primitive functions may be implemented in different ways in different Hydra language variants, though their behavior must be the same across variants.  
In Haskell, they are implemented [here](#) (compile-time implementations) and [here](#) (metadata about the primitives).  
In Java, the metadata about a primitive is bundled together with its implementation [here](#), and a registry of primitive functions is constructed [here](#).

The main requirements for primitives are that:
1. The primitives are available to be referenced by compile-time code in the host language
2. The primitives are available to be called in an interpreted environment
3. All of the required metadata, including the type signature, of each primitive can be looked up using the primitive's name.
4. The runtime behavior of the primitive conforms to Hydra's language-independent test suite (see below)

See `Graph.hs` for the required fields: name, type, and implementation.  
Note that while in principle, every Hydra language variant ought to implement every primitive in the standard library, currently there are small differences (a handful of primitives are not yet implemented in Java).  
At a bare minimum, all of the primitives which are referenced in the Hydra kernel need to be implemented.

### Step 6: generate sources

Now the fun part. You have your code generation solution for the new host language, and you have your primitive functions.  
You are ready to translate Hydra's kernel code and test suite from the Haskell DSL into the compile-time environment of the host language.  
Examples for generating main and test sources are available on the Hydra-Haskell README, e.g. for Java:

```haskell
writeJava "../hydra-java/src/gen-main/java" mainModules
writeJava "../hydra-java/src/gen-test/java" testModules
```

Once you have generated the sources, make sure they compile.  
If not, iterate on your coder and/or standard library.

### Step 7: fill in the gaps

Hydra has not yet "closed the loop", i.e. not all of its kernel code is available in the DSL, and can be generated directly into your new language variant.  
After Hydra does close the loop, this step will go away completely, but for now you need to be aware of the holes in the kernel sources and patch them with hand-written code as needed for your anticipated applications.  
You will know what is needed, because your application will not compile and/or run if certain code is missing from the kernel.  
However, this code can be written "lazily", since if you do not yet need it, it's best not to spend the effort reimplementing it in your new language, since it will eventually be generated for you.  
One example is the Hydra interpreter, which is needed for running the test suite (see the next step).  
You can find the interpreter in Haskell [here](#) and in Java [here](#).

### Step 8: create a test runner

Hydra includes a suite of test cases which are intended to be evaluating in the same way across language variants, ensuring parity.  
The sources for the test suite can be found in the Haskell DSL [here](#).  
At this time, the coverage of the tests is quite minimal (it just tests a few list and string primitives), but it can be expected to expand in the future, covering all of the standard library as well as various kernel code such as core coders and type inference.  
A Hydra implementation must include a runner for the test suite which executes every time unit tests are run.  
You can see the test suite in Haskell [here](#) (generated code) and [here](#) (runner) and in Java [here](#) and [here](#).  
The generated code should already be available (from the previous step), and you can see that the new code you need to write by hand is quite simple -- less than 100 lines of code in each language variant.

### Step 9: create native DSLs

OK, now for the really fun part. This is where you depart from the script and create whatever domain-specific languages and/or additional utilities will be useful in your new language variant.  
While there are a handful of more or less mandatory DSLs, language variants are allowed and expected to vary widely in the DSLs they provide, so as to make the best use of the programming constructs and/or libraries available in the particular host language.  
You should probably start with these three DSLs, and then add others as desired:
- **Type construction DSL:** allows developers to build type-level expressions. See `Types.hs` and `Types.java`.
- **Term construction DSL:** allows developers to build term-level expressions. See `Terms.hs` and `Terms.java`.
- **Term decoding DSL ("expect"):** allows developers to decode Hydra terms to native programming constructs. See `Expect.hs` and `Expect.java`.

In addition, you may find it useful to define shorthand constructors for types and/or terms, shorthand primitives, phantom term constructors, and also individual DSLs for specific Hydra kernel modules, e.g., the hydra/core DSL in Haskell.  
The last of these are only necessary if you plan to define new source-of-truth modules in your language variant, rather than just receiving generated code from Hydra-Haskell; see for example the `CoreDecoding` source.

### Step 10: profit

Time to build some applications in your language.

## Hydra developers

Joshua Shinavier edited this page on Aug 1, 2024 · 5 revisions  
This page is for developers of Hydra itself. See [Contributors](#).

### Dependency tiers

Since Hydra generates much of its own source code, it is challenging to prevent circular dependencies.  
To make things easier, we categorize Hydra source code into several dependency tiers, and enforce the simple rule that code in a lower tier may not depend on code in a higher tier:

| Tier | Description |
| --- | --- |
| **tier-0 kernel modules** | Generated sources consisting mainly of type definitions, including `hydra/core`, `hydra/graph`, etc. These are not allowed to depend on primitive functions. |
| **tier-0 DSLs** | Language-specific DSLs, such as Hydra-Haskell's Types and Terms DSLs, which depend only on each other and on tier-0 kernel modules. The kernel modules are not allowed to depend on the DSLs. |
| **tier-1 kernel modules** | Generated code which is allowed to depend on (tier-1) primitive functions, as well as tier-0 kernel modules and DSLs (but not on tier-1 primitive functions). |
| **tier-1 primitive functions** | Primitive functions whose implementations are allowed to depend on tier-0 kernel modules and tier-0 DSLs. |
| **tier-1 DSLs** | Language-specific DSLs which are allowed to depend on tier-1 kernel modules and primitive functions. |
| **tier-2 kernel modules** | The rest of Hydra's generated kernel code. |
| **tier-2 primitive functions** | The rest of Hydra's built-in functions. |
| **tier-3 kernel sources** | These are the DSL-based definitions of tier-0, tier-1, tier-2, and tier-3 kernel code. For example, `Hydra/Sources/Core.hs` in Hydra-Haskell is the DSL-based definition of `hydra/core`, which is generated to produce the tier-0 `Hydra/Core.hs` module in Haskell, the corresponding `hydra/core` package in Java, etc. |
| **tier-4 non-kernel code** | Built-in language coders and utilities which are provided with an implementation of Hydra, but are not part of the Hydra kernel. For example, Hydra-Haskell provides a Java coder among many others, but this is not part of the kernel; other Hydra implementations might not provide a Java coder. |
| **tier-5 code (aka third-party code)** | Sources, generated code, and user-defined functions which are external to Hydra proper. Developers of third-party code may create their own dependency tiers within this one, but these are not specified by Hydra. |

### Releases

#### Hydra-Haskell release process

Find previous Hydra-Haskell releases on Hackage. You will need a Hackage account and appropriate permissions to publish new releases. This is the process I usually follow:
- **Pre-release readiness checks**
  - Make sure the entire project has been brought up to date for the release. Check open issues, update documentation, etc., as needed.
  - Upgrade dependencies if this hasn't been done since the last release, taking care not to impact downstream consumers of Hydra
    - Bump `resolver: lts-xxx` in `hydra-haskell/stack.yaml` to the latest LTS release of Stack, then compile and run tests. Corresponding changes to dependencies in `hydra-haskell/package.yaml` may be required. Bump the version of the base dependency in `package.yaml` to the latest version supported by Stack. Also consider bumping the versions of the other dependencies in `package.yaml`, and remove any dependencies which are no longer needed. Compile and run tests.
    - Repeat the above step in `hydra-extensions`. This is not necessary for the release, but it keeps the repository self-consistent.
  - Commit, review, and push to main any changes we made as part of the readiness checks above.
- **Bump the version**
  - Update the Hydra version in `hydra-haskell/package.yaml`. Also update the corresponding versions in `hydra-java/build.sh` and in `hydra-extensions/package.yaml`
  - Update `CHANGELOG.md`, adding release notes.
  - Commit the change using a message like "Bumping Hydra version to 0.x.y".
- **Release to Hackage**
  - Create the Hackage release package (on main HEAD) with `cd hydra-haskell && stack sdist`
  - Upload the package here, then periodically check on the status of the release.
- **Tag the release**
  - As soon as the release is confirmed to have succeeded, create a tag (e.g., `git tag 0.x.y -m '0.x.y release' [hash]` && `git push --tags`) using the commit hash of main HEAD at the time of the release.

#### Hydra-Java release process

In most cases, Hydra-Java releases should be coordinated with Hydra-Haskell releases, the exception being bug fix or minor feature releases which concern only Hydra-Haskell or only Hydra-Java. The following additional steps are required for Hydra-Java releases:
- **Update JavaDocs**
  - After bumping the release version (in hydra-java, hydra-haskell, and hydra-extensions), re-generate JavaDocs with `rm -rf hydra-java/javadoc && cd hydra-java && ./bin/update-javadoc.sh`. Now add files, commit, and push to main as needed.
  - Check out the remote docs branch of Hydra, then `git pull origin main` and `git push origin docs`. This will trigger the generation of the online JavaDocs. You should see notifications in the #github channel on the LambdaGraph Discord server. The generated Hydra JavaDocs can be found [here](#).
- **Release to Nexus**
  - Once master HEAD is ready for the release, `cd hydra-java && ./gradlew publishToSonatype`. Then,
  - Log in to [oss.sonatype.org](https://oss.sonatype.org), assuming you have the credentials to publish Hydra-Java
  - Find the staging repository in the left sidebar under "Staging Repositories", then click to "close" it (do NOT "drop"). Supply a message like "Hydra-Java 0.x.y release".
  - Wait several minutes, then click "release" when that option is available. Again enter a message like "Hydra 0.x.y release".
  - It will take a variable amount of time for the release to propagate, sometimes up to an hour. When the release is available, you will see it [here](#) as `net.fortytwo.hydra`.
- **Tag the release**
  - If you haven't already done so as part of the Hydra-Haskell release process (see above).

### Property graphs

Joshua Shinavier edited this page on Apr 6, 2023 · 3 revisions  
**Property graphs in Hydra**

Hydra traces its origins to Algebraic Property Graphs (APG), and the Dragon implementation of APG.  
There is significant potential to apply the APG approach to TinkerPop now that there is open-source tooling available.

APG, like Hydra's formal model, is a hypergraph data model which generalizes property graphs from loosely-typed vertices and binary edges to strongly-typed vertices and hyperedges (i.e., edges which may have other projections apart from "out" and "in").  
This helps it bridge the gap between typical record-shaped enterprise data (expressed in languages like SQL, Avro, or Protobuf) and graphs with a minimum of domain-specific mapping.

However, in order to support typical property graphs including those supported by the current version of Apache TinkerPop, we do need a mapping language; n-ary records need to be mapped down to binary edges.  
Such a mapping language is currently being developed for Hydra, and is captured in `hydra/langs/tinkerpop/mapping`.  
The main design considerations have been that:
- Mappings are expressed as annotations on source schemas
- Elements in the source shall be treated as property graph elements in the target.  
  I.e., we avoid complicated n-to-n mappings of source types to target types, instead treating the source data as if it were already a property graph as in APG.

An example of these annotations in the context of an Avro schema can be seen [here](#), and example JSON data is [here](#).

#### Annotation keys

There are several annotation keys, and their names are configurable. The default names are as follows:
- `@label (vertex)`: provides the label for the annotated element as a vertex
- `@label (edge)`: provides the label for the annotated element as an edge. The presence of other properties is used to distinguish edges from vertices if the `@label` keys are the same, as they are by default.
- `@id (vertex)`: provides a value specification (see below) for the id of the annotated element as a vertex
- `@id (edge)`: provides a value specification (see below) for the id of the annotated element as an edge
- `@key`: provides the key for an annotated field as a property
- `@value`: provides a value specification for an annotated field as a property
- `@outVertex`: provides an incident vertex specification (see below) for the out-vertex of the annotated element as an edge
- `@inVertex`: provides an incident vertex specification for the in-vertex of the annotated element as an edge
- `@outEdge`: provides an incident edge specification for an out-edge of the annotated element as a vertex
- `@inEdge`: provides an incident edge specification for an in-edge of the annotated element as a vertex
- `@outVertexLabel`: provides a label for the out-vertex of an edge where this cannot be inferred
- `@inVertexLabel`: provides a label for the in-vertex of an edge where this cannot be inferred
- `@outEdgeLabel`: provides a label for an out-edge from a vertex
- `@inEdgeLabel`: provides a label for an in-edge to a vertex
- `@ignore`: specifies that an annotated field should be ignored, rather than treated as a property

#### Annotation format

**Values**

Vertex and edge ids are given by patterns like the following:
- `${}`: the value of the annotated field
- `${path/to/value}`: a simple path pattern, the steps of which are field names
- `prefix[pattern]suffix`: a prefix or suffix appended to another pattern

For example, if a string-valued Avro field `airportId` is annotated with `@outVertex: "${}"`, this means that the entire value of `airportId` for a given record, like `"KSJC"`, is the out-vertex id.  
If the field is annotated with `@outVertex: "airport-${}"`, then the out-id is `"airport-KSJC"`.  
If the field is annotated with `@outVertex: "airport-${info/identifiers/icao}"`, then we expect there to be a field named `info` under the annotated field, whose type has a field named `identifiers`, whose type has a field named `icao`, which we use to construct the out-vertex id.

The `${}` pattern produces a native Hydra term according to the source schema, which might be a string value, an integer value, or even a record or other complex term.  
The property graph schema (see below) which you supply in your application determines how these terms are converted to the appropriate data type for the application.  
When a pattern includes a prefix or suffix, the resulting term is always a string, and the terms produced by `${...}` are converted to strings in a canonical way, based on JSON serialization.

**Vertices and edges**

Any record type annotated with `@id` is taken to be the source type for a vertex or edge type.  
The TinkerPop coder looks for additional annotations (indicating incident vertices or edges) to determine whether the records of that type should be treated as vertices or edges.  
While `@id` is mandatory, `@label` may be omitted, in which case the vertex or edge label is taken from the name of the source type.

**Incident vertices**

Incident vertex (out-vertex or in-vertex) specifications are simply value specifications for the vertex ids.  
If a plain path (like `${}` or `${nearestCity}`) is provided, then Hydra will resolve the path and see whether the type it resolves to is annotated as a vertex (with `@id` or `@label`).  
If so, then it will pick up the id and label from the annotations.  
If not, then it will treat the whole type as an id, and look for `@outVertexLabel` or `@inVertexLabel` as the label of the incident vertex.

**Incident edges**

Incident edge (out-edge or in-edge) specifications are handled similarly to incident vertex specifications.  
An out-edge specification starts with a known out-vertex and parses the value specification for the in-vertex, picking up the `@id` and `@label` annotations from the source type of the vertex if they are available, or looking for `@inVertexLabel` (for an out-edge) or `@outVertexLabel` (for an in-edge) if the value resolves to an id value rather than an annotated record type.  
There is currently no specification for properties of such edges.

**Properties**

By default, all of the fields of a record type which are not annotated with the keys given above are assumed to be vertex or edge properties.  
Similar to vertex and edge labels with respect to the `@label` annotation key, the property key is based on the name of the field unless `@key` is provided explicitly.  
For example, a field named `airportCode` might be annotated with `@key: "icao"` to indicate that the resulting property should be called `icao`, not `airportCode`.

The `@value` annotation allows the value of a field to be turned into a property value in the same way that identity-giving fields are turned into ids (using a value specification).

The `@ignore` annotation (with any value) tells the coder to ignore a given field, rather than treating it as a property.

**Schemas**

A schema, in this limited context of property graph mappings, is a helper object which provides the following information:
- Application-specific names for each of the annotation keys described above
- A coder (bidirectional encoder/decoder) for vertex ids, mapping Hydra terms to and from application-specific ids
- A coder for edge ids (analogous to vertex ids)
- A coder for property types, mapping Hydra types to and from application-specific property types
- A coder for property values, mapping Hydra terms to and from application-specific property values

The definition of the `Schema` type can be seen in the TinkerPop mappings module.
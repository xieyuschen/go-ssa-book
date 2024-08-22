
# SSA Program Build

Program build actual invokes the build function for each package. Serially or concurrently according
to the mode. For the external function call from a package, the built function could be found in another package
through the ssa program. Hence, we only need to take care of how the package build works.

The "build" means to emit the code for all package top-level definitions(functions, global variables, import directives, etc...)
Const and type needn't be built because they're definitions instead of expression which is runnable.

Type `builder` is defined to handle all AST-to-SSA conversion, it builds all functions/variables
inside the packages.

## Building Global Variables and Inits

Go SSA creates a **_synthesized package initializer_** to hold all logics related to package initialization, namely:

- all imported package _synthesized package initializer_
- initialization of global variables
- all `init` functions defined under the package

```go
	/* synthesized package initializer */
	p.init = &Function{
		name:      "init",
		Signature: new(types.Signature),
		Synthetic: "package initializer",
		Pkg:       p,
		Prog:      prog,
		build:     (*builder).buildPackageInit,
		info:      p.info,
		goversion: "", // See Package.build for details.
	}
```

Method `buildPackageInit` helps to build the function body of the _synthesized package initializer_.
The _synthesized package initializer_ has name _init_ in its SSA representation while
all `init` functions defined under the package in go code are named with format `init#%d`.

!!! tip "complain to the synthesized package initializer"
    The naming here is really confusing, the _synthesized package initializer_ ssa function
    is named as `init`, while the `init` functions from go source code `init` function definition(`func init{}`)
    has the name `init#%d`. Personally, i would rather use `sinit` or a different name for the synthesized
    package initializer ssa function name, instead of reusing the name `init`.

The function body created by `buildPackageInit` emits a $\phi$-node for the package initialization logic,
while all real logics are hold by the `init.start` block.

```go
		initguard := p.Var("init$guard")
		doinit := fn.newBasicBlock("init.start")
		done = fn.newBasicBlock("init.done")
		emitIf(fn, emitLoad(fn, initguard), done, doinit)
```

The pseudo-code performs as the snippet shown below:

```shell
if init$guard
  then init.done
  else init.start 
```

At the start of `init.start`, a `true` value is stored to the `init$guard` to ensure the `init.done` won't be executed twice.
Moreover, as the _ssa init function_ is synthesized without corresponding `go.mod` file, the go version and
info fields are empty.

## Building Function

There are two kinds of function, [function declaration](https://go.dev/ref/spec#Function_declarations) and
[function literal](https://go.dev/ref/spec#Function_literals).

```go
// example is a function declaration
func example(){
    // the 'func() {}' here is a function literal
    var f = func() {}
}
```

All functions are represented by `ssa.Function` no matter what kind of function it is(decl, lit).
SSA differentiates them from the `syntax` filed typed `ast.Node`.

When we start to build the function body, SSA creates an `entry` block and a map field `vars` inside
the `ssa.Function` to store the local variables.
(todo): what's the local variables here? the map 2 here is only for the values `n` and `err` like this, is it?

```go
func demo() (n int, err error)
```

Then the function parameters and results are spilled, which creates allocations on the stack for function body to load/store to interact
with the parameters. For example, the `input`(parameter) and `i`(result) will be spilled. SSA treats these parameters as syntactic.

```go
func hello(input int) (i int) {
	return input
}
```

After that,  `deferstack` is processed for [the `range-over` feature](https://github.com/golang/tools/commit/0006edc438850cff5bf8435cc6a1cc2a5fd909d5) and we can ignore it for now.

Then it starts to emit SSA representation for all statements inside function body by calling the respective emitting functions.
and we will discuss the emitting functions later.

Finally, after all statements inside function body are emitted,
SSA helps to add two instructions `RunDefers` and `Return` when the current block satisfy ` cb != nil && (cb == fn.Blocks[0] || cb == fn.Recover || cb.Preds != nil)`.
The `RunDefers` instruction helps to trigger all registered functions in FILO(the concrete logic is in method `runDefers`).
The `Return` instruction helps to return the values and control back to the calling function. Don't get confusing
between `Return` instruction and the keyword `return` in go.

That's how the go supports `defer` keyword to support function executions after return/panic.
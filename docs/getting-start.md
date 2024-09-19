# Getting Start

The whole guide introduce the SSA concepts and usages in golang based on [golang/tools ssa@v0.24.0](https://github.com/golang/tools/blame/v0.24.0/go/ssa/ssa.go)

<!-- todo: add an overview section to introduce the whole book -->

The SSA package defines a representation of the elements of Go programs from all top-level statements using an SSA form IR.
It converts a buildable program into the program with SSA representation.

SSA provides a lower level but more powerful APIs to check the references among the AST symbols.
It tries to check types and references the symbols together which allows to do semantic analysis.
By contrast, golang AST provides the parsed AST representations without any type check and advanced optimizations
such as dead code elimination.

The motivation is to provides a detailed guides for the people who
want to utilize golang SSA to analyze golang source code for their own semantics. Using SSA, it's also
possible to detect every function call, know the branches, and so on for a given program.

<!-- state the precondition when reading this guide -->


## SSA Program and Packages

SSA builds the golang source code like the compiler without machine code generation,
so we could use SSA to process a file, a package, a module or a whole program.
For example, we can use SSA program to stand for a runnable project which has a `main` function.

The SSA program is consisted by several SSA packages, while each package contains many top-level statements
such as functions, variables, imports, types and constants. The function body and variables contain
more statements/expressions such as function call, control flow(looping, branching, switching) and so forth.


As Go manages the code under the package level, to create a SSA program we need to load all the packages
needed by the program and then parsing them together at once.

The function `Load` allows to load all packages under a path, and we can use `ssautil.Packages`
to process packages to the SSA representation. During processing, SSA tries to load all dependencies
of the provided packages to form a completed SSA program and SSA packages.

```go
cfg := &packages.Config{
    Mode: packages.LoadAllSyntax | packages.NeedExportsFile,
}
pkgs, err := packages.Load(cfg, "golang.org/x/tools/mocked")
if err != nil {
    panic(-1)
}
program, spkg := ssautil.Packages(pkgs, ssa.SanityCheckFunctions)
```

The `packages.Load` resolves the types
and does the syntax check at the package level. Moreover, it will check the correctness of the
imported function(existence, return type, parameter types and so on). Error is reported if the imported
package cannot be found.

Loading a SSA program won't emit all SSA related information, and a further build is needed to emit
the function bodies.

### SSA Package

We could say a SSA program is consisted by multiple SSA packages, and we usually analyze the program
at the package level. Hence it's helpful to learn about some fields of a `ssa.Package` structure.

#### Package Field `Members`

Field `Members` stands for all named members of a Go package, and  the top-level statements.


The named member includes all top-level named types, constants, functions and variables.
It holds SSA structures like `NamedConst`, `*Global`, `*Function`, and `*Type`.
Anonymous structures and functions won't be treated as a named member.

Usually, field `Members` is the entry for us to analyze the whole program. For example, you may find the entry
(main function in main package) of your program to start.

Members are already mentioned previously, they're the top-level structures(todo: this is not type structure, but
a general concept to refer the variable, function, type, const) of a package.
Members is actually a facade for type
All of them are package top-level structures.

#### Package Field `Program`

Each package contains the pointer to its belonged program, and the same package loaded in different programs
are not equal. By this way, you can find another package via the program when you import and use the symbols externally.

#### Package Field `Pkg`

Pkg is the original representation `go/types.Package` before SSA package, which is more primitive
from AST.

## Value, Instruction and Node in SSA

- Value: an expression that yields a value, namely the expression in golang, includes all expressions
  mentioned in the golang specification.
- Instruction: a statement that consumes values and performs computation. this is clearly because the
  statement calculates the expressions and do some flows on the data. The expression is the computation and
  some operations consume the value of expression.

- Node: either Value of Instruction, you could treat it as a sum-type to represent them both, to utilize them in value graph

Some statements don't yield value, but some yield, namely the StatDecl in golang specification, which not only
is a statement but also an expression, which contains the operation and yields the value. For example,
the `Alloc` contains the operation to access memory, and yields an address for further usages.
(todo): the 'operation' is a bit ambiguous, try to find a more detailed explanation.

When analyzing SSA representations, you should prevent to use name-based operations for reasons of efficiency and
implementation easiness.

> The program representation constructed by this package is fully resolved internally

<!-- todo: this part hasn't been finished, need to refine further -->
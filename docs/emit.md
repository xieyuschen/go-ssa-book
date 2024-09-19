

Function body emits its instructions into ssa instructions associated by the block.

## buildPackageInit

The synthesized generated function to build.

`BareInits` to decide whether call its dependencies `init` function recursively.

The `init` function might be called by another `init`,
so a guard is necessary. That's `init$guard`.

```
if init$guard
    true: done // skip to run init
    false: init$start // run the init logic
```

`init$guard` is loaded to true at the end of block `init.start`.

`Imports` init methods are called in source order.

After all deps' init functions are called, call `init` functions in each file of current package.

Emit returning to go back. 

Finish the body, which re-arranges the function body for some reason.

-	optimizeBlocks(f)
-	buildReferrers(f)
-	buildDomTree(f)

## createBound

The bound function delegates to a concrete or interface method denoted by obj.
It converts a method call's signature while using free variable to refer the receiver.
```go
	fn.FreeVars = []*FreeVar{{name: "recv", typ: recvType(obj), parent: fn}} // (cyclic)
```

<!-- todo: think about the purpose -->

Then it tries to build the bound function, concrete/interface method call is treated differently.
`Method` is designed for tackling interface call.

For interface, the receiver isn't confirmed yet(not put as the first place of the args) for now.
The `CallCommon` is emitted as an instruction, the actual code will be put later(todo, whether it's true?).

Handle the return value, which is a second-class tuple and it's handled by a return.

## buildWrapper

Based on the selection, the create wrapper need to handle the recv.

```go
type Person struct {
	Name string
}

func (p Person) Greet() {
	fmt.Printf("Hello, my name is %s\n", p.Name)
}

func main() {
	greetFunc := Person.Greet // MethodExpr: referencing method 'Greet' as an expression
	p := Person{Name: "Charlie"}
	greetFunc(p)
}
```
It will treat this kind of usage as `thunk`, the others are treated as wrapper.

During building wrapper, it spills the receiver to the stack. Then creates the params.

(selection): explicit(final) selection and implicit(previous) selections.

New logic if the receiver is a pointer(todo: why)?

todo: wrapper is quite complex, come back later.

## buildFromSyntax

Syntax refers the case that the function comes from the source code according to the parsed AST.

Create receiver, parameters and results.

Then it creates the defer stack so a new variable `defer$stack` is created,
the call is stored inside 

It creates a call with `vDeferStack` and emit this function call, and
the local `defer$stack` variable is spilled.(a function call and a local variable).

Then, it emits the function body. 

Then, it runs defers and return the function, and finish the function body.

## buildParamsOnly

Builds params by function signature, doesn't build function body. 
Sometimes it doesn't have a function body(from asm or C). It doesn't spill
the params as well unlike the buildFromSyntax.






## buildYieldFunc

Compiler helps to create the range-over-func yield function, and it's built
by this function.

Adding params but not spill, no signature(generated so there is presumption).

Create `yield-continue` block as start, and loop all labeled block to refer the yield-continue.

Here the targets help to decide the next block for each case.

```go
type targets struct {
	tail         *targets // rest of stack
	_break       *BasicBlock
	_continue    *BasicBlock
	_fallthrough *BasicBlock
}
```

save the current block and change it to ycont, 
set the `jump` field to mark it's a range-over-func.
(todo: it's used to shared state in each stage, is it?)

create `yield-loop` and `yield-invalid` blocks, then emit a if to compare `jump`
and re-direct to one of them. Then change the current to invalid block and emit
a panic instruction there.

switch to yield-loop to emit code and store a `busy` inside jump.

Get key and value type and create identVar for them.

Then `addr` operation for the `ast.RangeStmt` key and value, then load the value inside it.
This should happen during `for k,v:=range iter` part.

Then, it starts to stmt the function body of the range loop.


## UNKNOWN

1. finishBody
2. why we need to create bound, and free variable of receiver
3. why builder wrapper, why spill?
4. how yield function works? continue please






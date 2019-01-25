# Calculator

This challenge will be creating a command line calculator. There are multiple
parts with increasing difficulty.


## Part 1 - Polish notation

Those familiar with functional programming might already understand this. Polish
notation (in contrast with traditional infix notation) is a way of writing
mathematical expressions which does not require the reader to think of
operator precedence. In this notation, operators are written one the left
and operands to the right. For example:
```
1 + 2
```
would be written as:
```
+ 1 2
```

The operands are recursively evaluated, with the inner most expressions
being evaluated first:
```
1 + 2 - 3
```
would be:
```
- + 1 2 3
```

We start with the `-`, it is an operator, so the operands need to be evaluated.
The next symbol `+` is also an operator and its operands need to be evaluated.
The next symbol is `1` which is a value, we now have the first operand of `+`.
The next symbol is `2` which is a value, we now have the second, and last,
 operand of `+`. We evaluate, and get `3`, *returning* it as the first operand
to the `-`. The next symbol is `3` which is a value, we now have the second, and
last, operand of the `-`. Evaluating the operation gives the result `0`. This
is the basic algorithm for evaluating Polish notation.

Note that the after inner operation is evaluated, the outer operation should start
reading symbols where the inner operation left of. The following snippet, showcases
this, by use of cursors.
```go
// https://play.golang.org/p/oe5TdJojZ8M
func ParseStr(input string, cursor *int) (value int, err error) {
	if num, n, ok := startsWithNumber(input[*cursor:]); ok {
		*cursor += n
		return num, nil
	}

	op := input[*cursor]
	*cursor += 2 // move from operator, move from space

	left, err := ParseStr(input, cursor)
	if err != nil {
		return 0, err
	}

	*cursor += 1 // space between operands
	right, err := ParseStr(input, cursor)
	if err != nil {
		return 0, err
	}
	value, err = doOp(op, left, right)

	return value, nil
}
```

This works but, it has a downfall, it assumes all opertors have two operands.
Try implementing an algorithm to handle the factorial operator `!` which
only has one argumnet. *hint:* `switch` on the operator, it can make adding
new oprators easier.

## Part 2 - abstract syntax tree

The polish notation and it's recursive properties also facilitates the creation
of an abstract syntax tree (AST). This is a data structure which descibes a
program its operations. For our AST we should first define a concrete value:

```go
type ValueNode int
```

Next we'll define an expression, this will ecompass anything that resolves to a
concrete value. For example `* 1 3` is not a concrete value, but can be
evaluated.

```go
type ExprNode interface {
	ExprNode()
}
```
This interface does nothing but enforce typing rules. We could have
a method `Eval` that evalutaes the expression but, that'll make things too
easy ;) We'll also be adding to this interface later on.

We need to now implement this in value node.
```go
func (op *ValueNode) ExprNode() {}
```

With these types, we can now define an operation and its methods
```go
type OpNode struct {
	Op byte
	Operands []Expr
}

func (op *OpNode) ExprNode() {}
```

Now, change the `ParseStr` function to return an ExprNode.

## Part 3 - virtual machine

Once we have a tree of our program, we need to somehow run it. There's several
options here. For a small language such as ours and given the capabilities
of `Go` we can write methods on `ExprNode` to evaluate itself as noted before.
The typical approach to this in interpreted languages is to build a virtual machine
(VM). Simply put a virtual machine is the machinery that executes the primitive
operations of a language. In our langugage this is simple arithmatic such as
adding, subtracting, etc. In other languges this can get as abstract as creating
classes, memory management, and os iteraction.

Our VM will consist of a stack for results and two registers
for executing operations. One register for the op code and another for the argument.
The code might be easier to understand:

```go
// https://play.golang.org/p/IC_DHTxTGyp
type VM struct {
	Result []int
}

func (vm *VM) Exec(opcode int, arg interface{}) error {
	switch opcode {
	case OpAdd:
		argi, ok := arg.(int)
		if !ok {
			return fmt.Errorf("bad arg %v", arg)
		}

		vm.Result[len(vm.Result)-1] += argi
		return nil
	case OpPush:
		argi, ok := arg.(int)
		if !ok {
			return fmt.Errorf("bad arg %v", arg)
		}
		vm.Result = append(vm.Result, argi)
		return nil
	default:
		return fmt.Errorf("bad opcode %d", opcode)
	}
}
```

Note that here we also began to create the defining bytecode. By this, I mean
that an instruction is defined by an int and a generic value as an argument.
We also came up with an opcode for the addition operation.

## Part 4 - generating bytecode from the ast

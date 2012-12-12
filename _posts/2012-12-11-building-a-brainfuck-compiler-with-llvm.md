---
layout: post
title: "Building a Brainfuck compiler with LLVM"
date: 2012-12-11 21:09
comments: true
---

I recently decided to explore the [LLVM][llvm] API, so I needed to settle on a
simple project I could quickly accomplish with it. I settled on building a
[Brainfuck][wiki-brainfuck] compiler. Readers should have moderate familiarity
with programming languages and the principles of compilers, but no knowledge of
Brainfuck or LLVM in particular are required.

This will be posted in several parts. My source code for this project is
available on [GitHub](https://github.com/jeremyroman/llvm-brainfuck) if you
prefer to just skip to the end. If you want a more thorough introduction to
using LLVM, I highly recommend the [Kaleidoscope tutorial][llvm-kaleidoscope],
which goes through building a lexer, parser and compiler for a more C-like
language. I referred to it more than once while building this.

First, a brief overview of Brainfuck is in order. Brainfuck is a ridiculously
minimal programming language, which is nonetheless Turing-complete. In
Brainfuck, there 8 instructions, each one character long. All other characters
in the program source are ignored. The program has a data array consisting of
30,000 cells (typically one byte wide) and a pointer into this array.
Initially, all cells are zero and the pointer points to the first element of
the array. The instructions are:

Symbol | Effect
------ | ---------------------------
`>`    | Move the pointer forward by 1.
`<`    | Move the pointer backward by 1.
`+`    | Increment the cell at the current pointer.
`-`    | Decrement the cell at the current pointer.
`.`    | Output the character at the current cell.
`,`    | Read a character into the current cell (for our purposes, EOF produces 0).
`[`    | If the current cell is zero, skips to the matching `]` later on.
`]`    | If the current cell is non-zero, skips to the matching `[` earlier on.

The bracket notation forms the equivalent of a `while` loop (where the only
valid condition is whether the current cell is zero or not). Constants can be
constructed only by a series of calculations on existing cells. The [Wikipedia
article][wiki-brainfuck] contains a few example programs.

Now to get down to work building this with LLVM. I used LLVM 2.7 (from Debian
squeeze); some details may vary slightly between releases.

Setting Up
----------

We start by writing some boilerplate code to get started on `main.cpp`, where we
will implement our compiler.

{% highlight_git cpp https://gist.github.com/4264434.git main.cpp %}

First, we create a module which will contain the code that is generated for the
Brainfuck program, and create an IR builder. LLVM IR (Intermediate Runtime) is
the language that LLVM uses internally. We will be generating IR, which we can
later compile down to platform-specific assembly code.

Using the context, we retrieve the data type (8-bit integer) that we intend to
use for our cells, and using that, obtain the type of an array of 30,000 of
them. This allows us to define a global variable in the main module. We declare
that it should be of the array type we found earlier, have internal linkage
(that is, we don't need to be able to reach it from other objects we may link
with it later on), be initialized to null (i.e. all zeroes), and be called
`data`. Finally, we get the initial value of the pointer by emitting a
`getelementptr` instruction.

This instruction bears a little explanation (though there's a lot more depth
available [elsewhere][llvm-gep]). The `GlobalVariable` we have actually
represents a *pointer* to the array, which itself contains 30,000 integers.
In order to get a pointer to the first cell of the array, we need to provide two
indices (both zero). The first changes the value of type "pointer to array of
30,000 integers" to a value of type "array of 30,000 integers", by asking for
the first element of the pointer (i.e. the thing it points to). The second goes
deeper, giving us a value which refers to the first cell of that array. We are
given a *pointer* to the element we requested.

Note that no memory access is actually involved in this calculation (in fact, it
will not compile down to an actual assembly instruction). Rather, this LLVM IR
instruction simply represents the computation which produces a pointer of the
correct type at the beginning of the array.

Then we define the main function of our program, which we call `brainfuck_main`.
It will later be linked with a shim that invokes this function. The function has
the following signature:

{% highlight cpp %}
extern void brainfuck_main();
{% endhighlight %}

So we first have to construct the type which defines functions with this
signature, and then declare the function itself (with external linkage, as we
will need to be able to invoke this procedure from another object) in the main
module.

We also create a basic block (which represents a block of instructions with no
control flow; i.e. they always execute sequentially from top to bottom). Later
on we will create additional blocks when we implement the looping construct.
We also set the insertion point in this block, so that when we generate
instructions they are inserted in this block.

To start with, we simply emit a "return void" instruction (thus finishing the
block) and print out the module IR to standard output. A simple Makefile
suffices to build this.

{% highlight_git make https://gist.github.com/4264434.git Makefile %}

If we build and run this, we get the following output (a human-readable LLVM IR
representation of our program):

{% highlight llvm %}
; ModuleID = 'brainfuck program'

@data = internal global [30000 x i8] zeroinitializer ; <[30000 x i8]*> [#uses=1]

define void @brainfuck_main() {
entry:
  ret void
}
{% endhighlight %}

We see that this describes what we've written so far, in a nice readable format.
There's a global variable called `data` which is of type `[30000 x i8]`, or
"array of 30,000 8-bit integers", and the comment to the right tells us that the
pointer to it was used once. We can also see the function we defined, short
though it may be, and the block marked with the label `entry`.

The code for this section is available as a
[gist](https://gist.github.com/4264434).

Simple Instructions
-------------------

Now we would like to start compiling some actual instructions. We'll start with
the easy four: `>`, `<`, `+` and `-`.

Between the main function definition and the return statement, we add a code
generation loop. Because Brainfuck is such a simple language `getchar` suffices
as a lexer.

First, we'll support moving the pointer around.

{% highlight cpp %}
// Code generation.
int c;
while ((c = getchar()) != EOF) {
  switch (c) {
    case '>':
      DataPtr = Builder.CreateConstGEP1_32(DataPtr, 1, PTR_NAME);
      break;
    case '<':
      DataPtr = Builder.CreateConstGEP1_32(DataPtr, -1, PTR_NAME);
      break;
  }
}
{% endhighlight %}

This still only emits `getelementptr` instructions; this time with only one
index. If we wish to move a pointer forward, we need to get a pointer to element
1 of the pointer (this is analogous to `&a[1]` or `a+1`, if `a` is a pointer).
Similarly, if we wish to move the pointer backward, we use index -1. We
overwrite the `DataPtr` variable, so that further uses of it will refer to the
shifted version. Again, this does not actually cause memory access: that happens
when we use an explicit `load` or `store` instruction.

We can build and run this, but we still don't generate any code. So we will add
support for the `+` and `-` instructions as well. First, we'll need to have a
constant 1 of the appropriate type to add and subtract from cells.

{% highlight cpp %}
Constant *One = ConstantInt::get(CellType, 1);
{% endhighlight %}

Then we add a temporary variable `Value *Value;` to the code generation section
to make these instructions, which take a few steps, a little easier to read.
With that in place, we can simply add two cases to the switch:

{% highlight cpp %}
case '+':
  Value = Builder.CreateLoad(DataPtr);
  Value = Builder.CreateAdd(Value, One);
  Builder.CreateStore(Value, DataPtr);
  break;
case '-':
  Value = Builder.CreateLoad(DataPtr);
  Value = Builder.CreateSub(Value, One);
  Builder.CreateStore(Value, DataPtr);
  break;
{% endhighlight %}

So in order to increment the current cell, we need to:

1. Load the value of the cell from the current data pointer.
2. Add a constant 1 to that result.
3. Store the sum back into the cell at the data pointer.

The IR builder takes care of emitting the appropriate LLVM instructions to do
this.

We rebuild this, and now we can begin to see the fruits of our labour. Let's
consider the simple Brainfuck program `>>+++`, which adds 3 to the value of data
cell 2. Feeding this into our improved compiler produces this output:

{% highlight llvm %}
; ModuleID = 'brainfuck program'

@data = internal global [30000 x i8] zeroinitializer ; <[30000 x i8]*> [#uses=3]

define void @brainfuck_main() {
entry:
  %0 = load i8* getelementptr ([30000 x i8]* @data, i32 0, i32 2) ; <i8> [#uses=1]
  %1 = add i8 %0, 1                               ; <i8> [#uses=1]
  store i8 %1, i8* getelementptr ([30000 x i8]* @data, i32 0, i32 2)
  %2 = load i8* getelementptr ([30000 x i8]* @data, i32 0, i32 2) ; <i8> [#uses=1]
  %3 = add i8 %2, 1                               ; <i8> [#uses=1]
  store i8 %3, i8* getelementptr ([30000 x i8]* @data, i32 0, i32 2)
  %4 = load i8* getelementptr ([30000 x i8]* @data, i32 0, i32 2) ; <i8> [#uses=1]
  %5 = add i8 %4, 1                               ; <i8> [#uses=1]
  store i8 %5, i8* getelementptr ([30000 x i8]* @data, i32 0, i32 2)
  ret void
}
{% endhighlight %}

Note that the `getelementptr` instructions are automatically coalesced by LLVM,
and are simply used as addresses for the load and store instructions. LLVM
creates a number of "temporaries" marked with percent signs. These are static
single-assignment (SSA) variables, which can be assigned only once.

In this output, we can see the instructions to load cell 2, add 1 to it, and
store the result back three times. This is exactly what the Brainfuck program
we provided should do.

The code for this section is available as a
[gist](https://gist.github.com/4269878).

Building LLVM IR
----------------

Now that we have some valid LLVM IR generated, how do we build it? There are a
few tools at our disposal.

`llvm-as`, the LLVM assembler, takes our human-readable IR and generates LLVM
bitcode, a compact binary representation of the same code. Think of this as
analogous to the relationship between assembler and machine code when using a
standard assembler.

`opt`, the LLVM optimizer, takes LLVM bitcode and applies optimization passes of
your choice to produce more efficient bitcode.

`llc`, the LLVM compiler, compiles LLVM bitcode to assembler code for a
particular platform.

`clang`, the C compiler part of the LLVM project, can be used to assemble and
link these assembler files to produce an executable binary.

We can actually use some of these features of LLVM from directly within our
program, but for convenience we will use the external tools instead.

Let's see what the optimizer can do with our program. Try feeding the same
Brainfuck program into this command:

    ./llvm-brainfuck | llvm-as | opt -instcombine -print-module >/dev/null

This assembles the IR into bitcode, then performs an instruction-combining pass
that should simplify our program and prints the module in human-readable (IR)
form. It also produces optimized LLVM bitcode, which we discard for now.

We get this output:

{% highlight llvm %}
; ModuleID = '<stdin>'

@data = internal global [30000 x i8] zeroinitializer ; <[30000 x i8]*> [#uses=2]

define void @brainfuck_main() {
entry:
  %0 = load i8* getelementptr inbounds ([30000 x i8]* @data, i64 0, i64 2) ; <i8> [#uses=1]
  %1 = add i8 %0, 3                               ; <i8> [#uses=1]
  store i8 %1, i8* getelementptr inbounds ([30000 x i8]* @data, i64 0, i64 2)
  ret void
}
{% endhighlight %}

We can see that the optimizer has now combined the three separate increment
operations into a single operation which adds 3 to the cell. If you run `opt
--help`, you can see a full list of options that are possible. Many of them
don't really apply to our programs, so you won't see any difference.

Let's also take a look at the generated assembler code. We run:

    ./llvm-brainfuck | llvm-as | opt -instcombine | llc

There's a fair amount of boilerplate code here, but the actual program looks
like this on my platform (Linux on an x86_64 system).

{% highlight gas %}
     .globl     brainfuck_main
     .type      brainfuck_main,@function
brainfuck_main:                                             # @brainfuck_main
.Leh_func_begin1:
.LBB1_0:                                                    # %entry
     addb       $3, data+2(%rip)
     ret
     .size      brainfuck_main, .-brainfuck_main
.Leh_func_end1:
     .type      data,@object
     .bss
     .local     data
     .comm      data,30000,16                               # @data
{% endhighlight %}

We can see the `data` variable declared to have 30,000 elements. And we can see
the contents of our actual `brainfuck_main` function:

1. an `addb` instruction which adds a constant 3 to the byte offset by 2 from
the beginning of the data object (accessed relative to the instruction pointer)
2. a `ret` instruction which returns from the function

So the compiler was clever enough to combine the first three LLVM instructions
into a single x86_64 instruction which performs the load, addition and store.

OK, now we wish to actually make a working Linux binary out of this. We need to
provide a shim that will be invoked as the main function which calls
`brainfuck_main` appropriately. This small bit of C code (`shim.c`) will do.

{% highlight c %}
#include <stdio.h>

extern void brainfuck_main();

int main(int argc, char **argv) {
  brainfuck_main();
  return 0;
}
{% endhighlight %}

We modify our Makefile to build a static library out of this.

{% highlight_git make https://gist.github.com/4269856.git Makefile %}

After rebuilding, we can now build a Brainfuck program like this:

    ./llvm-brainfuck <example.bf | llvm-as | opt -instcombine | llc >example.s
    clang -o example example.s shim.a

It builds! It runs! But there's no input or output. So it's a little boring.

The code for this section is available as a
[gist](https://gist.github.com/4269856).

Input/Output
------------

We would like to support input and output from Brainfuck programs by supporting
the `.` and `,` instructions. We'll do this by providing two functions,
`brainfuck_put` and `brainfuck_get`, which implement the behaviour of these
instructions using the C standard library.

So we add the following simple code to our shim. The behaviour of the `,`
operator when end-of-file is reached varies between implementations. We choose
to return 0 in such cases, but other implementations may return -1 or leave the
cell unchanged (the latter would require a slightly different fucntion
signature).

{% highlight c %}
void brainfuck_put(char c) {
  putchar(c);
}

char brainfuck_get() {
  int c = getchar();
  return c >= 0 ? c : 0;
}
{% endhighlight %}

Now we need only use these in our compiler. After the constants, we declare the
functions we added to the shim (but don't implement them). By declaring them to
have external linkage, we are able to import them from the shim during linking.

{% highlight cpp %}
// Function prototypes for the shim.
FunctionType *PutFunctionType = FunctionType::get(
    Type::getVoidTy(Context) /* return type */,
    std::vector<const Type *>(1, CellType) /* argument types */,
    false /* var args */);
Function *PutFunction = Function::Create(PutFunctionType,
    Function::ExternalLinkage, "brainfuck_put", &MainModule);
FunctionType *GetFunctionType = FunctionType::get(
    CellType /* return type */,
    std::vector<const Type *>() /* argument types */,
    false /* var args */);
Function *GetFunction = Function::Create(GetFunctionType,
    Function::ExternalLinkage, "brainfuck_get", &MainModule);
{% endhighlight %}

With these in hand, we can add two more cases to the switch statement to
generate code for the Brainfuck instructions.

{% highlight cpp %}
case '.':
  Value = Builder.CreateLoad(DataPtr);
  Builder.CreateCall(PutFunction, Value);
  break;
case ',':
  Value = Builder.CreateCall(GetFunction);
  Builder.CreateStore(Value, DataPtr);
  break;
{% endhighlight %}

These will emit LLVM `call` instructions, which invoke the functions in the
shim. With these changes, you can now write a Brainfuck program which does I/O.
For example, you might try running the program `,.+.+.`, which reads a
character and outputs it back, along with the two following it. Given "a", it
prints "abc".

Input and output, done. Six instructions down, two to go.

The code for this section is available as a
[gist](https://gist.github.com/4270505).

Flow Control
------------

We can now go on to implement Brainfuck's flow control construct. It's important
that the brackets be matched properly, so we will abort if we detect a problem
there. We will track the state of each loop by creating a stack of data about
them.

But first, we require a little more knowledge about how flow control works in
LLVM. Recall that each basic block cannot contain flow control (except its last
instruction, which can be a branch instruction, a return instruction, or
something similar which specifies what happens next). So each loop has at a
minimum:

1. The entry block. This happens before the loop, and checks the condition to
determine whether the loop is entered or skipped.
2. The body. There may be more than one basic block in the body, but we will
explicitly create one for the beginning of the body. And when we reach the end
of the loop, we will need to add a conditional branch to the basic block at the
end of the loop to determine whether the loop exits or runs again.
3. The exit block. This happens after the loop is done, and is where code
following the loop is inserted.

Now, since LLVM IR is in static single-assignment form, no temporary variable
can be assigned more than once. How do we deal with loop counters, and other
variables that may be assigned different variables depending on control flow?

We need to create what are called &phi; nodes (phi nodes). These represent the
possibility of a variable value coming from any of a number of places. In this
case, we need two phi nodes:

1. At the beginning of the loop, the value of the data pointer could come from
either the code before the loop (via the entry block), or from a previous
iteration of the loop (via the final block in the loop body). So we need to
create a phi node to represent the value of the data pointer at this point.
2. After the end of the loop, the value of the data pointer could come from
either the code before the loop (if the loop body was skipped altogether), or
from the loop body (if it did execute). So we need to create a phi node for the
data pointer here as well.

For each phi node we create, we need to specify for each basic block which could
lead to the one we're defining the phi node in, what temporary variable
corresponds to that possible value.

So at the beginning of each loop, we store some data we can use to tie this all
together properly when the loop ends. We add this structure definition to our
program:

{% highlight cpp %}
struct Loop {
  BasicBlock *Entry, *Body, *Exit;
  PHINode *DataPtrBody, *DataPtrExit;
};
{% endhighlight %}

We also include `<stack>` to provide the stack implementation.

And in the code generation section, we create a local variable of this type, and
a stack in which we will store one such entry for each loop we enter. As we exit
loops, we pop these records off the stack.

{% highlight cpp %}
std::stack<Loop> Loops;
Loop ThisLoop;
{% endhighlight %}

Let's look first at what we do when we enter a loop:

{% highlight cpp %}
case '[':
  // Prepare data for the stack.
  ThisLoop.Entry = Builder.GetInsertBlock();
  ThisLoop.Body = BasicBlock::Create(Context, "loop", MainFunction);
  ThisLoop.Exit = BasicBlock::Create(Context, "exit", MainFunction);

  // Emit the beginning conditional branch.
  Value = Builder.CreateLoad(DataPtr);
  Value = Builder.CreateIsNotNull(Value);
  Builder.CreateCondBr(Value, ThisLoop.Body, ThisLoop.Exit);

  // Define the pointer after the loop.
  Builder.SetInsertPoint(ThisLoop.Exit);
  ThisLoop.DataPtrExit = Builder.CreatePHI(DataPtr->getType(), PTR_NAME);
  ThisLoop.DataPtrExit->addIncoming(DataPtr, ThisLoop.Entry);

  // Define the pointer within the loop.
  Builder.SetInsertPoint(ThisLoop.Body);
  ThisLoop.DataPtrBody = Builder.CreatePHI(DataPtr->getType(), PTR_NAME);
  ThisLoop.DataPtrBody->addIncoming(DataPtr, ThisLoop.Entry);

  // Continue generating code within the loop.
  DataPtr = ThisLoop.DataPtrBody;
  Loops.push(ThisLoop);
  break;
{% endhighlight %}

We create two new basic blocks, one for the loop body, and one for the loop
exit. Then we emit (in the entry block, since we haven't yet moved the builder's
insertion point) instructions to load the current cell, and branch into the body
if the cell is non-zero, and otherwise into the exit block.

We then move to the exit block and emit a phi node for the data pointer, and add
one incoming edge to it, noting that if execution arrives from the entry block,
then its value is defined by `DataPtr` (that is, whatever we computed the data
pointer to be before we hit the `[` instruction). We will add the second
incoming edge later on.

We then move the insertion point to the loop body, and emit a similar phi node.
We set the current definition of the dadta pointer to be this new phi node, push
the record we created onto the stack, and continue on to the next instruction.
During the body of the loop, we emit instructions into the loop body, rather
than the block we originally had.

When we reach the corresponding end of the loop, we just need to tidy up and
move the insertion point to the exit block.

{% highlight cpp %}
case ']':
  // Pop data off the stack.
  if (Loops.empty()) {
    fputs("] requires matching [\n", stderr);
    abort();
  }
  ThisLoop = Loops.top();
  Loops.pop();

  // Finish off the phi nodes.
  ThisLoop.DataPtrBody->addIncoming(DataPtr, Builder.GetInsertBlock());
  ThisLoop.DataPtrExit->addIncoming(DataPtr, Builder.GetInsertBlock());

  // Emit the ending conditional branch.
  Value = Builder.CreateLoad(DataPtr);
  Value = Builder.CreateIsNotNull(Value);
  Builder.CreateCondBr(Value, ThisLoop.Body, ThisLoop.Exit);

  // Move insertion after the loop.
  ThisLoop.Exit->moveAfter(Builder.GetInsertBlock());
  DataPtr = ThisLoop.DataPtrExit;
  Builder.SetInsertPoint(ThisLoop.Exit);
  break;
{% endhighlight %}

If we couldn't pop an entry off the stack, that means that we have a `]`
instruction which is not matched by a preceding `[` instruction, so we abort.

Otherwise, we finish off the phi nodes using the current definition of the data
pointer and the current block (which is the block through which the loop body
exits), and emit another conditional branch which determines whether we need to
cycle back again.

With that done, we move the exit block after all blocks in the loop body. This
isn't strictly necessary, but puts the code in a more logical (and perhaps more
cache-friendly) order. We then change the data pointer definition to the phi
node for the end of the loop, move the insertion point to the exit block, and
continue accepting instructions.

As one last check, after we have processed all instructions (i.e. after
receiving end-of-file), we ensure that all loops that were opened were closed by
ensuring that the stack is empty.

{% highlight cpp %}
// Ensure all loops have been closed.
if (!Loops.empty()) {
  fputs("[ requires matching ]\n", stderr);
  abort();
}
{% endhighlight %}

With that, we now have a completely functional Brainfuck compiler!

The code for this section is available as a
[gist](https://gist.github.com/4270777).

Some possible next steps, if you wanted to pursue them:

* Move the optimization logic inside the compiler by adding optimization passes.
* Emit LLVM bitcode directly instead of LLVM IR (or provide an option allowing
  the user to choose).
* Use an `ExecutionEngine` to run the Brainfuck code in the process itself,
  paired with a read-evaluate loop. This will turn the Brainfuck compiler into
  an interactive just-in-time compiler.
* Detect some common Brainfuck idioms, and compile them to more concise
  representations.

[llvm]: http://llvm.org/
[llvm-gep]: http://llvm.org/docs/GetElementPtr.html
[llvm-kaleidoscope]: http://llvm.org/docs/tutorial/
[wiki-brainfuck]: http://en.wikipedia.org/wiki/Brainfuck

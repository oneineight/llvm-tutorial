# 7. Mutable Variables


## 7.1 Chapter 7 Introduction

Welcome to Chapter 7 of the “Implementing a language with LLVM” tutorial. In chapters 1 through 6, we’ve built a very respectable, albeit simple, functional programming language. In our journey, we learned some parsing techniques, how to build and represent an AST, how to build LLVM IR, and how to optimize the resultant code as well as JIT compile it.

While Kaleidoscope is interesting as a functional language, the fact that it is functional makes it “too easy” to generate LLVM IR for it. In particular, a functional language makes it very easy to build LLVM IR directly in SSA form. Since LLVM requires that the input code be in SSA form, this is a very nice property and it is often unclear to newcomers how to generate code for an imperative language with mutable variables.

The short (and happy) summary of this chapter is that there is no need for your front-end to build SSA form: LLVM provides highly tuned and well tested support for this, though the way it works is a bit unexpected for some.


## 7.2 Why is this a hard problem?

To understand why mutable variables cause complexities in SSA construction, consider this extremely simple C example:

```c++
int G, H;
int test(_Bool Condition) {
  int X;
  if (Condition)
    X = G;
  else
    X = H;
  return X;
}
```

In this case, we have the variable “X”, whose value depends on the path executed in the program. Because there are two different possible values for X before the return instruction, a PHI node is inserted to merge the two values. The LLVM IR that we want for this example looks like this:

```c++
@G = weak global i32 0   ; type of @G is i32*
@H = weak global i32 0   ; type of @H is i32*

define i32 @test(i1 %Condition) {
entry:
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32* @G
  br label %cond_next

cond_false:
  %X.1 = load i32* @H
  br label %cond_next

cond_next:
  %X.2 = phi i32 [ %X.1, %cond_false ], [ %X.0, %cond_true ]
  ret i32 %X.2
}
```

In this example, the loads from the G and H global variables are explicit in the LLVM IR, and they live in the then/else branches of the if statement (cond_true/cond_false). In order to merge the incoming values, the `X.2` phi node in the `cond_next` block selects the right value to use based on where control flow is coming from: if control flow comes from the `cond_false` block, `X.2` gets the value of `X.1`. Alternatively, if control flow comes from cond_true, it gets the value of `X.0`. The intent of this chapter is not to explain the details of SSA form. For more information, see one of the many online references.

The question for this article is “who places the phi nodes when lowering assignments to mutable variables?”. The issue here is that LLVM requires that its IR be in SSA form: there is no “non-ssa” mode for it. However, SSA construction requires non-trivial algorithms and data structures, so it is inconvenient and wasteful for every front-end to have to reproduce this logic.


## 7.3 Memory in LLVM

The ‘trick’ here is that while LLVM does require all register values to be in SSA form, it does not require (or permit) memory objects to be in SSA form. In the example above, note that the loads from G and H are direct accesses to G and H: they are not renamed or versioned. This differs from some other compiler systems, which do try to version memory objects. In LLVM, instead of encoding dataflow analysis of memory into the LLVM IR, it is handled with Analysis Passes which are computed on demand.

With this in mind, the high-level idea is that we want to make a stack variable (which lives in memory, because it is on the stack) for each mutable object in a function. To take advantage of this trick, we need to talk about how LLVM represents stack variables.

In LLVM, all memory accesses are explicit with load/store instructions, and it is carefully designed not to have (or need) an “address-of” operator. Notice how the type of the @G/@H global variables is actually “i32*” even though the variable is defined as `i32`. What this means is that @G defines space for an i32 in the global data area, but its name actually refers to the address for that space. Stack variables work the same way, except that instead of being declared with global variable definitions, they are declared with the LLVM alloca instruction:

```c++
define i32 @example() {
entry:
  %X = alloca i32           ; type of %X is i32*.
  ...
  %tmp = load i32* %X       ; load the stack value %X from the stack.
  %tmp2 = add i32 %tmp, 1   ; increment it
  store i32 %tmp2, i32* %X  ; store it back
  ...
```

This code shows an example of how you can declare and manipulate a stack variable in the LLVM IR. Stack memory allocated with the alloca instruction is fully general: you can pass the address of the stack slot to functions, you can store it in other variables, etc. In our example above, we could rewrite the example to use the alloca technique to avoid using a PHI node:

```c++
@G = weak global i32 0   ; type of @G is i32*
@H = weak global i32 0   ; type of @H is i32*

define i32 @test(i1 %Condition) {
entry:
  %X = alloca i32           ; type of %X is i32*.
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32* @G
  store i32 %X.0, i32* %X   ; Update X
  br label %cond_next

cond_false:
  %X.1 = load i32* @H
  store i32 %X.1, i32* %X   ; Update X
  br label %cond_next

cond_next:
  %X.2 = load i32* %X       ; Read X
  ret i32 %X.2
}
```

With this, we have discovered a way to handle arbitrary mutable variables without the need to create Phi nodes at all:

Each mutable variable becomes a stack allocation.
Each read of the variable becomes a load from the stack.
Each update of the variable becomes a store to the stack.
Taking the address of a variable just uses the stack address directly.
While this solution has solved our immediate problem, it introduced another one: we have now apparently introduced a lot of stack traffic for very simple and common operations, a major performance problem. Fortunately for us, the LLVM optimizer has a highly-tuned optimization pass named “mem2reg” that handles this case, promoting allocas like this into SSA registers, inserting Phi nodes as appropriate. If you run this example through the pass, for example, you’ll get:

```c++
$ llvm-as < example.ll | opt -mem2reg | llvm-dis
@G = weak global i32 0
@H = weak global i32 0

define i32 @test(i1 %Condition) {
entry:
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32* @G
  br label %cond_next

cond_false:
  %X.1 = load i32* @H
  br label %cond_next

cond_next:
  %X.01 = phi i32 [ %X.1, %cond_false ], [ %X.0, %cond_true ]
  ret i32 %X.01
}
```

The mem2reg pass implements the standard “iterated dominance frontier” algorithm for constructing SSA form and has a number of optimizations that speed up (very common) degenerate cases. The mem2reg optimization pass is the answer to dealing with mutable variables, and we highly recommend that you depend on it. Note that mem2reg only works on variables in certain circumstances:

mem2reg is alloca-driven: it looks for allocas and if it can handle them, it promotes them. It does not apply to global variables or heap allocations.
mem2reg only looks for alloca instructions in the entry block of the function. Being in the entry block guarantees that the alloca is only executed once, which makes analysis simpler.
mem2reg only promotes allocas whose uses are direct loads and stores. If the address of the stack object is passed to a function, or if any funny pointer arithmetic is involved, the alloca will not be promoted.
mem2reg only works on allocas of first class values (such as pointers, scalars and vectors), and only if the array size of the allocation is 1 (or missing in the .ll file). mem2reg is not capable of promoting structs or arrays to registers. Note that the “scalarrepl” pass is more powerful and can promote structs, “unions”, and arrays in many cases.
All of these properties are easy to satisfy for most imperative languages, and we’ll illustrate it below with Kaleidoscope. The final question you may be asking is: should I bother with this nonsense for my front-end? Wouldn’t it be better if I just did SSA construction directly, avoiding use of the mem2reg optimization pass? In short, we strongly recommend that you use this technique for building SSA form, unless there is an extremely good reason not to. Using this technique is:

Proven and well tested: clang uses this technique for local mutable variables. As such, the most common clients of LLVM are using this to handle a bulk of their variables. You can be sure that bugs are found fast and fixed early.
Extremely Fast: mem2reg has a number of special cases that make it fast in common cases as well as fully general. For example, it has fast-paths for variables that are only used in a single block, variables that only have one assignment point, good heuristics to avoid insertion of unneeded phi nodes, etc.
Needed for debug info generation: Debug information in LLVM relies on having the address of the variable exposed so that debug info can be attached to it. This technique dovetails very naturally with this style of debug info.
If nothing else, this makes it much easier to get your front-end up and running, and is very simple to implement. Lets extend Kaleidoscope with mutable variables now!


## 7.4 Mutable Variables in Kaleidoscope

Now that we know the sort of problem we want to tackle, lets see what this looks like in the context of our little Kaleidoscope language. We’re going to add two features:

The ability to mutate variables with the `=` operator.
The ability to define new variables.
While the first item is really what this is about, we only have variables for incoming arguments as well as for induction variables, and redefining those only goes so far :). Also, the ability to define new variables is a useful thing regardless of whether you will be mutating them. Here’s a motivating example that shows how we could use these:

```c++
# Define ':' for sequencing: as a low-precedence operator that ignores operands
# and just returns the RHS.
def binary : 1 (x y) y;

# Recursive fib, we could do this before.
def fib(x)
  if (x < 3) then
    1
  else
    fib(x-1)+fib(x-2);

# Iterative fib.
def fibi(x)
  var a = 1, b = 1, c in
  (for i = 3, i < x in
     c = a + b :
     a = b :
     b = c) :
  b;

# Call it.
fibi(10);
```

In order to mutate variables, we have to change our existing variables to use the “alloca trick”. Once we have that, we’ll add our new operator, then extend Kaleidoscope to support new variable definitions.


## 7.5 Adjusting Existing Variables for Mutation

The symbol table in Kaleidoscope is managed at code generation time by the `NamedValues` map. This map currently keeps track of the LLVM `Value*` that holds the double value for the named variable. In order to support mutation, we need to change this slightly, so that it NamedValues holds the memory location of the variable in question. Note that this change is a refactoring: it changes the structure of the code, but does not (by itself) change the behavior of the compiler. All of these changes are isolated in the Kaleidoscope code generator.

At this point in Kaleidoscope’s development, it only supports variables for two things: incoming arguments to functions and the induction variable of `for` loops. For consistency, we’ll allow mutation of these variables in addition to other user-defined variables. This means that these will both need memory locations.

To start our transformation of Kaleidoscope, we’ll change the NamedValues map so that it maps to `AllocaInst*` instead of `Value*`. Once we do this, the C++ compiler will tell us what parts of the code we need to update:

```c++
static std::map<std::string, AllocaInst*> NamedValues;
```

Also, since we will need to create these alloca’s, we’ll use a helper function that ensures that the allocas are created in the entry block of the function:

```c++
/// CreateEntryBlockAlloca - Create an alloca instruction in the entry block of
/// the function.  This is used for mutable variables etc.
static AllocaInst *CreateEntryBlockAlloca(Function *TheFunction,
                                          const std::string &VarName) {
  IRBuilder<> TmpB(&TheFunction->getEntryBlock(),
                 TheFunction->getEntryBlock().begin());
  return TmpB.CreateAlloca(Type::getDoubleTy(getGlobalContext()), 0,
                           VarName.c_str());
}
```

This funny looking code creates an IRBuilder object that is pointing at the first instruction (.begin()) of the entry block. It then creates an alloca with the expected name and returns it. Because all values in Kaleidoscope are doubles, there is no need to pass in a type to use.

With this in place, the first functionality change we want to make is to variable references. In our new scheme, variables live on the stack, so code generating a reference to them actually needs to produce a load from the stack slot:

```c++
Value *VariableExprAST::Codegen() {
  // Look this variable up in the function.
  Value *V = NamedValues[Name];
  if (V == 0) return ErrorV("Unknown variable name");

  // Load the value.
  return Builder.CreateLoad(V, Name.c_str());
}
```

As you can see, this is pretty straightforward. Now we need to update the things that define the variables to set up the alloca. We’ll start with ForExprAST::Codegen (see the full code listing for the unabridged code):

```c++
Function *TheFunction = Builder.GetInsertBlock()->getParent();

// Create an alloca for the variable in the entry block.
AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, VarName);

  // Emit the start code first, without 'variable' in scope.
Value *StartVal = Start->Codegen();
if (StartVal == 0) return 0;

// Store the value into the alloca.
Builder.CreateStore(StartVal, Alloca);
...

// Compute the end condition.
Value *EndCond = End->Codegen();
if (EndCond == 0) return EndCond;

// Reload, increment, and restore the alloca.  This handles the case where
// the body of the loop mutates the variable.
Value *CurVar = Builder.CreateLoad(Alloca);
Value *NextVar = Builder.CreateFAdd(CurVar, StepVal, "nextvar");
Builder.CreateStore(NextVar, Alloca);
...
```

This code is virtually identical to the code before we allowed mutable variables. The big difference is that we no longer have to construct a PHI node, and we use load/store to access the variable as needed.

To support mutable argument variables, we need to also make allocas for them. The code for this is also pretty simple:

```c++
/// CreateArgumentAllocas - Create an alloca for each argument and register the
/// argument in the symbol table so that references to it will succeed.
void PrototypeAST::CreateArgumentAllocas(Function *F) {
  Function::arg_iterator AI = F->arg_begin();
  for (unsigned Idx = 0, e = Args.size(); Idx != e; ++Idx, ++AI) {
    // Create an alloca for this variable.
    AllocaInst *Alloca = CreateEntryBlockAlloca(F, Args[Idx]);

    // Store the initial value into the alloca.
    Builder.CreateStore(AI, Alloca);

    // Add arguments to variable symbol table.
    NamedValues[Args[Idx]] = Alloca;
  }
}
```

For each argument, we make an alloca, store the input value to the function into the alloca, and register the alloca as the memory location for the argument. This method gets invoked by FunctionAST::Codegen right after it sets up the entry block for the function.

The final missing piece is adding the mem2reg pass, which allows us to get good codegen once again:

```c++
// Set up the optimizer pipeline.  Start with registering info about how the
// target lays out data structures.
OurFPM.add(new DataLayout(*TheExecutionEngine->getDataLayout()));
// Promote allocas to registers.
OurFPM.add(createPromoteMemoryToRegisterPass());
// Do simple "peephole" optimizations and bit-twiddling optzns.
OurFPM.add(createInstructionCombiningPass());
// Reassociate expressions.
OurFPM.add(createReassociatePass());
It is interesting to see what the code looks like before and after the mem2reg optimization runs. For example, this is the before/after code for our recursive fib function. Before the optimization:

define double @fib(double %x) {
entry:
  %x1 = alloca double
  store double %x, double* %x1
  %x2 = load double* %x1
  %cmptmp = fcmp ult double %x2, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:       ; preds = %entry
  br label %ifcont

else:       ; preds = %entry
  %x3 = load double* %x1
  %subtmp = fsub double %x3, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %x4 = load double* %x1
  %subtmp5 = fsub double %x4, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ 1.000000e+00, %then ], [ %addtmp, %else ]
  ret double %iftmp
}
```

Here there is only one variable (x, the input argument) but you can still see the extremely simple-minded code generation strategy we are using. In the entry block, an alloca is created, and the initial input value is stored into it. Each reference to the variable does a reload from the stack. Also, note that we didn’t modify the if/then/else expression, so it still inserts a PHI node. While we could make an alloca for it, it is actually easier to create a PHI node for it, so we still just make the PHI.

Here is the code after the mem2reg pass runs:

```c++
define double @fib(double %x) {
entry:
  %cmptmp = fcmp ult double %x, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:
  br label %ifcont

else:
  %subtmp = fsub double %x, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %subtmp5 = fsub double %x, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ 1.000000e+00, %then ], [ %addtmp, %else ]
  ret double %iftmp
}
```

This is a trivial case for mem2reg, since there are no redefinitions of the variable. The point of showing this is to calm your tension about inserting such blatent inefficiencies :).

After the rest of the optimizers run, we get:

```c++
define double @fib(double %x) {
entry:
  %cmptmp = fcmp ult double %x, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp ueq double %booltmp, 0.000000e+00
  br i1 %ifcond, label %else, label %ifcont

else:
  %subtmp = fsub double %x, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %subtmp5 = fsub double %x, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  ret double %addtmp

ifcont:
  ret double 1.000000e+00
}
```

Here we see that the simplifycfg pass decided to clone the return instruction into the end of the `else` block. This allowed it to eliminate some branches and the PHI node.

Now that all symbol table references are updated to use stack variables, we’ll add the assignment operator.


## 7.6 New Assignment Operator

With our current framework, adding a new assignment operator is really simple. We will parse it just like any other binary operator, but handle it internally (instead of allowing the user to define it). The first step is to set a precedence:

```c++
int main() {
  // Install standard binary operators.
  // 1 is lowest precedence.
  BinopPrecedence['='] = 2;
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
```

Now that the parser knows the precedence of the binary operator, it takes care of all the parsing and AST generation. We just need to implement codegen for the assignment operator. This looks like:

```c++
Value *BinaryExprAST::Codegen() {
  // Special case '=' because we don't want to emit the LHS as an expression.
  if (Op == '=') {
    // Assignment requires the LHS to be an identifier.
    VariableExprAST *LHSE = dynamic_cast<VariableExprAST*>(LHS);
    if (!LHSE)
      return ErrorV("destination of '=' must be a variable");
```

Unlike the rest of the binary operators, our assignment operator doesn’t follow the “emit LHS, emit RHS, do computation” model. As such, it is handled as a special case before the other binary operators are handled. The other strange thing is that it requires the LHS to be a variable. It is invalid to have “(x+1) = expr” - only things like “x = expr” are allowed.

```c++
  // Codegen the RHS.
  Value *Val = RHS->Codegen();
  if (Val == 0) return 0;

  // Look up the name.
  Value *Variable = NamedValues[LHSE->getName()];
  if (Variable == 0) return ErrorV("Unknown variable name");

  Builder.CreateStore(Val, Variable);
  return Val;
}
...
```

Once we have the variable, codegen’ing the assignment is straightforward: we emit the RHS of the assignment, create a store, and return the computed value. Returning a value allows for chained assignments like `X = (Y = Z)`.

Now that we have an assignment operator, we can mutate loop variables and arguments. For example, we can now run code like this:

```c++
# Function to print a double.
extern printd(x);

# Define ':' for sequencing: as a low-precedence operator that ignores operands
# and just returns the RHS.
def binary : 1 (x y) y;

def test(x)
  printd(x) :
  x = 4 :
  printd(x);

test(123);
```

When run, this example prints “123” and then “4”, showing that we did actually mutate the value! Okay, we have now officially implemented our goal: getting this to work requires SSA construction in the general case. However, to be really useful, we want the ability to define our own local variables, lets add this next!


## 7.7 User-defined Local Variables

Adding var/in is just like any other other extensions we made to Kaleidoscope: we extend the lexer, the parser, the AST and the code generator. The first step for adding our new ‘var/in’ construct is to extend the lexer. As before, this is pretty trivial, the code looks like this:

```c++
enum Token {
  ...
  // var definition
  tok_var = -13
...
}
...
static int gettok() {
...
    if (IdentifierStr == "in") return tok_in;
    if (IdentifierStr == "binary") return tok_binary;
    if (IdentifierStr == "unary") return tok_unary;
    if (IdentifierStr == "var") return tok_var;
    return tok_identifier;
...
```

The next step is to define the AST node that we will construct. For var/in, it looks like this:

```c++
/// VarExprAST - Expression class for var/in
class VarExprAST : public ExprAST {
  std::vector<std::pair<std::string, ExprAST*> > VarNames;
  ExprAST *Body;
public:
  VarExprAST(const std::vector<std::pair<std::string, ExprAST*> > &varnames,
             ExprAST *body)
  : VarNames(varnames), Body(body) {}

  virtual Value *Codegen();
};
```

var/in allows a list of names to be defined all at once, and each name can optionally have an initializer value. As such, we capture this information in the VarNames vector. Also, var/in has a body, this body is allowed to access the variables defined by the var/in.

With this in place, we can define the parser pieces. The first thing we do is add it as a primary expression:

```c++
/// primary
///   ::= identifierexpr
///   ::= numberexpr
///   ::= parenexpr
///   ::= ifexpr
///   ::= forexpr
///   ::= varexpr
static ExprAST *ParsePrimary() {
  switch (CurTok) {
  default: return Error("unknown token when expecting an expression");
  case tok_identifier: return ParseIdentifierExpr();
  case tok_number:     return ParseNumberExpr();
  case '(':            return ParseParenExpr();
  case tok_if:         return ParseIfExpr();
  case tok_for:        return ParseForExpr();
  case tok_var:        return ParseVarExpr();
  }
}
```

Next we define ParseVarExpr:

```c++
/// varexpr ::= 'var' identifier ('=' expression)?
//                    (',' identifier ('=' expression)?)* 'in' expression
static ExprAST *ParseVarExpr() {
  getNextToken();  // eat the var.

  std::vector<std::pair<std::string, ExprAST*> > VarNames;

  // At least one variable name is required.
  if (CurTok != tok_identifier)
    return Error("expected identifier after var");
The first part of this code parses the list of identifier/expr pairs into the local VarNames vector.

while (1) {
  std::string Name = IdentifierStr;
  getNextToken();  // eat identifier.

  // Read the optional initializer.
  ExprAST *Init = 0;
  if (CurTok == '=') {
    getNextToken(); // eat the '='.

    Init = ParseExpression();
    if (Init == 0) return 0;
  }

  VarNames.push_back(std::make_pair(Name, Init));

  // End of var list, exit loop.
  if (CurTok != ',') break;
  getNextToken(); // eat the ','.

  if (CurTok != tok_identifier)
    return Error("expected identifier list after var");
}
```

Once all the variables are parsed, we then parse the body and create the AST node:

```c++
  // At this point, we have to have 'in'.
  if (CurTok != tok_in)
    return Error("expected 'in' keyword after 'var'");
  getNextToken();  // eat 'in'.

  ExprAST *Body = ParseExpression();
  if (Body == 0) return 0;

  return new VarExprAST(VarNames, Body);
}
```

Now that we can parse and represent the code, we need to support emission of LLVM IR for it. This code starts out with:

```c++
Value *VarExprAST::Codegen() {
  std::vector<AllocaInst *> OldBindings;

  Function *TheFunction = Builder.GetInsertBlock()->getParent();

  // Register all variables and emit their initializer.
  for (unsigned i = 0, e = VarNames.size(); i != e; ++i) {
    const std::string &VarName = VarNames[i].first;
    ExprAST *Init = VarNames[i].second;
```

Basically it loops over all the variables, installing them one at a time. For each variable we put into the symbol table, we remember the previous value that we replace in OldBindings.

```c++
  // Emit the initializer before adding the variable to scope, this prevents
  // the initializer from referencing the variable itself, and permits stuff
  // like this:
  //  var a = 1 in
  //    var a = a in ...   # refers to outer 'a'.
  Value *InitVal;
  if (Init) {
    InitVal = Init->Codegen();
    if (InitVal == 0) return 0;
  } else { // If not specified, use 0.0.
    InitVal = ConstantFP::get(getGlobalContext(), APFloat(0.0));
  }

  AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, VarName);
  Builder.CreateStore(InitVal, Alloca);

  // Remember the old variable binding so that we can restore the binding when
  // we unrecurse.
  OldBindings.push_back(NamedValues[VarName]);

  // Remember this binding.
  NamedValues[VarName] = Alloca;
}
```

There are more comments here than code. The basic idea is that we emit the initializer, create the alloca, then update the symbol table to point to it. Once all the variables are installed in the symbol table, we evaluate the body of the var/in expression:

```c++
// Codegen the body, now that all vars are in scope.
Value *BodyVal = Body->Codegen();
if (BodyVal == 0) return 0;
Finally, before returning, we restore the previous variable bindings:

  // Pop all our variables from scope.
  for (unsigned i = 0, e = VarNames.size(); i != e; ++i)
    NamedValues[VarNames[i].first] = OldBindings[i];

  // Return the body computation.
  return BodyVal;
}
```

The end result of all of this is that we get properly scoped variable definitions, and we even (trivially) allow mutation of them :).

With this, we completed what we set out to do. Our nice iterative fib example from the intro compiles and runs just fine. The mem2reg pass optimizes all of our stack variables into SSA registers, inserting PHI nodes where needed, and our front-end remains simple: no “iterated dominance frontier” computation anywhere in sight.

# Structure and Interpretation of Computer Programs
Harold Abelson and Gerald Jay Sussman

# Chapter 1: Building Abstractions with Procedures

## 1.1 The Elements of Programming
* Every powerful language must have the following means to organize our ideas about processes:
  * Primitive expressions: the simplest entities the language is concerned with.
  * Means of combination by which compound elements are built from simpler ones.
  * Means of abstraction by which compound elements can be named and manipulated as units.

### 1.1.1 Expressions
* Arguments are the values of operands.
* Prefix notation: placing the operator to the left of the operands.

### 1.1.2 Naming and the Environment
* Global environment: the memory used to keep track of name-object pairs (i.e. the values associated with symbols).

### 1.1.3 Evaluating Combinations
* The evaluation process itself is recursive. To eval a combination:
  1. Eval the subexpressions of a combination.
  2. Apply the procedure that is the value of the leftmost subexpression (the operator) to the arguments that are the values of the other subexpressions (the operands).
* Evaluation can be thought of as a tree in which the values of operands percolate upwards, starting from the terminal nodes and combining at higher and higher levels.
  * Known as tree accumulation.
* Recursion is a powerful technique for dealing with hierarchical, treelike objects.
* In the primitive base cases:
  * The values of numerals are the numbers they name.
  * The values of built-in operators are the machine instruction sequences that carry out the corresponding operations.
  * The values of other names are the objects associated with those names in the environment.
    * The second rule is a special case of the third: symbols like + and * are included in the global environment and the sequences of machine instructions are their "values".
  * Exceptions are special forms:
    * E.g. `(define x 3)` does not apply `define` to the values `x` and 3; it associates `x` with 3.

### 1.1.4 Compound Procedures
* Formal parameters: the names used within the body of the procedure to refer to the corresponding arguments of the procedure.

### 1.1.5 The Substitution Model for Procedure Application
* The Substitution Model: to apply a compound procedure, evaluate the body of the procedure with each formal parameter replaced by the corresponding argument.
  * A model only – interpreters don't evaluate procedure applications by manipulating the text of a procedure to substitute values for formal parameters. The substitution is interpreted using a local environment.
  * The substitution model breaks down when dealing with mutable data.
* Applicative-order evaluation: the interpreter first evaluates the operator and operands, then applies the resulting procedure to the resulting arguments.
  * Used by Lisp.
* Normal-order evaluation: The operands are evaluated only when they are needed. Operand expressions are substituted for parameters until an expression involving only primitive operators is obtained.
* For procedure applications that can be modeled using substitution and that yield legitimate values (i.e., are pure functions), normal-order and applicative-order evaluation produce the same result.
  * Normal order may be less efficient:
    ```scheme
    (square (+ 5 1)) ; =>
    ((* (+ 5 1) (+ 5 1))) ; (+ 5 1) must be evaluated twice.
    ```
  * Normal order is much more complex to deal with under certain conditions, e.g. non-deterministic evaluations.

### 1.1.6 Conditional Expressions and Predicates
* When a predicate of a `cond` expression is true, the interpreter returns the value of the corresponding *consequent expression*.
* Predicate-consequent expression pairs are called *clauses*:
  ```Scheme
  (cond (<p1> <e1>)
        (<pn> <en>))
  ```
* The value of the `cond` is undefined if none of the predicates are true.
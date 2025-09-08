
# Circom Basics

## The Advantage of `<==` Assign and Constrain

Circom further simplifies the witness population through its "assign and constrain" operator `<==`. 

### Example

Suppose we have the constraint:

```circom
z === x * y
```

If we supply the values for `x` and `y`, it would be a bit annoying to also have to supply the value for `z` because `z` only has one possible solution. With Circom, we use `<==` as follows:

```circom
z <== x * y
```

### Benefits

With this, the variable `z` no longer needs to be provided as an input as Circom populates it for us, and its value will be locked into `x * y` for the rest of the circuit.
## Templates and Components

    Templates define a blueprint for circuits, like a class defines the structure for objects in OOP (Object Oriented Programming).

    A component is an instantiation of a template, similar to how an object is an instance of a class in Object Oriented Programming.

### Creating a Template

```circom
    // create template
template SomeCircuit() {
  // .... stuff
}

// instantiate template 
component main = SomeCircuit();
```

## Main Component

`component main = SomeCircuit()` is needed because Circom requires a single top-level component, `main`, to define the circuit structure that will be compiled.

## Signal Inputs

Signal inputs are values that will be provided from outside the component. 

**Important Notes:**
- Circom does not enforce a value is actually provided — it is up to the developer to ensure that the values are actually supplied
- If they aren't provided, this can lead to a security vulnerability (this will be explored in a later chapter)
- Input signals are immutable and cannot be altered
- Signals are exactly the variables in a Rank 1 Constraint System witness vector

## Signal Ordering

Circom allows the default order to be changed via command-line argument.

## Finite Field Properties

The following should be obvious to the reader:

- `p` under mod `p` is congruent to `0`
- `p-1` is the largest integer in the finite field mod `p`
- Passing values that are larger than `p-1` will result in overflow

## Input Declaration

In Circom, we have the option to declare our inputs as separate signals or to declare an array which contains all the inputs. It is more conventional in Circom to group all the inputs into an array of signals called `in` instead of providing separate inputs `x` and `y`.

**Important:** Circom can only generate a proof for an input that actually satisfies the circuit.

## Compiling Circuits

```bash
circom somecircuit.circom --r1cs --sym --wasm
```

### Compilation Flags

- `--r1cs` flag means to output an r1cs file
- `--sym` flag gives the variables a human-readable name (more info can be found in the sym docs)
- `--wasm` is for generating wasm code to populate the witness of the R1CS, given an input JSON

**Note:** Observe that non-linear constraints are listed. Wires is the number of columns in the R1CS.

### Generated Files

The compiler creates the following:

- `somecircuit.r1cs` file
- `somecircuit.sym` file
- `somecircuit_js` directory

## Witness Generation Artifacts

The `somecircuit_js` directory contains artifacts for witness generation:

- `somecircuit.wasm`
- `generate_witness.js`
- `witness_calculator.js`

## File Types

### .r1cs File

This file contains the circuit's R1CS system of constraints in binary format. Can be used with different tool stacks to construct proving/verifying statements (e.g. snarkjs, libsnark).

**Viewing the file:**
```bash
snarkjs r1cs print somecircuit.r1cs
```

### .sym File

The `somecircuit.sym` file is a symbols file generated during compilation. This file is essential because:

- It maps human-readable variable names to their corresponding positions in the R1CS for debugging
- It helps in printing the constraint system in a more understandable format, making it easier to verify and debug your circuit

## Generating Witness

The `generate_witness.js` file is what we will use in the next section, the other two files are helpers for `generate_witness.js`. By supplying input values for the circuit, these artifacts will calculate the necessary intermediate values and create a witness that can be used to generate a ZK proof.

### Running Witness Generation

Run this command in the `somecircuit_js` directory:

```bash
node generate_witness.js somecircuit.wasm inputs.json witness.wtns
```

The output is the computed witness as a `witness.wtns` file.

### Viewing the Witness

If you run `cat witness.wtns`, the output is gibberish. This is because `witness.wtns` is a binary file in a format accepted by snarkjs.

To get the human-readable form, we export it to JSON via:

```bash
snarkjs wtns export json witness.wtns
```

We then view the JSON using:

```bash
cat witness.json
```

## Important Notes

### String Inputs

Circom expects strings instead of numbers because JavaScript does not work accurately with integers larger than 2^53.

### Output Formatting

The reason there is a 1 at the end is due to a flaw in how snarkjs formats the output. It is "trying" to say -1 1 but has no space between them.

## Programmatic Constraint Generation

Circom allows us to constrain an arbitrary number of signals using the following pattern to automatically generate the constraints:

```circom
template IsBinary(n) {
  // array of n inputs
  signal input in[n];

  // n loops: n constraints
  for (var i = 0; i < n; i++) {
    in[i] * (in[i] - 1) === 0;
  }
}

// instantiated w/ 4 inputs & 4 constraints
component main = IsBinary(4);
```

## Static Circuits Only

- Although constraints can be generated programmatically, the existence and configuration of constraints cannot conditionally depend on signals.
- While templates can use parameters, the circuit must be static and clearly defined. There is no support for “dynamic-length” circuits or constraints — everything must be fixed and well-defined from the start.
- If an R1CS’s structure were mutable based on input signal values, neither prover nor verifier could operate because the number of constraints would not be fixed.
- The value for `n` must be set at compile time.

## Variables vs Signals

Variables hold non-signal data and are mutable. Example of a variable declaration outside of a loop:

```circom
template VariableExample(n) {
  var acc = 2;
  signal s;
}
```

Key points:

- By default, variables are not part of the R1CS system of constraints.
- Variables can be used as additive or multiplicative constants inside the R1CS.
- Variables are used to compute values outside the R1CS to help define the R1CS.
- When working with variables, Circom behaves like a normal programming language.
- Math operations are done modulo `p`. Keep in mind:
  - `/` means multiplication with the multiplicative inverse
  - `\` means integer division
- Valid operators for signals are only `+`, `*`, `===`, `<--`, and `<==`.

## No Conditional Constraints on Signals

The following is not allowed because signal `a` is used as the conditional for the if statement:

```circom
template IfStatementViolation() {
  signal input a;
  signal input b;

  if (a == 2) {
    b === 3;
  } else {
    b === 4;
  }
}
```

In an R1CS, there can only be addition and multiplication between signals. Circom is a thin wrapper over R1CS and cannot translate arbitrary control flow like `if` into pure addition/multiplication constraints.

## Assertions and Constants

- `assert(n > 1)` does not generate constraints; it prevents instantiation when the template parameter condition is not met.
- Enforce a signal value via `signal === var` (same as `signal === 5`).

### No `const` Keyword

Circom does not have a constant keyword. Use variables to assign names to magic numbers for readability.

### Variables with Signals

Note:

- If variables are added to or multiplied with a signal, the variable compiles to a constant in the R1CS.
- For signals, only addition, subtraction, or multiplication by constants are allowed; subtraction is addition with the additive inverse.
- Dividing a signal by a constant multiplies it by the constant's multiplicative inverse (division by 0 is invalid and will not compile).

## Circom Constraints

A Rank 1 Constraint System has at most one multiplication between signals per constraint. This is called a "quadratic" constraint. Any constraint containing an operation other than addition or multiplication will be rejected by Circom with the "Non quadratic constraints are not allowed" error.

If an illegal operation (not addition or multiplication) is used in a constraint, the Circom compiler will report the "Non quadratic constraints are not allowed!" error.

### Signals Cannot Be Used to Index Signal Arrays

The following operation will result in a quadratic constraints violation. There is no direct translation from array indexing to addition and multiplication.

```circom
template KMustEqual5(n) {
  signal input in[n];
  signal input k;

  // not allowed
  in[k] === 5;
}
```

### Signals Cannot Use Operations Such as `%` and `<<`

The following constraints will create a "Non quadratic constraints are not allowed!" violation:

```circom
template Example() {
  signal input a;
  signal input b;

  // not allowed
  a === b % 5;

  // not allowed
  a === b << 2;
}
```

## Division Operations

### Division by Constants

Somewhat subtly, Circom will allow "division" by a constant, because it can simply be replaced by multiplication by that number's multiplicative inverse. As such, the following code is valid:

```circom
template Example() {
  signal input a;
  signal input b;

  a === b / 2;
}

component main = Example();
```

### Division Between Signals (Not Allowed)

However, dividing signals is not allowed because that means we computed the signal's multiplicative inverse, which does not have a direct translation to only addition and multiplication.

```circom
template Example() {
  signal input a;
  signal input b;
  signal input c;

  // not allowed
  a === b / c;
}

component main = Example();
```

### Subtraction of Signals (Allowed)

In contrast, subtraction of signals is allowed because it directly translates to multiplication by the constant -1:

```circom
template Example() {
  signal input a;
  signal input b;

  // allowed
  a === b - a;

  // equivalent
  a === b + -1*a
}

component main = Example();
```

### Integer Division with `\` Operator

Integer division, as opposed to multiplication by the modular inverse, is represented by the `\` operator and it is not allowed to be applied to signals:

```circom
template Example() {
  signal input a;
  signal input b;

  // can only use \ with variables
  // not signals
  a === b \ 2;
}

component main = Example();
```

### Division Summary

- **For variables**: You have both integer division (`\`) and "normal" division (i.e., multiplication with the multiplicative inverse of the divisor).
- **For signals**: Only "normal" division (multiplication by multiplicative inverse) is allowed.

## Symbolic Variables in Circom

When a signal is assigned to a variable (thereby turning it into a symbolic variable), the variable becomes a container for that signal and for any arithmetic operations applied to it. A symbolic variable is declared using the `var` keyword, just like other variables.

### Equivalent Circuits Example

For example, the following two circuits are equivalent, i.e. they produce the same underlying R1CS:

```circom
template ExampleA() {
    signal input a;
    signal input b;
    signal input c;

    a * b === c;
}
```

```circom
template ExampleB() {
    signal input a;
    signal input b;
    signal input c;

    // symbolic variable v "contains" a * b
    var v = a * b;

    // a * b === c under the hood
    v === c;
}
```

In ExampleB, the symbolic variable `v` is simply a placeholder for the expression `a * b`. Both ExampleA and ExampleB are compiled using the exact same R1CS, and there is zero functional difference between them.

### Common Use Case: Summing Signals in Loops

Symbolic variables are extremely handy if we want to sum up an array of signals in a loop. In fact, summing signals in a loop is their most common use case:

```circom
// assert sum of in === sum
template Sum(n) {
    signal input in[n];
    signal input sum;

    var accumulator;
    for (var i = 0; i < n; i++) {
        accumulator += in[i];
    }

    // in[0] + in[1] + in[2] + ... + in[n - 1] === sum
    accumulator === sum;
}
```

### Important Considerations

- Because symbolic variables can "contain" a multiplication between two signals, they can lead to embedding two multiplications into one constraint if we aren't careful.
- Doing operations like computing the modulo or bitshifting are allowed with (non-symbolic) variables. However, this means that the variable can no longer be used as part of a constraint.
- Only regular variables can be used to determine the boundary of a loop or the condition of an if statement. If a symbolic variable is used, then the code will not compile.

### Symbolic Variables Summary

**Definition:** Symbolic variables are variables that were assigned a value from a signal. They are most frequently used for adding a parameterizable number of signals together, as the sum can be accumulated in a for loop.

**Key Properties:**
- They are effectively a "container" or "bucket" that holds either a single signal or a collection of signals that are added or multiplied together
- If a variable is never assigned a value from a signal, then it is not a symbolic variable
- Since symbolic variables contain signals, care must be taken to avoid quadratic constraint violations when using them

## Intermediate Signals and Assignment

To avoid the hassle of supplying `s`, Circom offers the `==>` and `<==` operators that assigns the value of `s` to be calculated by Circom (remember that part of Circom's functionality is to generate the witness). Thus, the value of `s` will not need to be supplied as an input. The `==>` and `<==` operators (precisely) means "assign and constrain:"

```circom
template Mul3() {
  signal input a;
  signal input b;
  signal input c;
  signal input d;

  // no longer an input
  signal s;

  a * b ==> s;
  s * c === d;
}
```

### Arrow Direction Flexibility

Circom is flexible on the direction of the arrow, `a * b ==> s` means the same as `s <== a * b`.

### Intermediate Signals

In the code above, `s` is called an intermediate signal. An intermediate signal is a signal defined as `signal` keyword without the `input` keyword. Therefore, `signal s` is an intermediate signal, but `signal input a` is not.

The underlying R1CS is identical between the two templates above. The `==>` simply saves us the hassle of supplying the value for `s` as part of the input.

### Witness Vector and R1CS

Assuming the witness vector is represented as `[1, a, b, c, d, s]`, the underlying R1CS would be as follows:

This can be thought of as passing Circom the witness `[1, a, b, c, d, _]` and Circom computing the full witness `[1, a, b, c, d, s]` based on the input.

**Key Points:**
- Assignment to `s` happens outside the R1CS
- The R1CS only checks that a matrix equation is satisfied by the witness vector
- The R1CS expects the witness to be provided and does not compute any of its values
- This approach simplifies circuit design and reduces the manual effort while keeping the R1CS structure unchanged
- A signal represents a concrete entry in the witness vector. Thus, it cannot change the value once it is set

## Intermediate Signal Arrays

If we wanted to enforce that the input signal `k` is the result of the product of all the signals in the array, this would introduce a significant amount of intermediate signals. To keep the code clean, we can have all the intermediate signals be assigned to a separate array as follows:

```circom
template KProd(n) {
  signal input in[n];
  signal input k;

  // intermediate signal array
  signal s[n];

  s[0] <== in[0];
  for (var i = 1; i < n; i++) {
    s[i] <== s[i - 1] * in[i];
  }

  k === s[n - 1];
}
```

## Component Reuse

Suppose we had to do this twice with eight inputs. In this case, it might be tempting to copy and paste the code twice for the inputs `(a,b,c,d)`, and `(x,y,z,u)`, which would be ugly.

Instead, we can put `Mul3` as a separate template as follows:

```circom
// separate template
template Mul3() {
  signal input a;
  signal input b;
  signal input c;
  signal input d; // d === a * b * c

  // no longer an input
  signal s;

  a * b ==> s;
  s * c === d;
}

// main component
template Mul3x2() {
  signal input a;
  signal input b;
  signal input c;
  signal input d; // d === a * b * c

  signal input x;
  signal input y;
  signal input z;
  signal input u; // u === x * y * z

  component m3_1 = Mul3();
  m3_1.a <== a;
  m3_1.b <== b;
  m3_1.c <== c;
  m3_1.d <== d;

  component m3_2 = Mul3();
  m3_2.a <== x;
  m3_2.b <== y;
  m3_2.c <== z;
  m3_2.d <== u;
}
```

### Component Usage Notes

- We declare components with the syntax `component m3_1 = Mul3();`. This is the same syntax we use to declare the main component.
- We "connect" the signals using the `<==` operator.
- The code above is entirely equivalent to copying and pasting the core logic of `Mul3` twice.

## Output Signals and Sub-components

It would be handy in some situations if a sub-component could "pass results back" to the component that created it.

For example, the following main component uses a sub-component `Square` to assign and constrain `out` to be the square of `in`:

```circom
template Square() {
  signal input in;
  signal output out;

  out <== in * in;
}

template Main() {
  signal input a;
  signal input b;
  signal input sumOfSquares;

  component a2 = Square();
  component b2 = Square();

  a2.in <== a;
  b2.in <== b;

  // assert that a^2 + b^2 === sum of Squares
  a2.out + b2.out === sumOfSquares;
}

component main = Main();
```

### Output Signal Definition

In the context of sub-components, an output signal is a signal that expects to be assigned a value via the `<==` operator and can be used to pass values back to the component that created it.

## Circomlib Library

The circomlib library is a library of Circom templates for various common operations. One such operation is to convert a binary array to a signal:

```circom
include "circomlib/bitify.circom";

template Main(n) {
  signal input in[n];
  signal input v;

  // instantiate the Bits2Num component
  component b2n = Bits2Num(n);

  // loop over each binary value
  // and assign and constrain it to the
  // b2n input array
  for (var i = 0; i < n; i++) {
    b2n.in[i] <== in[i];
  }

  b2n.out === v;
}

component main = Main(4);
```

## Anonymous Components

Rather than assign the input signals to a component separately, it is possible to provide them as an argument. This is called an "anonymous component." Consider the following example:

```circom
template Mul() {
  signal input in[2];
  signal output out;

  out <== in[0] * in[1];
}

template Example() {
  signal input a;
  signal input b;
  signal output out;

  // one line instantiation
  out <== Mul()([a, b]);
}

component main = Example();
```

### Security Consideration

An output signal must be part of constraints in the component that instantiated it. If an output signal is left "floating" then in some circumstances, a malicious prover can assign any value to it.

## Summary

- The `<==` and `==>` saves us the hassle of supplying the value of a signal explicitly in the `input.json`.
- We can use `<==` or `==>` whenever the value of one signal is directly determined by the value of another.
- `<==` is equivalent to `==>`. The arguments are simply reversed, but the effect is the same.
- Components can instantiate other sub-components and send values to their input signals using `<==` or `==>`.
- The output signals of a sub-component should be constrained to equal other signals in the component that instantiated it.





## Indicate Then Constrain

### The Problem

We want to say that "x is less than 5 or x is greater than 17." In this case, we cannot just combine both conditions directly, because if x is less than 5, it will violate the constraint that x is greater than 17 and vice versa.

### The Solution

The solution is to create indicator signals of the different conditions (e.g., x being less than 5, or being greater than 17), then apply constraints to the indicators.

### Circomlib Comparator Library

The Circomlib comparator library contains a component `LessThan` that returns 0 or 1 to indicate if `in[0]` is less than `in[1]`.

## Comparison Operations

### Bit-based Comparisons

For general n-bit numbers, we can check if x is greater than or equal to 2^n by checking if the most significant bit is set.

Here is a minimal example of using the `LessThan` template:

```circom
include "circomlib/comparators.circom";

template Example () {
  signal input a;
  signal input b;
  signal output out;

  // 252 will be explained in the next section
  out <== LessThan(252)([a, b]);
}

component main = Example();

/* INPUT = {
  "a": "9",
  "b": "10"
} */
```

### Finite Field Limitations

Numbers in a finite field (which is what Circom uses) cannot be compared to each other as "less than" or "greater" since the typical algebraic laws of inequalities do not hold.

The 252 specifies the number of bits in the `LessThan` component to limit how large x and y can be, so that a meaningful comparison can be made (the section above used 4 bits as an example).

Circom can hold numbers up to 253 bits large in the finite field. For security reasons discussed in the Alias Check chapter, we should not convert a field element to a binary representation that can encode numbers larger than the field. Therefore, Circom does not allow comparison templates to be instantiated with more than 252 bits.

## Practical Examples

### Example 1: x is less than 5 or x is greater than 17

Thankfully, the Circomlib library will do the bulk of the work for us. We will use the output signals of `LessThan` and `GreaterThan` components to indicate if x is less than 5 or greater than 17.

Then, we constrain that at least one of them is 1 by using the OR component (which is simply `out <== a + b - a * b` under the hood).

```circom
pragma circom 2.1.6;

include "circomlib/comparators.circom";
include "circomlib/gates.circom";

template DisjointExample1() {
  signal input x;

  signal indicator1;
  signal indicator2;

  indicator1 <== LessThan(252)([x, 5]);
  indicator2 <== GreaterThan(252)([x, 17]);

  component or = OR();
  or.a <== indicator1;
  or.b <== indicator2;

  or.out === 1;
}

component main = DisjointExample1();

/* INPUT = {
  "x": "18"
} */
```

**Important:** It is very important to include the constraint `or.out === 1;`, otherwise the circuit would accept the signals `indicator1` and `indicator2` both being zero.

### Example 2: Both x < 100 and y < 100

To express the above case where both x < 100 and y < 100, we can use a NAND gate:

```circom
pragma circom 2.1.6;

include "circomlib/comparators.circom";
include "circomlib/gates.circom";

template DisjointExample2() {
  signal input x;
  signal input y;

  component nand = NAND();
  nand.a <== LessThan(252)([x, 100]);
  nand.b <== LessThan(252)([y, 100]);
  nand.out === 1;   
}

component main = DisjointExample2();

/* INPUT = {
  "x": "18",
  "y": "100"
} */
```

### Example 3: k is greater than at least 2 of x, y, or z

In this example, we are trying to express that k is greater than x and y or k is greater than x and z, or k is greater than y and z:

```circom
pragma circom 2.1.6;

include "circomlib/comparators.circom";
include "circomlib/gates.circom";

template DisjointExample3(n) {
  signal input k;
  signal input x;
  signal input y;
  signal input z;

  signal totalGreaterThan;

  signal greaterThanX;
  signal greaterThanY;
  signal greaterThanZ;

  greaterThanX <== GreaterThan(252)([k, x]);
  greaterThanY <== GreaterThan(252)([k, y]);
  greaterThanZ <== GreaterThan(252)([k, z]);

  totalGreaterThan = greaterThanX + greaterThanY + greaterThanZ;

  signal atLeastTwo;
  atLeastTwo <== GreaterEqThan(252)([totalGreaterThan, 2]);
  atLeastTwo === 1;
}

component main = DisjointExample3();

/* INPUT = {
  "k": 20,
  "x": 18,
  "y": 100,
  "z": 10
} */
```

## Security Warning: Constrain Component Outputs!

**Do not forget to constrain the outputs of components!**

Sometimes, developers may forget to constrain the output of components, which can lead to severe security vulnerabilities! For example, the following code might seem like it enforces that x and y both be equal 1, but this is not the case. x and y could be zero (or any other arbitrary value). The output of the AND gate will be zero if x and y are zero, but the output is not constrained to be anything.

```circom
template MissingConstraint1() {
  signal input x;
  signal input y;

  component and = AND();
  and.a <== x;
  and.b <== y;

  // and.out is not constrained, so x and y can have any values!
}
```

## Compute Then Constrain

“Compute then constrain” is a design pattern in ZK circuits where an algorithm’s correct output is first computed without constraints. The correctness of the solution is then verified by enforcing invariants related to the algorithm.

For example, computing the square root of a number may require several iterative estimates, which would make the circuit considerably larger. It is often more practical to run the computation outside the circuit (no constraints), then add constraints that are satisfied if and only if the computed answer is correct.

### The “thin arrow” `<--` vs `<==`

We need a mechanism to tell Circom “compute and assign the value for this signal as a function of other signals, but do not create a constraint.” The syntax for that operation is the `<--` operator:

```circom
template InputEqualsZero() {
  signal input in;
  signal output out;

  // out = 1 if in == 0
  out <-- (in == 0 ? 1 : 0);
}

component main = InputEqualsZero();
```

The operation `in == 0 ? 1 : 0` is sometimes called an “out-of-circuit computation” or a “hint.” The code above compiles, but `out` and `in` have no constraints applied to them.

### What happens during witness generation

- Check `in == 0`
- If true → set `out = 1`
- If false → set `out = 0`
- The value of `out` appears in the witness file
- No constraint exists that enforces the relation in the R1CS

Result: A prover could supply `in = 5, out = 1` and still pass verification because there is no equation linking `in` and `out`.

### Important security note

The `<--` operator is convenient because it computes values without generating constraints, avoiding the need to manually supply certain signal values. However, it is a common source of security bugs.

**Important:** Circom does not enforce that developers create constraints after using `<--`. Even if no constraints are added (dangerous), the compiler will not warn. Unconstrained signals can take any value, allowing the circuit to produce ZK proofs for nonsensical statements.


### Modular Square Root

- The `pointbits` file in circomlib provides the function for computing the modular square root. Note that functions must be declared outside of a Circom template. A “function” in Circom is simply a convenience for grouping related code.

Modular square roots have two solutions: the square root itself and its additive inverse. Thus, we can generate both solutions as follows:

```circom
template ValidSqrt() {
  signal input in;
  signal output out1; // sqrt(in)
  signal output out2; // -sqrt(in)

  out1 <-- sqrt(in);
  out2 <-- out1 * -1; // computation (unconstrained)
  out1 * out1 === in; // verification (constraint)
  out2 * out2 === in; // verification (constraint)
}
```

WARNING: The code presented here is hardcoded to the default field size of Circom. If you configure Circom to use some other field, it may produce the wrong answer!

This demonstrates “compute then constrain”: compute the square root without constraints (keeps the circuit small), then enforce correctness with constraints.

This illustrates how Circom is both a programming language and a constraint DSL. The function `sqrt(n)` is program code; `in === out * out` generates constraints.


### Modular Inverse

Suppose we want to compute the multiplicative inverse of signal `in`, i.e., find a signal `out` such that `out * in === 1`.

Circom interprets the `/` operator as modular division, so the inverse of a value `n` can be computed as:

```circom
inv <-- 1 / n;
```

```circom
template MulInv() {
  signal input in;
  signal output out;

  // compute
  out <-- 1 / in;

  // then constrain
  out * in === 1;
}

component main = MulInv();
```

Note: Modular division is a non-quadratic operation, so it must be used only with variables or with the thin-arrow assignment (`<--`) — i.e., computed out-of-circuit.

### Advice Inputs (Non-deterministic Inputs)

Values computed outside the circuit that enable more concise constraints are called “advice inputs” or “non-deterministic inputs.” The `inv` signal in the modular inverse example above is an advice input: it is computed out-of-circuit using `<--` and then constrained in-circuit.

## Components in Loops

Circom does not allow components to be directly instantiated inside a loop. For example, compiling the following code results in an error:

```circom
include "./node_modules/circomlib/circuits/comparators.circom";

template IsSorted(n) {
  signal input in[n];

  for (var i = 0; i < n; i++) {
    component lt = LessEqThan(252); // error here
    lt.in[0] <== in[0];
    lt.in[1] <== in[1];
    lt.out === 1;
  }
}

component main = IsSorted(8);
```

Error:

```
Signal or component declaration inside While scope. Signal and component can only be defined in the initial scope or in If scopes with known condition
```

### Workaround: Component Arrays

Declare an array of components outside the loop without specifying the type. Then, inside the loop, assign the concrete component type and wire signals.

```circom
pragma circom 2.1.8;
include "./node_modules/circomlib/circuits/comparators.circom";

template IsSorted(n) {
  signal input in[n];

  // declare array of components, type assigned later
  component lessThan[n];

  for (var i = 0; i < n - 1; i++) {
    lessThan[i] = LessEqThan(252);
    lessThan[i].in[0] <== in[i];
    lessThan[i].in[1] <== in[i+1];
    lessThan[i].out === 1;
  }
}

component main = IsSorted(8);
```

Note: When components are declared this way, one-line anonymous instantiation inside a loop is not possible. Outside a loop it works; inside a loop, write explicit wiring.

```circom
pragma circom 2.1.8;
include "./node_modules/circomlib/circuits/comparators.circom";

template IsSorted() {
  signal input in[4];
  signal leq1;
  signal leq2;
  signal leq3;

  // one-line assignment (allowed outside loops)
  leq1 <== LessEqThan(252)([in[0], in[1]]);
  leq2 <== LessEqThan(252)([in[1], in[2]]);
  leq3 <== LessEqThan(252)([in[2], in[3]]);

  leq1 === 1;
  leq2 === 1;
  leq3 === 1;
}

component main = IsSorted();
```

### Example: All Items Unique

```circom
pragma circom 2.1.8;
include "./node_modules/circomlib/comparators.circom";

template ForceNotEqual() {
  signal input in[2];

  component iseq = IsEqual();
  iseq.in[0] <== in[0];
  iseq.in[1] <== in[1];
  iseq.out === 0;
}

template AllUnique (n) {
  signal input in[n];

  // runs n * (n - 1) / 2 times
  component Fneq[n * (n-1)/2];

  var index = 0;
  for (var i = 0; i < n - 1; i++) {
    for (var j = i + 1; j < n; j++) {
      Fneq[index] = ForceNotEqual();
      Fneq[index].in[0] <== in[i];
      Fneq[index].in[1] <== in[j];
      index++;
    }
  }
}

component main = AllUnique(5);
```

### Summary

- Declare `component arr[size];` outside loops without a type.
- Inside loops, assign the type and wire inputs/outputs explicitly.



## The whole chapter is important [Hacking Underconstrained Circom Circuits With Fake Proofs](https://rareskills.io/post/underconstrained-circom)

## Practice Problems

- [Hello World Circom Practice Problems](https://rareskills.io/post/hello-world-circom#practice-problems)
- [Circom Syntax Practice Problems](https://rareskills.io/post/circom-syntax#practice-problems)

### Challenge: Quadratic Roots over a Finite Field

- Write a Circom function that finds the roots of a degree-2 polynomial using the quadratic formula. Use the modular square root from the example above.
- Add constraints that the two roots (if they exist) satisfy the polynomial.
- Pass the polynomial to the Circom template as an array of three coefficients.
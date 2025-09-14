## Conditional Statements in Circom

Circom is very strict with the usage of if-statements. The following rules must be followed:

    Signals cannot be used to alter the behavior of an if-statement.
    A signal cannot be assigned a value inside an if-statement.

If-statements are acceptable if they are not affected by any signals, and do not affect any signals.

Effectively, they are not part of the underlying Rank 1 Constraint system (R1CS).

## Branching in Circom

It might seem that Circom is incapable of conditional branching, but this is not the case. To create conditional branches in Circom, all branches of a statement must be executed, with the ‘unwanted’ branches multiplied by zero and the ‘correct’ branch multiplied by one.

### Conditions (Python)

```python
def foo(x):
  if x == 5:
    out = 14
  elif x == 9:
    out = 22
  elif x == 10:
    out = 23
  else:
    out = 45
  return out
```

### Circom implementation

```circom
include "./node_modules/circomlib/circuits/comparators.circom";

template MultiBranchConditional() {
  signal input x;

  signal output out;

  signal x_eq_5;
  signal x_eq_9;
  signal x_eq_10;
  signal otherwise;

  x_eq_5 <== IsEqual()([x, 5]);
  x_eq_9 <== IsEqual()([x, 9]);
  x_eq_10 <== IsEqual()([x, 10]);
  otherwise <== IsZero()(x_eq_5 + x_eq_9 + x_eq_10);

  signal branches_5_9;
  signal branches_10_otherwise;

  branches_5_9 <== x_eq_5 * 14 + x_eq_9 * 22;
  branches_10_otherwise <== x_eq_10 * 23 + otherwise * 45;

  out <== branches_5_9 + branches_10_otherwise;
}

component main = MultiBranchConditional();
```




## Many branches (inner product formulation)

We can think of the computation as the inner product (generalized dot product) of the switches and the branches.

To implement this efficiently in Circom, we use the EscalarProduct template from multiplexer.circom . This template takes two vectors of length n, multiplies them element-wise, and sums the result. In the following code block, we use EscalarProduct to multiply each switch by each branch. Note that the final switch and branch are handled slightly differently because the final condition is a “catch-all” else statement.

```circom
include "./node_modules/circomlib/circuits/comparators.circom";
include "./node_modules/circomlib/circuits/multiplexer.circom";

template BranchN(n) {
  assert(n > 1); // too small

  signal input x;

  // conds n - 1 is otherwise
  signal input conds[n - 1];

  // branch n - 1 is the otherwise branch
  signal input branches[n];
  signal output out;

  signal switches[n];

  component EqualityChecks[n - 1];

  // only compute IsEqual up to the second-to-last switch
  for (var i = 0; i < n - 1; i++) {
    EqualityChecks[i] = IsEqual();

    EqualityChecks[i].in[0] <== x;
    EqualityChecks[i].in[1] <== conds[i];
    switches[i] <== EqualityChecks[i].out;
  }

  // check the last condition
  var total = 0;
  for (var i = 0; i < n - 1; i++) {
    total += switches[i];
  }

  // if none of the first n - 1 switches
  // are active, then `otherwise` must be 1
  switches[n - 1] <== IsZero()(total);

  component InnerProduct = EscalarProduct(n);
  for (var i = 0; i < n; i++) {
    InnerProduct.in1[i] <== switches[i];
    InnerProduct.in2[i] <== branches[i];
  }

  out <== InnerProduct.out;
}

template MultiBranchConditional() {
    signal input x;

    signal output out;

    component branchn = BranchN(4);

  var conds[3] = [5, 9, 10];
  var branches[4] = [14, 22, 23, 45];
  for (var i = 0; i < 4; i++) {
    if (i < 3) {
        branchn.conds[i] <== conds[i];
    }

    branchn.branches[i] <== branches[i];
  }

  branchn.x <== x;
  branchn.out ==> out; // same as out <== branch4.out
}

component main = MultiBranchConditional();
```


## When is it okay to use if-statements?

Suppose we wanted to create a template that returns a completely different circuit depending on the circuit parameter. For example, if we are creating a Max component that takes an array in[n] and returns the max, it would be more efficient to simply return the 0th item in the index if n is equal to 1.

Below, we show an example of a valid use of the if-statement when used with defining constraints. Here, the if-statement is executed at compile time, so the template will produce a well-defined circuit:

```circom
include "./node_modules/circomlib/circuits/comparators.circom";

template Max(n) {
  signal input in[n];
  signal output out;

  assert(n > 0);

  if (n == 1) {
    out <== in[0];
  }

  // it is okay to declare signals inside
  // the if-statement because the evaluation
  // of the if-statement is known at compile time
  else if (n == 2) {
    signal zeroGtOne;
    signal branch0;
    signal branch1;

    zeroGtOne <== GreaterThan(252)([in[0], in[1]]);
    branch0 <== zeroGtOne * in[0];
    branch1 <== (1 - zeroGtOne) * in[1];

    out <== branch0 + branch1;
  }
  else {
    // case for n > 2
  }
}

component main = Max(2);
```

## Guidelines

When using ZK to prove a computation, optimize for:

- Having as few branches as possible (each branch increases prover work)
- Minimizing total computational cost across all branches, not just expected cost
- Avoiding conditional statements where possible

## Quin Selector

The Quin Selector is a design pattern that allows us to use a signal as an index for an array of signals.

### Mathematical Definition

The Quin Selector is nothing more than:

```
out = ∑(i=0 to n-1) arr[i] ⋅ sel[i]
```

where:
- `arr[i]` = candidate values
- `sel[i]` = selector bits (0 or 1)
- exactly one `sel[i] = 1`, the rest are 0

So effectively it's scalar multiplication + sum that filters out all but one element.

## Circomlib Implementation of Quin Selector

The multiplexer in the Circomlib library accomplishes the same thing as Quin Selector. However, it indexes a 2-dimensional array and returns a 1-dimensional array. For example, given the array `in = [[5,5],[6,6],[7,7]]` and `index = 1`, it would return `[6, 6]`.

### Multiplexer Template

The component has the following inputs and outputs:

```circom
template Multiplexer(wIn, nIn) {
  signal input inp[nIn][wIn];
  signal input sel;
  signal output out[wIn];

  // ...
}
```

Using the example `in = [[5,5],[6,6],[7,7]]`, `wIn` would be 2 and `nIn` would be 3. The signal `sel` is the index to pick; for example if `sel = 1` then `out = [6,6]`.

### How It Works

Instead of looping through the array and checking if the index `IsEqual` to the `sel` value, the Multiplexer generates a "mask" of all zeros with a 1 at the desired index and multiplies that mask with the input. For example, if `sel = 1` it generates the mask `[0,1,0]` and multiplies the input by that mask.

### Example Usage

Here is an example of using Circomlib's multiplexer:

```circom
include "circomlib/multiplexer.circom";

template MultiplexerExample(n) {
  signal input in[n];
  signal input k;
  signal output out;

  component mux = Multiplexer(1, n);

  for (var i = 0; i < n; i++) {
    mux.inp[i][0] <== in[i];
  }
  mux.sel <== k;

  out <== mux.out[0];
}

component main = MultiplexerExample(4);

/* INPUT = {
  "in": [3, 7, 9, 11],
  "k": "1"
} */
```

### Historical Note

This algorithm was referred to as "Linear Scan" in the xjsnark paper, which predates the eth Dark Forest implementation.

## Introduction to Stateful Computations in ZK

When carrying out iterative computations such as powers, factorials, or computing the Fibonacci sequence, we need to "stop the computation" after a certain point. 

Therefore, the solution is to compute every possible value up to some limit greater than what we expect to compute in practice. Then we use a Quin selector to pick the desired value.

### Why This Approach is Necessary

- Circom does not support if statements, so the `if x == 0: return 0` line will not compile.
- Circom does not support loops of an unknown number of iterations. Since `x` determines the value of the loop, this also won't compile. Circom compiles to an R1CS under the hood, and the underlying R1CS needs to have a fixed size and can't change size based on the value of the inputs.

If we know we will never need to compute more than 99 factorial, then we must compute every factorial from 0 to 99 inclusive. If we want to create a proof for 80 factorial, we still need to compute the factorials from 0 to 99, but we use a Quin selector to return the result for 80.

We are creating an array of length 100 and populating the values with the factorial of that index. We will then "select" the factorial we care about using the Quin Selector.

## Factorial Example

```circom
include "./node_modules/circomlib/circuits/multiplexer.circom";
include "./node_modules/circomlib/circuits/comparators.circom";

template factorial(n) {
  signal input in;
  signal output out;

  // precompute factorials from 0 to n
  signal factorials[n+1];

  // compute the factorials
  factorials[0] <== 1;
  for (var i = 1; i <= n; i++) {
    factorials[i] <== factorials[i - 1] * i;
  }

  // ensure that in < n
  signal inLTn;
  inLTn <== LessThan(252)([in, n]);
  inLTn === 1;

  // select the factorial of interest
  component mux = Multiplexer(1, n);
  mux.sel <== in;

  // assign factorials into the multiplexer
  for (var i = 0; i < n; i++) {
    mux.inp[i][0] <== factorials[i];
  }

  out <== mux.out[0];
}

component main = factorial(100);

/*
  INPUT = { "in": "3" }
*/
```

## Fibonacci Example

The circuit below computes the Fibonacci sequence up to the nth number modulo p, then outputs the "in" Fibonacci number of interest.

```circom
include "./node_modules/circomlib/circuits/multiplexer.circom";
include "./node_modules/circomlib/circuits/comparators.circom";

template Fibonacci(n) {
  assert(n >= 2); // so we don't break the hardcoding

  signal input in; // compute the kth Fibonacci number
  signal output out;

  // precompute Fibonacci sequence from 0 to n
  signal fib[n + 1];

  // compute the Fibonacci sequence
  fib[0] <== 1;
  fib[1] <== 1;

  for (var i = 2; i < n; i++) {
    fib[i] <== fib[i - 1] + fib[i - 2];
  }

  // ensure that in < n
  signal inLTn;
  inLTn <== LessThan(252)([in, n]);
  inLTn === 1;

  // select the fibonacci number of interest
  component mux = Multiplexer(1, n);
  mux.sel <== in;

  // assign Fibonacci into the Quin Selector
  for (var i = 0; i < n; i++) {
    mux.inp[i][0] <== fib[i];
  }

  out <== mux.out[0];
}

component main = Fibonacci(99);

/*
  INPUT = {"in": 5}
*/
```

### Important Security Note

As usual, it is important to explicitly constrain each update of the Fibonacci sequence, and not simply compute the result in an unconstrained loop.

## Practice Problems

- [Power Circuit Example](https://github.com/RareSkills/zero-knowledge-puzzles/blob/main/Power/pow.circom)

## Swapping Two Items in an Array in Circom

This chapter shows how to swap two signals in a list of signals. This is an important subroutine for a sorting algorithm. More generally, lists are a fundamental building block for more interesting functions like hash functions or modeling memory in a CPU, so we must learn how to update their values.

### Challenges in a ZK Circuit

- **First**, we cannot directly index an array of signals. For that, we need to use a Quin selector.
- **Second**, we cannot "write to" a signal in an array of signals because signals are immutable.

Instead, we need to create a new array and copy the old values to the new array, subject to the following conditions:

- If we are at index `s`, write the value at `arr[t]`
- If we are at index `t`, write the value at `arr[s]`
- Otherwise, write the original value

Every write we make to the new array is a conditional operation.

### Edge Case Handling

We need to explicitly detect if `s == t` and multiply one of either `branchS` or `branchT` by zero to avoid doubling the value. In other words, if the switches for `s` and `t` are both active, then the resulting value would be `s + t`. But we don't want that, we want the value to effectively remain unchanged by selecting `branchS` or `branchT` arbitrarily (they will have the same value):

## Swap Implementation

```circom
template Swap(n) {
  signal input in[n];
  signal input s;
  signal input t;
  signal output out[n];

  // NEW CODE to detect if s == t
  signal sEqT;
  sEqT <== IsEqual()([s, t]);

  // get the value at s
  component qss = QuinSelector(n);
  qss.idx <== s;
  for (var i = 0; i < n; i++) {
    qss.in[i] <== in[i];
  }

  // get the value at t
  component qst = QuinSelector(n);
  qst.idx <== t;
  for (var i = 0; i < n; i++) {
    qst.in[i] <== in[i];
  }

  component IdxEqS[n];
  component IdxEqT[n];
  component IdxNorST[n];
  signal branchS[n];
  signal branchT[n];
  signal branchNorST[n];
  
  for (var i = 0; i < n; i++) {
    IdxEqS[i] = IsEqual();
    IdxEqS[i].in[0] <== i;
    IdxEqS[i].in[1] <== s;

    IdxEqT[i] = IsEqual();
    IdxEqT[i].in[0] <== i;
    IdxEqT[i].in[1] <== t;

    // if IdxEqS[i].out + IdxEqT[i].out
    // equals 0, then it is not i ≠ s and i ≠ t
    IdxNorST[i] = IsZero();
    IdxNorST[i].in <== IdxEqS[i].out + IdxEqT[i].out;

    // if we are at index s, write in[t]
    // if we are at index t, write in[s]
    // else write in[i]
    branchS[i] <== IdxEqS[i].out * qst.out;
    branchT[i] <== IdxEqT[i].out * qss.out;
    branchNorST[i] <== IdxNorST[i].out * in[i];

    // multiply branchS by zero if s equals T
    out[i] <== (1-sEqT) * (branchS[i]) + branchT[i] + branchNorST[i];
  }
}
```

## Conclusion

Any array manipulation in Circom requires creating a new array and copying the old values to the new one, except where the update happens.

By using this pattern in a loop, we can do things like:
- Sort a list
- Model data structures like stacks and queues
- Change the state of a CPU or VM

## ZK Proof of Selection Sort - Everything is Important

> **Note**: This section needs to be reviewed again for completeness.

The concept of an "intermediate state" and proving that we moved between intermediate states correctly is core to the verification of most ZK algorithms proved in practice, notably hash functions and ZK Virtual Machines. The Selection Sort algorithm presented in this chapter provides a gentle introduction to stateful computation.

## [ZK Stack Overview](https://rareskills.io/post/zk-stack) - Everything is important

## How a ZKVM Works - Everything is Important

A Zero-Knowledge Virtual Machine (ZKVM) is a virtual machine that can create a ZK-proof that verifies it executed a set of machine instructions correctly. This allows us to take a program (a set of opcodes), a virtual machine specification (how the virtual machine behaves, what opcodes it uses, etc), and prove that the output generated is correct. A verifier does not have to re-run the program, but only check the generated ZK proof — this allows verification to be succinct. Such succinctness is the basis of what makes ZK layer 2 blockchains scalable. It also enables someone to check that a machine learning algorithm ran as claimed without re-running the entire algorithm.

### Important Clarification

Contrary to the name, ZKVMs are rarely "zero knowledge" in the sense that they keep the computation private. Rather, they use ZK algorithms to produce a succinct proof that the program executed correctly on a certain input so that a verifier can double-check the computation with exponentially less work. Even though revealing the program input is optional, avoiding accidental data leaks and having multiple parties agree upon private state are very challenging engineering problems that still have unsolved challenges and scaling limitations.

## 32-Bit Emulation in ZK

### Why Do We Need 32-Bit Emulation?

Many cryptographic hash functions operate on 32-bit words since, historically, 32 bits was the default word size of many CPUs. This later increased to 64 bits. The EVM uses 256 bits so that it can easily accommodate the keccak256 hash.

If we want to use ZK to prove the correct execution of a traditional hash function or some virtual machine that does not use finite fields (most do not), then we need to "model" traditional datatypes with a field element. Therefore, we use a field element (signal) in Circom to hold a number that cannot exceed what a 32-bit number can hold, even though the signal can hold values much larger than 32 bits.

### Field Overflow Considerations

In Circom, or any language using the bn128 curve, the overflow happens at `21888242871839275222246405745257275088548364400416034343698204186575808495617`. In a 32-bit machine, the overflow happens at `4294967296` or, in general, at `2^n` where `n` is the number of bits in the virtual machine.

A more efficient approach would be to take advantage of binary representation. The key idea is to encode a number with 32 bits, and if it fits into 32 bits, the circuit executes normally. In contrast, if the number doesn't fit into 32 bits, then the constraints cannot be satisfied. Hence, the circuit below ensures that `in` is `2^32 - 1` or less.

### Range Check Implementation

```circom
include "circomlib/comparators.circom";

// 32 bit range check
template RangeCheck() {
  signal input in;
  component n2b = Num2Bits(32);
  n2b.in <== in;
}

component main = RangeCheck();

// if in = 2**32 - 1, it will accept
// if in = 2**32 it will reject
```

## 32-Bit Addition

Suppose we want to add two field elements `x` and `y` together, which represent 32-bit numbers.

The naïve implementation of 32-bit addition is to turn the field element into 32-bits, then build an "addition circuit" that adds each bit and carries the overflow. However, this creates a larger circuit than necessary.

Instead, we can do the following:

1. Range check `x` and `y` using the strategy outlined above
2. Add `x` and `y` together as field elements, i.e., `z <== x + y`
3. Convert `z` to a 33-bit number
4. Convert the least significant 32 bits of the 33-bit number to a field element.

## 32-Bit Multiplication

The logic for 32-bit multiplication is extremely similar to 32-bit addition, except that we need to allow for the 32-bit multiplication to temporarily require up to 64 bits before only saving the final 32 bits. The final number needs 64 bits to hold.

## 32-Bit Division and Modulo

Integer division is one of the most problematic bugs in ZK, as properly constraining it is much harder than the examples of addition and multiplication.

In integer division, the relationship between the numerator, denominator, quotient, and remainder is:
```
num = deno * quotient + remainder
```

However, that constraint alone is not sufficient to ensure that the division was conducted properly.

We can guard against this by adding a constraint that the remainder is strictly less than the denominator.

### Division Implementation

```circom
include "circomlib/comparators.circom";
include "circomlib/bitify.circom";

template DivMod(wordSize) {
  // a wordSize over this could overflow 252
  assert(wordSize < 125);

  signal input numerator;
  signal input denominator;

  signal output quotient;
  signal output remainder;

  quotient <-- numerator \ denominator;
  remainder <-- numerator % denominator;

  // quotient and remainder still need
  // to be range checked because the
  // prover can supply any value

  // range check all the signals
  component n2bN = Num2Bits(wordSize);
  component n2bD = Num2Bits(wordSize);
  component n2bQ = Num2Bits(wordSize);
  component n2bR = Num2Bits(wordSize);
  n2bN.in <== numerator;
  n2bD.in <== denominator;
  n2bQ.in <== quotient;
  n2bR.in <== remainder;

  // core constraint
  numerator === quotient * denominator + remainder;

  // remainder must be less than the denominator
  signal remLtDen;

  // depending on the application, we might be able
  // to use fewer than 252 bits
  remLtDen <== LessThan(wordSize)([remainder, denominator]);
  remLtDen === 1;

  // denominator is not zero
  signal isZero;
  isZero <== IsZero()(denominator);
  isZero === 0;
}

component main = DivMod(32);
```

## How ZK EVMs Handle 256-Bit Numbers

The default Circom field cannot hold 256-bit numbers. Instead, each word in the EVM must be modeled with a list of smaller word sizes, similar to how a 64-bit computer can emulate the EVM.

For example, a 256-bit number can be modeled with four 64-bit words. When adding, we carry the overflow from the less significant words to the next significant word. If the most significant word overflows, we simply discard the overflow.

## MD5 Hash In Circom

To create a proof that we know the preimage of the MD5 hash without revealing it, we need to prove we executed every step of the hash correctly and produced a certain result. Specifically, the MD5 hash has the following subroutines:

- Bitwise AND, OR, NOT, and XOR
- LeftRotate
- Add 32-bit numbers and overflow at 2^32
- The function Func, which combines registers B, C, and D together using bitwise operators
- The padding step at the beginning, which adds a 1 bit after the input and puts the length (in bits) of the input

Additionally, the output of MD5 is usually written as a 128-bit number in big-endian form.

```circom
include "circomlib/bitify.circom";
include "circomlib/gates.circom";

template BitwiseAnd32() {
    signal input in[2];
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    component n2bb = Num2Bits(32);
    n2ba.in <== in[0];
    n2bb.in <== in[1];

    component b2n = Bits2Num(32);
    component Ands[32];
    for (var i = 0; i < 32; i++) {
        Ands[i] = AND();
        Ands[i].a <== n2ba.out[i];
        Ands[i].b <== n2bb.out[i];
        Ands[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template BitwiseOr32() {
    signal input in[2];
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    component n2bb = Num2Bits(32);
    n2ba.in <== in[0];
    n2bb.in <== in[1];

    component b2n = Bits2Num(32);
    component Ors[32];
    for (var i = 0; i < 32; i++) {
        Ors[i] = OR();
        Ors[i].a <== n2ba.out[i];
        Ors[i].b <== n2bb.out[i];
        Ors[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template BitwiseXor32() {
    signal input in[2];
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    component n2bb = Num2Bits(32);
    n2ba.in <== in[0];
    n2bb.in <== in[1];

    component b2n = Bits2Num(32);
    component Xors[32];
    for (var i = 0; i < 32; i++) {
        Xors[i] = XOR();
        Xors[i].a <== n2ba.out[i];
        Xors[i].b <== n2bb.out[i];
        Xors[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template BitwiseNot32() {
    signal input in;
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    n2ba.in <== in;

    component b2n = Bits2Num(32);
    component Nots[32];
    for (var i = 0; i < 32; i++) {
        Nots[i] = NOT();
        Nots[i].in <== n2ba.out[i];
        Nots[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

// n is the number of bytes
template ToBytes(n) {
    signal input in;
    signal output out[n];

    component n2b = Num2Bits(n * 8);
    n2b.in <== in;

    component b2ns[n];
    for (var i = 0; i < n; i++) {
        b2ns[i] = Bits2Num(8);
        for (var j = 0; j < 8; j++) {
            b2ns[i].in[j] <== n2b.out[8*i + j];
        }
        out[i] <== b2ns[i].out;
    }
}

// n is the number of bytes
template Padding(n) {
    // 56 bytes = 448 bits
    assert(n < 56);

    signal input in[n];

    // 64 bytes = 512 bits
    signal output out[64];

    for (var i = 0; i < n; i++) {
        out[i] <== in[i];
    }

    // add 128 = 0x80 to pad the 1 bit (0x80 = 10000000b)
    out[n] <== 128;

    // pad the rest with zeros
    for (var i = n + 1; i < 56; i++) {
        out[i] <== 0;
    }

    var lenBits = n * 8;
    if (lenBits < 256) {
        out[56] <== lenBits;
    }
    else {
        var lowOrderBytes = lenBits % 256;
        var highOrderBytes = lenBits \ 256;
        out[56] <== lowOrderBytes;
        out[57] <== highOrderBytes;
    }
}
template Overflow32() {
    signal input in;
    signal output out;

    component n2b = Num2Bits(252);
    component b2n = Bits2Num(32);

    n2b.in <== in;
    for (var i = 0; i < 32; i++) {
        n2b.out[i] ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template LeftRotate(s) {
    signal input in;
    signal output out;

    component n2b = Num2Bits(32);
    component b2n = Bits2Num(32);

    n2b.in <== in;

    for (var i = 0; i < 32; i++) {
        b2n.in[(i + s) % 32] <== n2b.out[i];
    }

    out <== b2n.out;
}

template Func(i) {
    assert(i <= 64);
    signal input b;
    signal input c;
    signal input d;

    signal output out;

    if (i < 16) {
        component a1 = BitwiseAnd32();
        a1.in[0] <== b;
        a1.in[1] <== c;

        component a2 = BitwiseAnd32();
        component n1 = BitwiseNot32();
        n1.in <== b;
        a2.in[0] <== n1.out;
        a2.in[1] <== d;

        component o1 = BitwiseOr32();
        o1.in[0] <== a1.out;
        o1.in[1] <== a2.out;

        out <== o1.out;
    }
    else if (i >= 16 && i < 32) {
        // (D & B) | (~D & C)
        component a1 = BitwiseAnd32();
        a1.in[0] <== d;
        a1.in[1] <== b;

        component n1 = BitwiseNot32();
        n1.in <== d;
        component a2 = BitwiseAnd32();
        a2.in[0] <== n1.out;
        a2.in[1] <== c;

        component o1 = BitwiseOr32();
        o1.in[0] <== a1.out;
        o1.in[1] <== a2.out;

        out <== o1.out;
    }
    else if (i >= 32 && i < 48) {
        component x1 = BitwiseXor32();
        component x2 = BitwiseXor32();

        x1.in[0] <== b;
        x1.in[1] <== c;
        x2.in[0] <== x1.out;
        x2.in[1] <== d;

        out <== x2.out;
    }
    // i must be < 64 by the assert statement above
    else {
        component o1 = BitwiseOr32();
        component n1 = BitwiseNot32();
        n1.in <== d;
        o1.in[0] <== n1.out;
        o1.in[1] <== b;

        component x1 = BitwiseXor32();
        x1.in[0] <== o1.out;
        x1.in[1] <==c;

        out <== x1.out;
    }
}

// n is the number of bytes
template MD5(n) {

    var s[64] = [7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22,
     5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20,
     4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23,
    6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21];

    var K[64] = [0xd76aa478, 0xe8c7b756, 0x242070db, 0xc1bdceee,
     0xf57c0faf, 0x4787c62a, 0xa8304613, 0xfd469501,
     0x698098d8, 0x8b44f7af, 0xffff5bb1, 0x895cd7be,
     0x6b901122, 0xfd987193, 0xa679438e, 0x49b40821,
     0xf61e2562, 0xc040b340, 0x265e5a51, 0xe9b6c7aa,
     0xd62f105d, 0x02441453, 0xd8a1e681, 0xe7d3fbc8,
     0x21e1cde6, 0xc33707d6, 0xf4d50d87, 0x455a14ed,
     0xa9e3e905, 0xfcefa3f8, 0x676f02d9, 0x8d2a4c8a,
     0xfffa3942, 0x8771f681, 0x6d9d6122, 0xfde5380c,
     0xa4beea44, 0x4bdecfa9, 0xf6bb4b60, 0xbebfbc70,
     0x289b7ec6, 0xeaa127fa, 0xd4ef3085, 0x04881d05,
     0xd9d4d039, 0xe6db99e5, 0x1fa27cf8, 0xc4ac5665,
     0xf4292244, 0x432aff97, 0xab9423a7, 0xfc93a039,
     0x655b59c3, 0x8f0ccc92, 0xffeff47d, 0x85845dd1,
     0x6fa87e4f, 0xfe2ce6e0, 0xa3014314, 0x4e0811a1,
     0xf7537e82, 0xbd3af235, 0x2ad7d2bb, 0xeb86d391];

    var iter_to_index[64] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15,
     1, 6, 11, 0, 5, 10, 15, 4, 9, 14, 3, 8, 13, 2, 7, 12,
     5, 8, 11, 14, 1, 4, 7, 10, 13, 0, 3, 6, 9, 12, 15, 2,
    0, 7, 14, 5, 12, 3, 10, 1, 8, 15, 6, 13, 4, 11, 2, 9];

    signal input in[n];

    signal inp[64];
    component Pad = Padding(n);

    for (var i = 0; i < n; i++) {
        Pad.in[i] <== in[i];
    }
    for (var i = 0; i < 64; i++) {
        Pad.out[i] ==> inp[i];
    }

    signal data32[16];
    for (var i = 0; i < 16; i++) {
        data32[i] <== inp[4 * i] + inp[4 * i + 1] * 2**8 + inp[4 * i + 2] * 2**16 + inp[4 * i + 3] * 2**24;
    }

    var A = 0;
    var B = 1;
    var C = 2;
    var D = 3;
    signal buffer[65][4];
    buffer[0][A] <== 1732584193;
    buffer[0][B] <== 4023233417;
    buffer[0][C] <== 2562383102;
    buffer[0][D] <== 271733878;

    component Funcs[64];
    signal toRotates[64];
    component SelectInputWords[64];
    component LeftRotates[64];
    component Overflow32s[64];
    component Overflow32s2[64];
    for (var i = 0; i < 64; i++) {
        Funcs[i] = Func(i);
        Funcs[i].b <== buffer[i][B];
        Funcs[i].c <== buffer[i][C];
        Funcs[i].d <== buffer[i][D];

        Overflow32s[i] = Overflow32();
        Overflow32s[i].in <== buffer[i][A] + Funcs[i].out + K[i] + data32[iter_to_index[i]];

        // rotated = rotate(to_rotate, s[i])
        toRotates[i] <== Overflow32s[i].out;
        LeftRotates[i] = LeftRotate(s[i]);
        LeftRotates[i].in <== toRotates[i];

        // new_B = rotated + B
        Overflow32s2[i] = Overflow32();
        Overflow32s2[i].in <== LeftRotates[i].out + buffer[i][B];

        // store into the next state
        buffer[i + 1][A] <== buffer[i][D];
        buffer[i + 1][B] <== Overflow32s2[i].out;
        buffer[i + 1][C] <== buffer[i][B];
        buffer[i + 1][D] <== buffer[i][C];
    }

    component addA = Overflow32();
    component addB = Overflow32();
    component addC = Overflow32();
    component addD = Overflow32();

    // we hardcode initial state because we only
    // process one 512 bit block
    addA.in <== 1732584193 + buffer[64][A];
    addB.in <== 4023233417 + buffer[64][B];
    addC.in <== 2562383102 + buffer[64][C];
    addD.in <== 271733878 + buffer[64][D];

    signal littleEndianMd5;
    littleEndianMd5 <== addA.out + addB.out * 2**32 + addC.out * 2**64 + addD.out * 2**96;

    // convert the answer to bytes and reverse
    // the bytes order to make it big endian
    component Tb = ToBytes(16);
    Tb.in <== littleEndianMd5;

    // sum the bytes in reverse
    var acc;
    for (var i = 0; i < 16; i++) {
        acc += Tb.out[15 - i] * 2**(i * 8);
    }
    signal output out;
    out <== acc;
}

component main = MD5(10);

// The result out = 
// "RareSkills" in ascii to decimal
/* INPUT = {"in": [82, 97, 114, 101, 83, 107, 105, 108, 108, 115]} */

// The result is 246193259845151292174181299259247598493

// The MD5 hash of "RareSkills" is 0xb93718dd21d2f5081239d7a16cf69b9d when converted to decimal is 246193259845151292174181299259247598493
```

The R1CS produced by the code above is over fifty-two thousand rows long, as highlighted in the figure below. There are a lot of opportunities to reduce the size of the circuit, especially by not converting the field elements to 32-bit arrays every time we use them.
However, each word in an MD5 (and other modern hashes) is 32 bits, so it will take 32 times as many signals to represent compared to regular code.


## ZK-Friendly Hash Functions ([Watch](https://www.youtube.com/watch?v=_MIxjDs70W8) for better understanding)

ZK-friendly hash functions require far fewer constraints to prove and verify than traditional cryptographic hashes.

Traditional hashes like SHA256 or keccak256 rely heavily on bitwise operations (XOR, rotations). Proving those requires decomposing values into 32 bits — i.e., 32 signals per 32-bit word — which is expensive. ZK-friendly hashes instead operate directly on field elements using only modular addition and multiplication, avoiding bit-decomposition.

### Properties we care about

- Preimage resistance: given an output, finding an input is infeasible.
- Collision resistance: given an input–output pair, finding a different input with the same output is infeasible.
- Pseudorandomness: outputs appear random with no statistical relation to inputs.

We briefly describe two popular ZK-friendly hashes: MiMC and Poseidon.

### MiMC (high level)

- Input and output are single field elements.
- Uses a sequence of fixed public round constants `C[i]` (e.g., 91 of them), which can be generated transparently (e.g., repeated SHA256 of a seed like "MiMC"). Conventionally, `C[0] = 0`.
- Uses a fixed small exponent `e` (often 3 or 7). For security we need `gcd(e, p - 1) = 1`. In Circom’s default field, `gcd(7, p - 1) = 1`, while `gcd(3, p - 1) ≠ 1`.
- A typical round updates the state roughly as: `x <- (x + k + C[i])^e` (all operations mod p). In basic single-input mode, `k` can be 0.

#### Multi-input with MiMC (Merkle–Damgård-style sponge)

To hash multiple field elements into one:

- Hash the first input.
- Use the previous output as the `k` value (key/tweak) for the next round sequence.
- Absorb the next input and continue; the final state is the hash output.

### Poseidon (high level)

- Similar to MiMC but operates on a vector state and includes matrix multiplications between element-wise S-boxes and additions.
- For a single input, the state can be expanded to `[0, input]` and multiplied by carefully tuned 2×2 matrices each round, combined with S-boxes and round constants.
- Matrix mixing increases diffusion, allowing fewer rounds than MiMC for comparable security, often yielding fewer constraints overall for multi-input hashing.
- Instead of re-hashing per element, larger inputs use wider states and matrices to absorb more elements per round.

Note: Circomlib’s Poseidon implementation supports up to 17 field-element inputs. For large datasets this can be limiting, but Merkle tree use-cases only require hashing two elements.

### Pedersen hash (elliptic-curve based)

Pedersen hashing uses elliptic-curve operations. It is typically more expensive in-circuit than Poseidon or MiMC, but still far cheaper than traditional bit-oriented hashes, and benefits from strong assumptions related to the hardness of discrete log (pre-quantum).


## The Permutation Argument

A permutation argument proves that two lists contain the same elements, possibly in a different order. For example, `[2,3,1]` is a permutation of `[1,2,3]` and vice versa.

### How the permutation argument works

- See: [RareSkills — How the permutation argument works](https://rareskills.io/post/permutation-argument#how-the-permutation-argument-works)

### Vulnerability: Not hashing all elements

When generating random points in this manner, the hash must depend on all inputs to the computation. Otherwise, a malicious prover can keep the hash value fixed and tweak the output array until finding an intersection point of the polynomial.


## https://rareskills.io/post/how-does-tornado-cash-work
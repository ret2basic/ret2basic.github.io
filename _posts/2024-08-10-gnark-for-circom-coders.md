---
layout: post
title: gnark for Circom coders
---

{{ page.title }}
================

<p class="meta">August 10, 2024</p>

# Why writing this blogpost

There aren't many articles on the Internet on how to write gnark code. During my research, I found the following resources:

- [gnark doc](https://docs.gnark.consensys.io/), of course.
- ["How to use the Consenys's Gnark Zero Knowledge Proof library and disclosure of a DoS bug" by LambdaClass](https://blog.lambdaclass.com/how-to-use-the-consenyss-gnark-zero-knowledge-proof-library-and-disclosure-of-a-ddos-bug/) -> this article describes how to work with low-level gnark API, which isn't covered in official gnark doc.
- There are a few code examples in [gnark Github repo](https://github.com/Consensys/gnark/tree/master/examples), but there is no explanation besides comments.

The target audience is those who already know Circom very well but don't have experience with gnark. These two DSLs are in fact very similar but gnark is more high-level than Circom. Personally I think Circom is similar to x86 assembly and gnark is similar to C, loosely.

I will try to map terminologies in one-to-one correspondence and introduce some gnark features that Circom does not have.

> Before learning gnark, make sure you work through [A tour of Go](https://go.dev/tour/) at least once.

# Circom vs. gnark

## Circom

### How to declare signals/variables in Circom

In Circom, there are two types of variables:

1. public / private / intermediate signals -> these appear in the witness
2. variables declared by `var` -> not in the witness, just computation helpers

For example, from the [default zkREPL code example](https://zkrepl.dev/):

```javascript
pragma circom 2.1.6;

include "circomlib/poseidon.circom";
// include "https://github.com/0xPARC/circom-secp256k1/blob/master/circuits/bigint.circom";

template Example () {
    signal input a;
    signal input b;
    signal output c;
    
    var unused = 4;
    c <== a * b;
    assert(a > 2);
    
    component hash = Poseidon(2);
    hash.inputs[0] <== a;
    hash.inputs[1] <== b;

    log("hash", hash.out);
}

component main { public [ a ] } = Example();

/* INPUT = {
    "a": "5",
    "b": "77"
} */
```

Here `a` is a public signal since it is declared public when instantiating the template, `b` is a private signal by default, and `c` is a public ouput. These 3 signals will appear in the witness. In contrast, `unused` is just a helper variable that will not appear in the witness. This type of variables exists because we can work with them like normal imperative programming language instead of declarative DSL.

### How to write constraints in Circom

In Circom, you can add constraint by `===` (constrain) or `==>` (constrain and assign). For example:

```javascript
c <== a * b;
```

means compute `a * b` and assign the result to `c`, also add a constraint on this computation. Constraints are the most important things in a circuit. If you use `c <-- a * b` here, malicious prover can always modify the witness and give you some fake proof since there is no constraint on the value of `c`. It can be 1337, 1314521, or whatever value. It does not have to equal `a * b` if you don't add a constraint.

### How to use templates in Circom

To wire another template into your circuit, you would use something like:

```javascript
    component hash = Poseidon(2);
    hash.inputs[0] <== a;
    hash.inputs[1] <== b;

    log("hash", hash.out);
```

**Side note:** `hash.out` is underconstrained. In production-level code, always remember to add a constraint to output of external template, otherwise malicious prover can change it to anything.

This pattern is pretty tough to work work, to be honest. In gnark, you can use API function to write constraints, which is much easier.

## gnark

### How to declare variables in gnark

In gnark, variables are declared in a struct, usually named `Circuit`, but it really can be anything. For example, look at the [gnark playground default code example](https://play.gnark.io/):

```go
// Welcome to the gnark playground!
package main

import "github.com/consensys/gnark/frontend"

// gnark is a zk-SNARK library written in Go. Circuits are regular structs.
// The inputs must be of type frontend.Variable and make up the witness.
// The witness has a
//   - secret part --> known to the prover only
//   - public part --> known to the prover and the verifier
type Circuit struct {
	X frontend.Variable `gnark:"x"`       // x  --> secret visibility (default)
	Y frontend.Variable `gnark:",public"` // Y  --> public visibility
}

// Define declares the circuit logic. The compiler then produces a list of constraints
// which must be satisfied (valid witness) in order to create a valid zk-SNARK
func (circuit *Circuit) Define(api frontend.API) error {
	// compute x**3 and store it in the local variable x3.
	x3 := api.Mul(circuit.X, circuit.X, circuit.X)

	// compute x**3 + x + 5 and store it in the local variable res
	res := api.Add(x3, circuit.X, 5)

	// assert that the statement x**3 + x + 5 == y is true.
	api.AssertIsEqual(circuit.Y, res)
	return nil
}
```

Similar to Circom, if a variable is not declared public, then it is private by default, such as `X`. The `gnark:"x"` and `gnark:",public"` part is called a **"tag"**, it adds metadata to fields in a struct.

### How to write constraints in gnark

Constraints are written in a method called `Define`, and it needs to be linked to the struct you just defined. In this case, the struct containing all variable declarations is named `Circuit` and we name the new instance `circuit`. Later we can use `circuit` to access variables, such as `circuit.X` and `circuit.Y`.

In `Define`:

```go
func (circuit *Circuit) Define(api frontend.API) error {
	// compute x**3 and store it in the local variable x3.
	x3 := api.Mul(circuit.X, circuit.X, circuit.X)

	// compute x**3 + x + 5 and store it in the local variable res
	res := api.Add(x3, circuit.X, 5)

	// assert that the statement x**3 + x + 5 == y is true.
	api.AssertIsEqual(circuit.Y, res)
	return nil
}
```

It is easy to tell that the circuit verifies if `x**3 + x + 5 == y`. Note that most API functions take variable number of inputs. For example, `api.Mul(circuit.X, circuit.X)` computes `x**2` and `api.Mul(circuit.X, circuit.X, circuit.X)` computes `x**3`, and you can provide more inputs to it. It is also possible to write this in a for loop, for example, from [gnark doc](https://docs.gnark.consensys.io/HowTo/write/instructions):

```go
func (circuit *Circuit) Define(api frontend.API) error {
    for i := 0; i < n; i++ {
        circuit.X = api.Mul(circuit.X, circuit.X)
    }
    api.AssertIsEqual(circuit.X, circuit.Y)
    return nil
}
```

But similar to Circom, you can't write if statement for constraints. You can use a selector pattern to achieve similar result, but still pretty hard to work with:

```go
// Select if b is true, yields i1 else yields i2
func (cs *ConstraintSystem) Select(b Variable, i1, i2 interface{}) Variable {
    ...
}
```

You can find high-level API methods [here](https://github.com/Consensys/gnark/blob/master/frontend/api.go). Cryptography functions can be found [here](https://github.com/ConsenSys/gnark-crypto). Unfortunately, there is no easy way to reference these methods, you have to look through the resource code. But at least you know where to look at: if you encounter unfamiliar function names, it must reside in one of those two repos.

Besides high-level API, gnark also provides low-level API. The [LambdaClass article](https://blog.lambdaclass.com/how-to-use-the-consenyss-gnark-zero-knowledge-proof-library-and-disclosure-of-a-ddos-bug/) discussed how to use low-level API in gnark. This info is rarely discussed on the Internet and I found the article helpful. The article is nicely written so I am not going to repeat its content here, just go read it. To add a comment, I think it is better to stick with high-level API since going low-level is error-prone.

# Hints in gnark, a slightly advanced concept

"Hint" is a novel terminology that does not appear in Circom, but similar pattern exists in Circom.

What is a hint? Take an example from [SoK: What Don’t We Know? Understanding Security Vulnerabilities in SNARKs](https://arxiv.org/pdf/2402.15293) page 3 on the right, suppose we want to write a circuit that verifies $$X \neq 0$$. One way to do it is to use Fermat's little theorem and test if $$X ^ {p-1} = 1 \mod p$$, but that computation is expensive. Another way is let prover provide a hint $$H = X^{-1}$$ and test $$X * H = 1$$, and this works because non-zero elements over a field (which includes finite fields) always have multiplicative inverse by definition.

**In other words, "hint" means I can't compute the thing / I don't want to compute the thing since it is expensive, so I let you provide a "helper" to help me do the computation.**

In Circom, we can see similar pattern in the [IsZero template](https://github.com/iden3/circomlib/blob/cff5ab6288b55ef23602221694a6a38a0239dcc0/circuits/comparators.circom#L24-L34):

```javascript
template IsZero() {
    signal input in;
    signal output out;

    signal inv;

    inv <-- in!=0 ? 1/in : 0;

    out <== -in*inv +1;
    in*out === 0;
}
```

You can call this `inv` intermediate signal a hint following the definition in the paper. The computation of `inv` utilizes `<--`, which does not add any constraint into R1CS but allows more degrees of freedom in computation, such as `1/in` and the `<bool>?<expr>:<expr>` ternary operator.

The paper also provided an example in gnark:

```go
type Circuit struct {
    X frontend.Variable `gnark:",private"`
    C frontend.Variable `gnark:",public"`
    Y frontend.Variable `gnark:",public"`
}

func (circuit *Circuit) Define(api frontend.API) error {
    outputs := api.Compiler().NewHint(hint.SqrtHint, 1, circuit.X)
    squareRoot := outputs[0]
    api.AssertIsEqual(api.Mul(squareRoot, squareRoot), circuit.X)
    result := api.Mul(squareRoot, circuit.C)
    api.AssertIsEqual(circuit.Y, result)
    return nil
}
```

In this example, square root of $$X$$ is provided as a hint since computing it in a circuit is tough. The circuit verifies if $$Y = C \cdot sqrt(X)$$. Note that you must add a constraint testing hint variables: `api.AssertIsEqual(api.Mul(squareRoot, squareRoot), circuit.X)`, otherwise it will be an underconstraint bug.

There is also an example in [gnark doc](https://docs.gnark.consensys.io/HowTo/write/hints):

```go
    var b []frontend.Variable
    var Σbi frontend.Variable
    base := 1
    for i := 0; i < nBits; i++ {
        b[i] = cs.NewHint(hint.IthBit, a, i)
        cs.AssertIsBoolean(b[i])
        Σbi = api.Add(Σbi, api.Mul(b[i], base))
        base = base << 1
    }
    cs.AssertIsEqual(Σbi, a)
```

This circuit takes ith bit as hint since gnark does not support binary decomposition. The correctness of these hint variables is checked by `Σbi = api.Add(Σbi, api.Mul(b[i], base))`, which computes the weighted sum of all binary digits and compares it to the original decimal value. Without this set of constraints, it will be an underconstraint bug.

**In conclusion, from an engineering persepctive, you can think of compiler hints in gnark as variables computed by a hint functions off-circuit. They are just here to help you do some computation, and they are provided by gnark instead of the prover. And always remember to add constraints to hint variables!!!**

# Conclusion

We covered basic usage of snark here. In future blogposts, I will dig deeper into [official gnark code examples](https://github.com/Consensys/gnark/tree/master/examples) and how to reproduce gnark bugs. Hope you enjoyed the article!

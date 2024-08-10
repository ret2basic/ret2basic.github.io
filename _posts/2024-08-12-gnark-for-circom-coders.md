---
layout: post
title: gnark for Circom coders
---

{{ page.title }}
================

<p class="meta">August 12th, 2024</p>

# Why writing this blogpost

There aren't many articles on the Internet on how to write gnark code. During my research, I found the following resources:

- [gnark doc](https://docs.gnark.consensys.io/), of course.
- ["How to use the Consenys's Gnark Zero Knowledge Proof library and disclosure of a DoS bug" by LambdaClass](https://blog.lambdaclass.com/how-to-use-the-consenyss-gnark-zero-knowledge-proof-library-and-disclosure-of-a-ddos-bug/) -> this article describes how to work with low-level gnark API, which isn't covered in official gnark doc.
- There are a few code examples in [gnark Github repo](https://github.com/Consensys/gnark/tree/master/examples), but there is no explanation besides comments.

The target audience is those who already know Circom very well but don't have experience with gnark. These two DSLs are in fact very similar but gnark is more high-level than Circom. Personally I think Circom is similar to x86 assembly and gnark is similar to C, loosely.

I will try to map terminologies in one-to-one correspondence and introduce some gnark features that Circom does not have.

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
-- witness.json --
{
    "x": 3,
    "Y": 35
}

```

Similar to Circom, if a variable is not declared public, then it is private by default, such as `X`. The `gnark:"x"` and `gnark:",public"` part is called a **"tag"**, it adds metadata to fields in a struct.

### How to write constraints in gnark (using high-level API)

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

You can find high-level API methods here: https://github.com/Consensys/gnark/blob/master/frontend/api.go. Cryptography functions can be found here: https://github.com/ConsenSys/gnark-crypto. Unfortunately, there is no easy way to reference these methods, you have to look through the resource code. But at least you know where to look at: if you encounter unfamiliar function names, it must reside in one of those two repos.

### How to use low-level API in gnark

The [LambdaClass article](https://blog.lambdaclass.com/how-to-use-the-consenyss-gnark-zero-knowledge-proof-library-and-disclosure-of-a-ddos-bug/) discussed how to use low-level API in gnark. This info is rarely discussed on the Internet and I found the article helpful.





# Official gnark code examples

## Example 1: MiMC









## Example 2: Plonk





## Example 3: Rollup






# Coercions from numbers #

This document is not about coercions in general -- see [section 10.6 of TPIL](https://leanprover.github.io/theorem_proving_in_lean/type_classes.html#coercions-using-type-classes) for a general overview. This is an overview of how to work with the coercions `ℕ → ℤ → ℚ → ℝ → ℂ` (maps which mathematicians fondly call "the identity function", and which computer scientists call `↑`) and also the natural coercions from `ℤ` to a general ring and so on.

In brief: this document might help if you have three integers `x y z`, a proof that `x * y = z`, and your goal is `↑x * ↑y = ↑z`, something which you suspect is a statement about real numbers.

# The problems people have with coercions.

Here are two types of problems that people run into with coercions.

1) They are faced with a goal which is "obvious in maths", e.g.

```
a b : ℤ
⊢ ↑(a + b) = ↑a + ↑b
```

with the goal being an equality of real numbers.

2) They are faced with a goal which is "the same as a hypothesis", or "the same as something they know how to prove", e.g.

```
a b c : ℤ,
H : a + b * c = 12
⊢ ↑a + ↑b * ↑c = 12
```

These two problems are of a slightly different nature, and require two different solutions. In the next few sections I explain what I hope are enough tricks to make solving questions like this easy. But first here's a warning.

# A subtlety with automatic coercions

Before we go on, let me get this potentially confusing issue out of the way. Because there's a coercion from `ℤ` to `ℝ` one can just write `a : ℝ` if `a` is an integer, and Lean knows you mean the corresponding real number. Watch out for this gotcha though:

```lean
variables (a b : ℤ) 

#check ((a + b) : ℝ) -- ↑a + ↑b : ℝ
#check ((a + b : ℤ) : ℝ) -- ↑(a + b) : ℝ
```

I was initially surprised that these two terms didn't evaluate to exactly the same thing. What is happening here is that Lean figures out what is going on from the outside in. So for the first term, Lean knows that the type of `a + b` is supposed to be `ℝ`, so it then decides that the addition we want must be `real.add`, so it then decides that we must want `a` and `b` to be reals, giving us `↑a + ↑b`. In the second term, we explicitly say that we want `a + b` to be treated like an integer, and so Lean uses integer addition.

### "Obvious in maths" goals.

These are goals where there is no extra hypothesis or implication sign needed, they are just goals which are of the form `X = X'` where `X` and `X'` are "obviously equal" modulo the fact that calculations like additions might be taking place in different types. Goals like this should all be provable with `simp`. Examples:

```lean
import data.complex.basic

example (a b : ℤ) : ((a + b : ℤ) : ℝ) = a + b :=
-- ↑(a + b) = ↑a + ↑b
by simp

example (a : ℤ) (b : ℕ) (c : ℚ) (d : ℝ) :
(a : ℂ) + b * c - d = (((a + ((b * c) : ℚ) : ℚ) - d) : ℝ) :=
-- ⊢ ↑a + ↑b * ↑c - ↑d = ↑(↑(↑a + ↑b * c) - d)
by simp
```

What is going on behind the scenes is that there are a whole bunch of lemmas of the form "coercing two numbers from X to Y and then adding, equals adding and then coercing", or "if a and b have type X and `↑` is the coercion to `Y` then `a < b` iff `↑a < ↑b`". These lemmas do not always have the most intuitive names, but they are all tagged with `simp`, so hopefully in most cases end users do not need to know their names. I will say more about their names later.

But remember -- subtraction on naturals and division on naturals/integers are not what you expect if you're a mathematician. Lean can prove `(2 : ℕ) - (7 : ℕ) = (0 : ℕ)`, and `(5 : ℤ) / (2 : ℤ) = (2 : ℤ)` ("rounding" is occurring here), so `simp` fails here because the goals are actually false:

```lean
example (a b : ℕ) : ((a - b : ℕ) : ℤ) = a - b := by simp -- fails because false!
example (a b : ℕ) : ((a / b : ℕ) : ℚ) = a / b := by simp -- fails because false!
example (a b : ℤ) : ((a / b : ℤ) : ℂ) = a - b := by simp -- fails because false!
```

### "I have a hypothesis which says this already" goals

If the hypothesis is in the "smaller" set of numbers, and the conclusion in the larger set (e.g. you have a hypothesis involving an equality of rationals and a conclusion about equality of complex numbers), then this shouldn't be too hard. Here's an example.

```
example (a b c : ℤ) (H : a + b * c = 12) : (a : ℝ) + b * c = 12 :=
/-
a b c : ℤ,
H : a + b * c = 12
⊢ ↑a + ↑b * ↑c = 12
-/
begin
  have H2 : (a : ℝ) + b * c = ((a + b * c : ℤ) : ℝ) := by simp,
  rw H2, -- ⊢ ↑(a + b * c) = 12
  rw H, -- ⊢ ↑12 = 12
  simp,
end
```

In this example, a hypothesis says that `a + b * c = 12` (this is a statement about integers), and the goal is to prove a version of this for real numbers. Unfortunately the goal is not `↑(a + b * c) = 12`; instead, each of `a`, `b` and `c` are being coerced individually. However `↑a + ↑b * ↑c = ↑(a + b * c)` is exactly a "trivial in maths" goal which `simp` can prove (as we saw in the previous section), and then one can rewrite this and *then* rewrite `H` and get the goal into a form that `simp` can deal with.

Unfortunately, things can be harder when one is going the other way around.

```lean
example (a b c : ℤ) (H : (a : ℝ) + b * c = 12) : 
a + b * c = 12 :=
/-
a b c : ℤ,
H : ↑a + ↑b * ↑c = 12
⊢ a + b * c = 12
-/
begin
  have H2 : (a : ℝ) + b * c = ((a + b * c : ℤ) : ℝ) := by simp,
  -- now for the trick
  suffices H3 : (((a + b * c) : ℤ) : ℝ) = (12 : ℤ),
    exact int.cast_inj.1 H3,
    -- rwa int.cast_inj at H3, -- also works
  -- now the same as before
  rw ←H2,
  rw H,
  simp
end
```

Some magic happened here -- we had to invoke an explicit function `int.cast_inj.1` to show that if the real numbers corresponding to two integers were the same, then the integers were also the same. At the time of writing, the tactic which would do this automatically is just on a wishlist of future tactics. Until then, here are the ten explicit functions which you need to know, which prove all ten cases of the problem.

```lean
import data.complex.basic

example (q : ℕ) : (q : ℤ) = (3 : ℕ) → q = 3 := int.of_nat_inj
example (q : ℕ) : (q : ℚ) = (3 : ℕ) → q = 3 := nat.cast_inj.1
example (q : ℕ) : (q : ℝ) = (3 : ℕ) → q = 3 := nat.cast_inj.1
example (q : ℕ) : (q : ℂ) = (3 : ℕ) → q = 3 := nat.cast_inj.1

example (q : ℤ) : (q : ℚ) = (3 : ℤ) → q = 3 := int.cast_inj.1
example (q : ℤ) : (q : ℝ) = (3 : ℤ) → q = 3 := int.cast_inj.1
example (q : ℤ) : (q : ℂ) = (3 : ℤ) → q = 3 := int.cast_inj.1

example (q : ℚ) : (q : ℝ) = (3 : ℚ) → q = 3 := rat.cast_inj.1
example (q : ℚ) : (q : ℂ) = (3 : ℚ) → q = 3 := rat.cast_inj.1

example (q : ℝ) : (q : ℂ) = (3 : ℝ) → q = 3 := complex.of_real_inj.1
```

Note however that the issue here is that we are dealing with numerals. With variables things are much better:

```lean
example (q r : ℕ) : (q : ℤ) = r → q = r := int.of_nat_inj -- missing simp lemma?
example (q r : ℕ) : (q : ℚ) = r → q = r := by simp
example (q r : ℕ) : (q : ℝ) = r → q = r := by simp
example (q r : ℕ) : (q : ℂ) = r → q = r := by simp

example (q r : ℤ) : (q : ℚ) = r → q = r := by simp
example (q r : ℤ) : (q : ℝ) = r → q = r := by simp
example (q r : ℤ) : (q : ℂ) = r → q = r := by simp

example (q r : ℚ) : (q : ℝ) = r → q = r := by simp
example (q r : ℚ) : (q : ℂ) = r → q = r := by simp

example (q r : ℝ) : (q : ℂ) = r → q = r := by simp
```

Hopefully these clues will be enough to get beginners through. Remember that numerals sometimes need coercing too, and  `↑7 = 7` might not be true by `refl` -- although it will be provable using `simp`.

# More information about what is going on.

The rest of this document gives more background about what is actually going on here. If anyone finds that they are having to read it without really wanting to, e.g. because they have a goal which the techniques above won't solve, they might want to let me (Kevin Buzzard) know so I can try to make the simple helpful bit better.

What structure do we need on a general type `α` in order to get a natural
map from `ℕ` to it? Well, if we know `α` has `zero`, `one` and `add` then this would be enough, as we see here:

`example (α : Type) [has_zero α] [has_one α] [has_add α] (n : ℕ) : α := n`

For example, if `α` is a semiring, ring or field, then there's a natural map from `ℕ` to `α`.

```lean
import data.real.basic

example (n : ℕ) : ℝ := n

example (α : Type) [semiring α] (n : ℕ) : α := n
```

Similarly, if `α` has additive inverses, e.g. `α` is a ring or a field, then there is a natural map from `ℤ` to `α`, and if `α` is a characteristic zero division ring (for example `ℝ` or `ℂ`) then there is a natural map from `ℚ` to `α`. There is as far as I know no analogous general construction giving maps from `ℝ` or `ℂ` to general algebraic structures, however there is a coercion from `ℝ` to `ℂ`:

```lean
import data.complex.basic

def from_R_to_C (r : ℝ) : ℂ := r

#print from_R_to_C 
-- def from_R_to_C : ℝ → ℂ := λ (r : ℝ), ↑r
```

Looking at the definition of the function, we see that Lean is using `↑` to mean "use a coercion".

# How do I know which coercion the arrow means?

If the tricks at the top work for you, you might not even have to worry about exactly what `↑a` means. But if you do not understand what your goal says, because you have lost track of whether `↑a` means the rational number `a`, or the real number `a` or the complex number `a`, and you do need to know, then try writing `set_option pp.all true` before your theorem and then taking another look. For example, what was `↑(a + b) = ↑a + ↑b` above might now become around 25 lines of output, starting with

```
⊢ @eq.{1} real
    (@coe.{1 1} int real
       (@coe_to_lift.{1 1} int real
          (@coe_base.{1 1} int real
	  ...
```

and the very first line tells you that the goal is an equality between two real numbers (and the second line indicates that the left hand side of this equality is a coercion from the integers).

# "Special" coercions.

The coercion from `ℕ` to `ℤ` is not defined as the one coming from the fact that `ℤ` is a ring; it is more computationally efficient to use the constructor. The two functions are equal, but this is a theorem and not a definition. The coercion from `ℚ` to `ℝ` is noncomputable (although it doesn't have to be), and the coercion from `ℝ` to `ℂ` is defined by hand rather than being part of a general scheme of maps from `ℝ` to lots of places. 

# Examples of names.

If you do want to prove "trivial in maths" goals "by hand" for some reason, then you can write `set_option trace.simplify.rewrite true` before your theorem, see what `simp` is doing, and then mimic it.

```lean
import data.complex.basic

example (a b : ℤ) : ((a + b : ℤ) : ℝ) = a + b := int.cast_add a b

-- set_option trace.simplify.rewrite true
example (a : ℤ) (b : ℕ) (c : ℚ) (d : ℝ) :
(a : ℂ) + b * c - d = (((a + ((b * c) : ℚ) : ℚ) - d) : ℝ) :=
begin
  rw rat.cast_add,
  rw rat.cast_coe_int,
  rw complex.of_real_sub,
  rw complex.of_real_add,
  rw rat.cast_mul,
  rw complex.of_real_mul,
  rw rat.cast_coe_nat,
  rw complex.of_real_int_cast,
  rw complex.of_real_nat_cast,
  rw complex.of_real_rat_cast,
end
```

The second example shows the difficulties that you face when trying to do this by hand. Some of the lemmas are of this form:

```
rat.cast_add : ∀ (m n : ℚ), ↑(m + n) = ↑m + ↑n
```

This lemma is in the `rat` namespace, and it says that if you are coercing *from* `rat` to somewhere else, then adding and then coercing is the same as coercing and then adding.

But others are of this form:

```lean
complex.of_real_mul : ∀ (r s : ℝ), ↑(r * s) = ↑r * ↑s
```

This lemma is in the `complex` namespace, but it is about coercions *to* the complexes from the reals. Writing code by hand is full of pitfalls like this? Why is this? Is this just poor design? No, it's much more complicated than this. The problem is that the coercions from `nat` to `int`, and from `real` to `complex` are *not* defined as special cases of generic coercions from `nat` to a general semiring or from `real` to a general complete field -- mathlib's designers made a conscious decision to use other more computationally efficient coercions, hence the confusion. The designers are not bothered by this, because `simp` works!

Hopefully the example I gave above was sufficiently convoluted that if you really want to know the exact names of the lemmas you need, you can start to guess. Here's a brief selection. Because there are so many special cases, this can be a bit of a minefield.


coercions commute with multiplication:

```
nat.cast_mul -- ∀ (m n : ℕ), ↑(m * n) = ↑m * ↑n
int.cast_mul -- ∀ (m n : ℤ), ↑(m * n) = ↑m * ↑n
rat.cast_mul -- ∀ {α : Type u_1} (m n : ℚ), ↑(m * n) = ↑m * ↑n
```

coercions commute with inequalities.

```
nat.cast_le -- ∀ {m n : ℕ}, ↑m ≤ ↑n ↔ m ≤ n
```

etc etc

coercion from int to rat to X equals coercion from int to X [in rat namespace]

```
rat.cast_coe_int : ∀ (n : ℤ), ↑↑n = ↑n
```

coercion from int to real to complex equals coercion from int to complex [in complex namespace]

```
complex.of_real_int_cast : ∀ (n : ℤ), ↑↑n = ↑n
```
special coercions commute with multiplication:

```
complex.of_real_mul : ∀ (r s : ℝ), ↑(r * s) = ↑r * ↑s [a theorem about complexes]
int.coe_nat_mul : ∀ (m n : ℕ), ↑(m * n) = ↑m * ↑n [a theorem about integers]
```

Note that these are in the namespace of the target type not the source.

# Zero Cost Abstractions

### Eliminating runtime overhead in Scala

http://caryrobbins.com/zero-talk

Cary Robbins

June 5, 2019

---

## Me

* Software development consultant for Estatico Studios <!-- .element: class="fragment" -->
* Primarily work in Scala and Haskell, but love digging into new languages. <!-- .element: class="fragment" -->
* Interested in collaborating? Send me a note at: cary@estatico.io <!-- .element: class="fragment" -->

---

## Zero Cost Strategies

* Polymorphic values    <!-- .element: class="fragment" -->
* NewTypes              <!-- .element: class="fragment" -->
* Generic programming   <!-- .element: class="fragment" -->

Note:
We're going to discuss 3 different strategies for improving
existing abstractions with implementations that do not incur
runtime overhead.

---

# WARNING

<p>We will be using</p>

<p class=fragment style="
  font-family:monospace;
  font-weight:bold;
  font-size:100px;
  color:red;
">asInstanceOf</span>

<small class=fragment>a lot.</small>

<small><small class=fragment>It's cool don't worry.</small></small>

---

## But runtime overhead isn't my bottleneck!

Note:

Sure, maybe it's not your bottleneck. Your bottleneck is
probably IO. But this doesn't mean we shouldn't discover ways
of optimizing our compiled code to eliminate overhead which
is truly unnecessary.

---

## But muh Premature Optimization is evil!

Note:

Yes, this is true.
Don't optimize for the sake of optimizing,
and you should benchmark if you are doing heavy optimizations.

But what we're going to cover here are general optimizations
which should not only improve runtime performance, but actually
give you new ways of programming that you may not be aware of.

---

## We're talking JVM

<small class=fragment>(Although, many of these work just as good if not better in ScalaJS)</small>

---

## Polymorphic values

<div class=fragment>
```scala
implicit def eitherFunctor[L]: Functor[Either[L, ?]] = new Functor[Either[L, ?]] {
  override def map[A, B](fa: Either[L, A])(f: A => B): Either[L, B] =
    fa.right.map(f)
}
```

<div class=fragment>
**Problem is...**
```scala
<frag>implicitly[Functor[Either[String, ?]]]</frag>
<frag>// $anon$1@2cf46f9e</frag>

<frag>implicitly[Functor[Either[String, ?]]]</frag>
<frag>// $anon$1@7cd0fda6</frag><frag> ← new instance!</frag>
```

Note:
Here's an example of a polymorphic type class instance, the Functor instance
for Either.

Functor's type argument must be a type which itself takes a single type argument.
Either takes two, so we must pass the Left type argument into the implicit def.

Problem is...each time we summon the implicit instance, we alloc a new instance.

---

## Caching Polymorphic Values

```scala
private val eitherFunctorCached = new Functor[Either[Nothing, ?]] {
  override def map[A, B](fa: Either[Nothing, A])(f: A => B): Either[Nothing, B] =
    fa.right.map(f)
}

implicit def eitherFunctor[L]: Functor[Either[L, ?]] =
  eitherFunctorCached.asInstanceOf[Functor[Either[L, ?]]]
```

<div class=fragment>
**As you'd expect...**
```scala
<frag>implicitly[Functor[Either[String, ?]]]</frag>
<frag>// $anon$1@3c466028</frag>

<frag>implicitly[Functor[Either[String, ?]]]</frag>
<frag>// $anon$1@3c466028</frag><frag> ← same instance!</frag>
```


Note:
Here we implement the same Functor instance, except this time instead of making
it polymorphic, we stub in the Nothing type for the Left type param so we can cache
it into a val. We then have the implicit def simply call the val and cast it to
the appropriate type.

---

## Why does this work?

Type parameters are erased at runtime

```scala
implicit def eitherFunctor[L]: Functor[Either[L, ?]] = new Functor[Either[L, ?]] {
  override def map[A, B](fa: Either[L, A])(f: A => B): Either[L, B] =
    fa.right.map(f)
}
```

<div class=fragment>
```java
// Bytecode
public eitherFunctor()LFunctor;
  NEW $anon$1                      // new anon Functor instance
  DUP
  INVOKESPECIAL $anon$1.<init> ()V // Functor constructor "init"
  ARETURN
```

Note:

Why does this work? Type parameters are erased at runtime.

---

## Why does this work?

```scala
private val eitherFunctorCached = new Functor[Either[Nothing, ?]] {
  override def map[A, B](fa: Either[Nothing, A])(f: A => B): Either[Nothing, B] =
    fa.right.map(f)
}

implicit def eitherFunctor[L]: Functor[Either[L, ?]] =
  eitherFunctorCached.asInstanceOf[Functor[Either[L, ?]]]
```

<div class=fragment>
```java
// Bytecode
public eitherFunctor()LFunctor; <frag>// ← Runtime type is still Functor</frag>
  ALOAD 0
  INVOKESPECIAL eitherFunctorCached ()LFunctor;
  ARETURN            <frag>// Same runtime type ↑</frag>
```

---

## The @cached macro

```scala
@cached
implicit def eitherFunctor[L]: Functor[Either[L, ?]] = new Functor[Either[L, ?]] {
  override def map[A, B](fa: Either[L, A])(f: A => B): Either[L, B] =
    fa.right.map(f)
}
```

<div class=fragment>
Expands to...

<div class=fragment>
```scala
implicit def eitherFunctor[L]: Functor[Either[L, ?]] =
  __cached__eitherFunctor.asInstanceOf[Functor[Either[L, ?]]]

private val __cached__eitherFunctor = new Functor[Either[Nothing, ?]] {
  override def map[A, B](fa: Either[Nothing, A])(f: A => B): Either[Nothing, B] =
    fa.right.map(f)
}
```

---

## newtypes

---

## NewTypes: A Motivation

Haskell's `newtype` gives us type safety without the cost

```haskell
newtype WidgetId = WidgetId String

<frag>myId = WidgetId "a" </frag><frag> -- ← Is just a String at runtime</frag>

<frag>upperId :: WidgetId -> WidgetId
upperId (WidgetId s) = WidgetId (map toUpper s)</frag>
<frag>-- The unwrap ↑ and rewrap ↑ do not occur at runtime</frag>
```

Note:
In Haskell, the `newtype` keyword allows us to define a new type
which incurs no overhead and is just some other type at runtime.

In this case, we're defining a `WidgetId` to improve type safety
so we know that this value is more than just a `String`.

Wrapping and unwrapping newtype values incurs no overhead at runtime; it is
all omitted during compilation.

---

## NewTypes: Specialized type class instances

```haskell
instance ToJSON WidgetId where
  toJSON (WidgetId s) = object [ "widgetId" .= toJSON s ]
```

<div class=fragment>
```haskell
newtype Sum = Sum Int

instance Monoid Sum where
  mempty = 0
  mappend (Sum x) (Sum y) = Sum (x + y)

newtype Product = Product Int

instance Monoid Product where
  mempty = 1
  mappend (Product x) (Product y) = Product (x * y)
```

Note:
Another common use case for newtypes is to give us the ability
to provide alternative type class implementations.

Here, instead of using String's default ToJSON instance, we can provide
our own implementation for our WidgetId.

The canonical example of this is that Int forms a Monoid via
addition _and_ multiplication. Instead of having to decide which
instance we want Int to use, we can say that Int is not a Monoid
on its own, and that only Sum and Product (which are newtype wrappers
around Int) are.

---

## NewTypes: O(n) ⇒ O(1) conversions

```haskell
ids :: [String]
ids = ["a","b","c","d","e"]

xs :: [WidgetId]
xs = coerce ids :: [WidgetId]
```

---

## Value Classes: NewTypes for Scala?

```scala
final case class WidgetId(value: String) <frag>extends AnyVal</frag>
```

<div class=fragment>
Usage
```scala
def getWidget(id: WidgetId): Widget = ???
```

<div class=fragment>
```java
// Bytecode
public getWidget(Ljava/lang/String;)LWidget;
```

Note:
Here is a value class. This allows us to define a new class which,
at runtime, should be represented as a String.
We can confirm this by looking at the compiled bytecode.

---

## Value Class Pitfalls: Generics

```scala
def optId = Option(WidgetId("a"))
```

<div class=fragment>
```java
// Bytecode
public optId()Lscala/Option;
  GETSTATIC scala/Option$.MODULE$ : Lscala/Option$;
  <mark>NEW WidgetId</mark><frag> // ← We don't like this</frag>
  DUP
  LDC "a"
  INVOKESPECIAL WidgetId.<init> (Ljava/lang/String;)V
  INVOKEVIRTUAL scala/Option$.apply (Ljava/lang/Object;)Lscala/Option;
  ARETURN
```

Note:
Let's see what's _wrong_ with value classes. One of the big problems with them
is when you try to use them with generics.

Also remember that since generics are erased at runtime,
this method returns Option, not Option of String or WidgetId.

---

## Value Class Pitfalls: Generics

```scala
def unOptId = optId.get
```

<div class=fragment>
```java
// Bytecode
public unOptId()Ljava/lang/String;
  ALOAD 0
  INVOKEVIRTUAL optId ()Lscala/Option;
  INVOKEVIRTUAL scala/Option.get ()Ljava/lang/Object;
  <mark>CHECKCAST WidgetId</mark> <frag>// ← Why the allocation occurs</frag>
  <mark>INVOKEVIRTUAL WidgetId.value ()Ljava/lang/String;</mark> <frag> // ← Unwrap</frag><frag> ლ(ಠ益ಠლ)</frag>
  ARETURN
```

Note:
* The allocation must occur so the JVM can validate that we actually have
  a WidgetId at runtime.
* Interestingly enough, we then just unwrap the value we had, getting back the
  String we wrapped to begin with.

---

## Value Class Pitfalls: O(n) conversions

```scala
def convIdList(xs: List[String]): List[WidgetId] =
  xs.map(WidgetId.apply) <frag>// ← Hey JIT, try to optimize this</frag>
```

Note:
Any time we need to convert container elements to value class instances,
we'll always incur a performance overhead; this just can't be optimized
away, not even by the JIT.

---

## Tagged Types

```scala
// Adapted from scalaz
type @@[A, T] = { type Tag = T; type Self = A }
def tag[A, T](a: A): A @@ T = a.asInstanceOf[A @@ T]

<frag>trait WidgetIdTag</frag>
<frag>type WidgetId = String @@ WidgetIdTag</frag>
<frag>def WidgetId(s: String): WidgetId = tag[String, WidgetIdTag](s)</frag>
```

Note:
Scalaz and shapeless provide something called `tagged types`.
Note that Scalaz's and shapeless' implementation of tagged types
are different, but we'll cover the difference later.

A `tagged type` will be defined with this `@@` type operator. We'll
use a refinement to hold onto the types passed in but not expose it
so the compiler sees this as a unique type. This refinement will
be represented as a java Object at runtime.

To construct one, we'll just perform a cast using `.asInstanceOf`.
This ends up being fine since any value we pass in will be an
instance of `Object`.

---

## Tagged Types: Under the hood

```scala
def tag[A, T](a: A): A @@ T = a.asInstanceOf[A @@ T]
```

<div class=fragment>
```java
// Bytecode
public tag(Ljava/lang/Object;)Ljava/lang/Object;
  ALOAD 1
  ARETURN
```

<div class=fragment>
```scala
def WidgetId(s: String): WidgetId = tag[String, WidgetIdTag](s)
```

<div class=fragment>
```java
// Bytecode
public WidgetId(Ljava/lang/String;)Ljava/lang/Object;
  ALOAD 0
  ALOAD 1
  INVOKEVIRTUAL tag (Ljava/lang/Object;)Ljava/lang/Object;
  ARETURN
```

---

## Tagged Types: Under the hood

```scala
def optId = Option(WidgetId("a"))
```

<div class=fragment>
```java
// Bytecode
public optId()Lscala/Option;
  GETSTATIC scala/Option$.MODULE$ : Lscala/Option$;
  ALOAD 0
  LDC "a"
  INVOKEVIRTUAL WidgetId (Ljava/lang/String;)Ljava/lang/Object;
  INVOKEVIRTUAL scala/Option$.apply (Ljava/lang/Object;)Lscala/Option;
  ARETURN
```

---

## Tagged Types: Under the hood

```scala
def unOptId = optId.get
```

<div class=fragment>
```java
// Bytecode
public unOptId()Ljava/lang/Object;
  ALOAD 0
  INVOKEVIRTUAL optId ()Lscala/Option;
  INVOKEVIRTUAL scala/Option.get ()Ljava/lang/Object;
  ARETURN
```

---

## Tagged Types: Faster conversions

```scala
def convIdList(ids: List[String]): List[WidgetId] =
  ids.asInstanceOf[List[WidgetId]]
```

<div class=fragment>
```java
// Bytecode
public convIdList(Lscala/collection/immutable/List;)Lscala/collection/immutable/List;
  ALOAD 1
  ARETURN
```

Note:
So far this isn't looking too bad. These are all constant operations
that are really easy for the JIT to inline. We have no `new` allocations
and no `checkcasts`.

---

## Tagged Types: Things left to be desired

* How do we deal with type class instances/implicits? <!-- .element: class="fragment" -->
* How do we add methods? <!-- .element: class="fragment" -->
* Can we make casting safer? <!-- .element: class="fragment" -->
* Can we make it less ad hoc and reduce boilerplate? <!-- .element: class="fragment" -->

---

## NewType: Formalizing the approach

```scala
<frag 4>type WidgetId = WidgetId.Type</frag>
object WidgetId {
  <frag 1>type Base = Any { type WidgetId$newtype }</frag>
  <frag 2>trait Tag extends Any</frag>
  <frag 3>type Type <: Base with Tag</frag>

  <frag 5>def apply(value: String): WidgetId = value.asInstanceOf[WidgetId]</frag>

  <frag 6>implicit final class Ops(private val me: WidgetId) extends AnyVal {
    def value: String = me.asInstanceOf[String]
  }</frag>
}
```

Note:

Let's start with a companion object. This should be where implicits are defined for our
newtype.

`Base` -
* A unique type that gets compiled as Object in the bytecode
* Any
    * Signals we want Object
    * Aid in array construction for primitives
* Refinement
    * Ensures that Any is actually used as the base type instead of trait mixins

`Tag`
  * A unique trait that we'll mix into our newtype to aid in implicit resolution.
  * Similarly as with `Base` above it, we need to extend `Any` to help with primitives.

`Type` -
* This is our final newtype
* Defined as an abstract type to prevent scalac from expanding the type alias
  which would break implicit resolution

`WidigetId` - Top-level type alias for convenience and simplicity

`apply` - Smart constructor for our newtype

`Ops` - Extension methods for our newtype

---

## NewType: Type semantics

```scala
WidgetId("a").<frag><mark>toUpperCase</mark></frag> <frag>// Does not compile</frag>
<frag>WidgetId("a").value.toUpperCase</frag> <frag>// Compiles</frag>

<frag>def upperStr(s: String) = s.toUpperCase</frag>
<frag>upperStr("a")</frag> <frag>// Compiles</frag>
<frag><mark>upperStr</mark>(WidgetId("a"))</frag> <frag>// Does not compile</frag>

<frag>def upperWidgetId(id: WidgetId) = WidgetId(id.value.toUpperCase)</frag>
<frag><mark>upperWidgetId</mark>("a")</frag> <frag>// Does not compile</frag>
<frag>upperWidgetId(WidgetId("a"))</frag> <frag>// Compiles</frag>

```

---

## NewType: Builder trait

```scala
trait NewType[Repr] {
  <frag 10>type Base = Any { type Repr$newtype = Repr }</frag>
  <frag 20>trait Tag extends Any</frag>
  <frag 30>type Type <: Base with Tag</frag>
  <frag 40>def apply(value: Repr): Type = value.asInstanceOf[Type]</frag>
  <frag 50>def repr(x: Type): Repr = x.asInstanceOf[Repr]</frag>
}
```

<div class=fragment data-fragment-index=70>
Usage
```scala
<frag 80>type WidgetId = WidgetId.Type</frag>
object WidgetId extends NewType[String] <frag 90>{
  implicit final class Ops(private val me: WidgetId) extends AnyVal {
    def value: Repr = repr(me)
  }
}</frag>
```

Note:

So what does this buy us?

We get a smart constructor and destructure for free.
We get safe casts for collection types.
We get an idiomatic companion object that **just works** for implicit resolution.

---

## NewType: At runtime

```java
public apply(Ljava/lang/Object;)Ljava/lang/Object;
  ALOAD 0
  ALOAD 1
  INVOKESTATIC <mark green>NewType.apply$</mark> (LNewType;Ljava/lang/Object;)Ljava/lang/Object;
  ARETURN

<frag>public static synthetic apply$(LNewType;Ljava/lang/Object;)Ljava/lang/Object;
  ALOAD 0
  ALOAD 1
  INVOKESPECIAL <mark green>NewType.apply</mark> (Ljava/lang/Object;)Ljava/lang/Object;
  ARETURN</frag>

<frag>public default apply(Ljava/lang/Object;)Ljava/lang/Object;
  ALOAD 1
  ARETURN</frag>
```

---

## NewType: Extension Methods

```scala
def widgetIdValue = WidgetId("a").value
```

<div class=fragment>
```java
// Bytecode
public widgetIdValue()Ljava/lang/String;
  GETSTATIC WidgetId$.MODULE$ : LWidgetId$;
  GETSTATIC WidgetId$.MODULE$ : LWidgetId$;
  LDC "a"
  INVOKEVIRTUAL WidgetId$.apply (Ljava/lang/Object;)Ljava/lang/Object;
  INVOKEVIRTUAL WidgetId$.Ops (Ljava/lang/Object;)Ljava/lang/Object;
  INVOKEVIRTUAL <mark green>WidgetId$Ops$.value$extension</mark> (Ljava/lang/Object;)Ljava/lang/String;
```

---

## NewType: Extension Methods

```scala
implicit final class Ops(private val self: Type) extends AnyVal {
  def value: Repr = repr(self)
}
```

<div class=fragment>
```java
// Bytecode
public final value$extension(Ljava/lang/Object;)Ljava/lang/String;
  GETSTATIC WidgetId$.MODULE$ : LWidgetId$;
  ALOAD 1
  INVOKEVIRTUAL WidgetId$.repr (Ljava/lang/Object;)Ljava/lang/Object;
  CHECKCAST java/lang/String
  ARETURN
```

---

## NewType: Generics

```scala
def optId = Option(WidgetId("a"))
```

<div class=fragment>
```java
// Bytecode
public optId()Lscala/Option;
  GETSTATIC scala/Option$.MODULE$ : Lscala/Option$;
  GETSTATIC WidgetId$.MODULE$ : LWidgetId$;
  LDC "a"
  INVOKEVIRTUAL WidgetId$.apply (Ljava/lang/Object;)Ljava/lang/Object;
  INVOKEVIRTUAL scala/Option$.apply (Ljava/lang/Object;)Lscala/Option;
  ARETURN
```

---

## NewType: Generics

```scala
def unOptId = optId.get
```

<div class=fragment>
```java
public unOptId()Ljava/lang/Object;
  ALOAD 0
  INVOKEVIRTUAL optId ()Lscala/Option;
  INVOKEVIRTUAL scala/Option.get ()Ljava/lang/Object;
  ARETURN
```

---

## NewType: Constant-time conversions

```scala
def convIdList(ids: List[String]): List[WidgetId] = ids.asInstanceOf[List[WidgetId]]
```

<div class=fragment>
```java
// Bytecode
public convIdList(Lscala/collection/immutable/List;)Lscala/collection/immutable/List;
  ALOAD 1
  ARETURN
```

---

## NewTypes: Primitive Problems

```scala
type Feet = Feet.Type
object Feet extends NewType.Of[Int]
```

<div class=fragment>
```scala
def mkFeet = Feet(12)
```

<div class=fragment>
```java
// Bytecode
public mkFeet()Ljava/lang/Object;
  GETSTATIC Feet$.MODULE$ : LFeet$;
  BIPUSH 12
  INVOKESTATIC scala/runtime/BoxesRunTime.<mark>boxToInteger</mark> (I)Ljava/lang/Integer;
  INVOKEVIRTUAL Feet$.apply (Ljava/lang/Object;)Ljava/lang/Object;
  ARETURN
```

---

## NewSubType: An Attempt

```scala
<frag 20>trait NewSubType[Repr] {
  <frag 50>trait Tag extends Any</frag>
  <frag 60>type Type <: Repr with Tag</frag>

  <frag 70>def apply(x: Repr): Type = x.asInstanceOf[Type]</frag>
  <frag 80>def repr(x: Type): Repr = x.asInstanceOf[Repr]</frag>
}</frag>
```

---

## NewSubType


```scala
<frag>type Feet = Feet.Type</frag>
<frag>object Feet extends NewSubType[Int] {
  <frag>implicit final class Ops(private val me: Feet) extends AnyVal {
    def value: Int = repr(me)
  }</frag>
}</frag>
```

<div class=fragment>
```scala
Feet(1)<frag>.signum</frag> <frag>// Compiles, returns Int</frag>

<frag>def add(x: Int, y: Int): Int = x + y</frag>
<frag>add(Feet(1), Feet(2))</frag> <frag>// Compiles, returns Int</frag>

<frag>def addFeet(x: Feet, y: Feet): Feet = Feet(x.value + y.value)</frag>
<frag>addFeet(Feet(1), Feet(2))</frag> <frag>// Compiles, returns Feet</frag>
<frag><mark>addFeet</mark>(1, 2)</frag> <frag>// Does not compile</frag>
```


---

## NewSubType: Is it really zero cost?

<div class=fragment data-fragment-index=1>
```scala
def mkFeet = Feet(12)
```

<div class=fragment data-fragment-index=2>
```java
// Bytecode
public mkFeet()I
  GETSTATIC Feet$.MODULE$ : LFeet$;
  BIPUSH 12
  INVOKESTATIC scala/runtime/BoxesRunTime.<mark 3>boxToInteger</mark> (I)Ljava/lang/Integer;
  INVOKEVIRTUAL Feet$.apply <mark 6>(Ljava/lang/Object;)Ljava/lang/Object;</mark> <frag 7>// oh</frag>
  INVOKESTATIC scala/runtime/BoxesRunTime.<mark 4>unboxToInt</mark> (Ljava/lang/Object;)I <frag 5>// wat</frag>
  IRETURN
```

---

## NewSubType: Fixing our broken ctor

```scala
type Feet = Feet.Type
object Feet extends NewSubType[Int] {
  <frag>override def apply(x: Int): Feet = x.asInstanceOf[Feet]</frag>
}
```

<div class=fragment>
```java
public apply(I)I
  ILOAD 1
  <mark>INVOKESTATIC scala/runtime/BoxesRunTime.boxToInteger</mark> (I)Ljava/lang/Integer;
  <mark>INVOKESTATIC scala/runtime/BoxesRunTime.unboxToInt</mark> (Ljava/lang/Object;)I <frag>// srsly</frag>
  IRETURN
```

---

## NewSubType: Giving scalac a hand

```scala
type Feet = Feet.Type
object Feet extends NewSubType.Of[Int] {
  override def apply(x: Int): Feet = x.asInstanceOf[Feet]
}
```

<div class=fragment>
```bash
% scalac <mark green>-opt:l:inline</mark> ...
```

<div class=fragment>
```java
public apply(I)I
  ILOAD 1
  IRETURN <frag>// ...Did we just get it to work?</frag>
```

<div class=fragment>
```scala
def mkFeet = Feet(12)
```

<div class=fragment>
```java
public mkFeet()I
  GETSTATIC Feet$.MODULE$ : LFeet$;
  BIPUSH 12
  INVOKEVIRTUAL Feet$.apply (I)I
  IRETURN <frag>// ...We did!</frag>
```

---

## Coercible: Abstracting over newtypes

```scala
trait Coercible[A, B] {
  final def apply(a: A): B = a.asInstanceOf[B]
}
```

---

## Generated Coercible instances

```scala
trait NewType[Repr] {
  type Base = Any { type Repr$newtype = Repr }
  trait Tag extends Any
  type Type <: Base with Tag
  def apply(value: Repr): Type = value.asInstanceOf[Type]
  def repr(x: Type): Repr = x.asInstanceOf[Repr]

  <frag>implicit val coerceTo:   Coercible[Repr, Type] = Coercible.instance
  implicit val coerceFrom: Coercible[Type, Repr] = Coercible.instance</frag>
}
```

---

## Coercible in action

```scala

WidgetId("foo")<frag>.coerce[String]<frag>.toUpperCase</frag></frag>

<frag>def upper(s: String) = s.toUpperCase</frag>

<frag>upper(WidgetId("foo").coerce)</frag> <frag>// Compiles, returns String</frag>

<frag>implicit def coercibleListTo[A, B](
  implicit ev: Coercible[A, B]
): Coercible[List[A], List[B]] = Coercible.instance</frag>

<frag>List("a","b","c").coerce[List[WidgetId]]</frag>
```

---

## Aside on type roles

https://github.com/estatico/scala-newtype/issues/29

```scala
trait TypeRole[A] {
  type Role
}

object TypeRole {

  def mk[A, R]: TypeRole[A] { type Role = R } =
    _instance.asInstanceOf[TypeRole[A] { type Role = R }]

  private val _instance = new TypeRole[Nothing] {}

  type Nominal[A] = TypeRole[A] { type Role = types.Nominal }
  type Representational[A] = TypeRole[A] { type Role = types.Representational }

  object types {
    sealed trait Representational
    sealed trait Nominal
  }
}
```

---

## Shilling for @newtype

https://github.com/estatico/scala-newtype

```scala
import io.estatico.newtype.macros._

<frag>@newtype case class WidgetId(value: String)</frag>

<frag>@newsubtype case class Feet(value: Int)</frag>
```

---

## @newtype: inspecting generated code

```scala
@newtype(debug = true) case class WidgetId(value: String)
```

<div class=fragment>
```scala
type WidgetId = WidgetId.Type
object WidgetId {
  type Repr = String
  type Base = Any { type WidgetId$newtype }
  abstract trait Tag extends Any
  type Type <: Base with Tag

  def apply(value: String): WidgetId = value.asInstanceOf[WidgetId];

  def deriving[TC[_]](implicit ev: TC[Repr]): TC[Type] = ev.asInstanceOf[TC[Type]]

  // ...
}
```

---

## @newtype: deriving type class instances

```scala
@newtype case class Attributes(toList: List[(String, String)])
object Attributes {
  implicit val monoid: Monoid[Attributes] = <frag>deriving</frag>
}
```

---

## @newtype: derivingK

```scala
@newtype case class Slice[A](toVector: Vector[A])
object Slice {
  implicit val monad: Monad[Slice] = <frag>derivingK</frag>
}
```

---

## customized smart constructors

```scala
@newsubtype class PosInt(val toInt: Int)
<frag>object PosInt {
  def of(x: Int): Option[PosInt] =
    if (x < 0) None else Some(x.coerce[PosInt])
}</frag>
```

<div class=fragment>
```scala
PosInt(1) <frag>// Compile error: PosInt.type does not take parameters</frag>

<frag>PosInt.of(1)</frag>  <frag>// Some(1): Option[PosInt]</frag>
<frag>PosInt.of(-1)</frag> <frag>// None:    Option[PosInt]</frag>
```

---

## Unboxed Maybe type

```scala
@newtype class Maybe[A](val unsafeGet: A) {
  def isEmpty:   Boolean = unsafeGet == Maybe._empty
  def isDefined: Boolean = unsafeGet != Maybe._empty

  def map[B](f: A => B): Maybe[B] =
    if (isEmpty) Maybe.empty else Maybe(f(unsafeGet))

  def filter(p: A => Boolean): Maybe[A] =
    if (isEmpty || !p(unsafeGet)) Maybe.empty else this
  ...
}
```
```scala
object Maybe {
  def apply[A](a: A): Maybe[A] = if (a == null) empty else unsafe(a)
  def unsafe[A](a: A): Maybe[A] = a.asInstanceOf[Maybe[A]]
  def empty[A]: Maybe[A] = _empty.asInstanceOf[Maybe[A]]
  private val _empty = new Empty
  private final class Empty { override def toString = "Maybe.empty" }
}
```

---

## NewType Transformers

```scala
@newtype case class OptionT[F[_], A](value: F[Option[A]]) {

  def map[B](f: A => B)(implicit F: Functor[F]): OptionT[F, B] =
    OptionT(F.map(value)(_.map(f)))

  def flatMapF[B](f: A => F[Option[B]])(implicit F: Monad[F]): OptionT[F, B] =
    OptionT(F.flatMap(value)(_.fold(F.pure[Option[B]](None))(f)))

  def flatMap[B](f: A => OptionT[F, B])(implicit F: Monad[F]): OptionT[F, B] =
    flatMapF(a => f(a).value)

  ...
}
```

---

## Generic Programming: Shapeless

```scala
import shapeless._

case class Foo(a: Int, b: String)

<frag>val foo = Foo(1, "hey")
<frag>val g = Generic[Foo].to(foo)
<frag>// g: Int :: String :: HNil
<frag>//  = 1   :: "hey"  :: HNil

<frag>foo eq g
<frag>// false
```

---

## HList allocations

```scala
(1 :: "hey" :: HNil) == g
<frag>// true

<frag>new ::(1, new ::("hey", HNil)) == g
<frag>// true
```

---

## HList: An alternative

```scala
sealed trait HList
<frag 1>final case class ::[+H, +T <: HList](head : H, tail : T) extends HList
<frag 2>sealed trait HNil extends HList
```

<div class=fragment data-fragment-index=3>
Alternatively...
```scala
<frag 20>sealed trait GList</frag>
<frag 30>sealed trait #:[H, T <: GList] extends GList</frag>
<frag 40>sealed trait GNil extends GList</frag>
<frag 50>object GList {
  @newtype case class Of[A, L <: GList](value: A)
  <frag 70>object Of {</frag>           <frag 60>//  ^ phantom type</frag>
    <frag 70>// It's valid (and desirable) to coerce to the tail of our GList
    implicit def coerceToTail[A, H, T <: GList]: Coercible[Of[A, H #: T], Of[A, T]] =
        Coercible.instance
  }</frag>
}</frag>
```

---

## Generic Products

```scala
trait GProduct[A] { type Repr <: GList }

object GProduct {

  type Aux[A, R <: GList] = GProduct[A] { type Repr = R }

  def to[A](a: A)(implicit ev: GProduct[A]): GList.Of[A, ev.Repr] = GList.Of(a)
}
```

---

## Accessing Fields Generically

```scala
trait IsGCons[A] {
  <frag>type Head
  type Tail <: GList</frag>
  <frag>def head(a: GList.Of[A, Head #: Tail]): Head</frag>
  <frag>final def tail(a: GList.Of[A, Head #: Tail]): GList.Of[A, Tail] =
    a.coerce[GList.Of[A, Tail]]</frag>
}

<frag>object IsGCons {
  type Aux[A, H, T <: GList] = IsGCons[A] { type Head = H ; type Tail = T }
}</frag>
```

---

## Deriving GProduct and IsGCons

```scala
@DeriveGProduct case class Foo(a: String, b: Int, c: Float, d: Double)
```

<div class=fragment>
```scala
object Foo {
  implicit val isGCons4: IsGCons.Aux[Foo, String, Int #: Float #: Double #: GNil] =
    IsGCons.instance(_.a)
  <frag>implicit val isGCons3: IsGCons.Aux[Foo, Int, Float #: Double #: GNil] =
    IsGCons.instance(_.b)</frag>
  <frag>implicit val isGCons2: IsGCons.Aux[Foo, Float, Double #: GNil] =
    IsGCons.instance(_.c)</frag>
  <frag>implicit val isGCons1: IsGCons.Aux[Foo, Double, GNil] =
    IsGCons.instance(_.d)</frag>
  <frag>implicit def gProduct: GProduct.Aux[Foo, String #: Int #: Float #: Double #: GNil] =
    GProduct.instance</frag>
};
```

---

## Deriving new type class instances

```scala
trait CsvEncoder[A] {
  def encode(a: A): String
}
```

---

## Deriving new type class instances

```scala
implicit def gCons[A, H, T <: GList](
  implicit
  <frag>hEnc: CsvEncoder[H],</frag>
  <frag>tEnc: CsvEncoder[GList.Of[A, T]],</frag>
  <frag>isGCons: IsGCons.Aux[A, H, T]</frag>
): CsvEncoder[GList.Of[A, H #: T]] =
  <frag>CsvEncoder.instance(a => hEnc.encode(a.head) + ',' + tEnc.encode(a.tail))</frag>

<frag>implicit def gSingle[A, H](
  implicit
  <frag>hEnc: CsvEncoder[H],</frag>
  <frag>isGCons: IsGCons.Aux[A, H, GNil]</frag>
): CsvEncoder[GList.Of[A, H #: GNil]] =
  <frag>CsvEncoder.instance(a => hEnc.encode(a.head))</frag></frag>
```

---

## It works!

```scala
@DeriveGProduct case class Foo(a: String, b: Int, c: Float, d: Double)
object Foo {
  implicit val csvFoo = CsvEncoder[Foo] = CsvEncoder.derive[Foo]
}

<frag>CsvEncoder.encode(Foo("ya", 2, 8.2f, 7.6))</frag>
<frag>// "ya,2,8.2,7.6"</frag>
```

---

## Thank you!

* NewType Library - https://github.com/estatico/scala-newtype
* `@cached` Implementation - https://github.com/estatico/scala-cached
* GProduct Implementation - https://github.com/carymrobbins/scala-derivable

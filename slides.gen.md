<!--
****    THIS IS A GENERATED FILE, DO NOT EDIT!    ****
**** CHANGES SHOULD BE MADE IN slides.md INSTEAD! ****
-->
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
<pre><code data-noescape data-trim class=scala>
implicit def eitherFunctor[L]: Functor[Either[L, ?]] = new Functor[Either[L, ?]] {
  override def map[A, B](fa: Either[L, A])(f: A =&gt; B): Either[L, B] =
    fa.right.map(f)
}
</code></pre>

<div class=fragment>
**Problem is...**
<pre><code data-noescape data-trim class=scala>
<span class="fragment">implicitly[Functor[Either[String, ?]]]</span>
<span class="fragment">// $anon$1@2cf46f9e</span>

<span class="fragment">implicitly[Functor[Either[String, ?]]]</span>
<span class="fragment">// $anon$1@7cd0fda6</span><span class="fragment"> ← new instance!</span>
</code></pre>

Note:
Here's an example of a polymorphic type class instance, the Functor instance
for Either.

Functor's type argument must be a type which itself takes a single type argument.
Either takes two, so we must pass the Left type argument into the implicit def.

Problem is...each time we summon the implicit instance, we alloc a new instance.

---

## Caching Polymorphic Values

<pre><code data-noescape data-trim class=scala>
private val eitherFunctorCached = new Functor[Either[Nothing, ?]] {
  override def map[A, B](fa: Either[Nothing, A])(f: A =&gt; B): Either[Nothing, B] =
    fa.right.map(f)
}

implicit def eitherFunctor[L]: Functor[Either[L, ?]] =
  eitherFunctorCached.asInstanceOf[Functor[Either[L, ?]]]
</code></pre>

<div class=fragment>
**As you'd expect...**
<pre><code data-noescape data-trim class=scala>
<span class="fragment">implicitly[Functor[Either[String, ?]]]</span>
<span class="fragment">// $anon$1@3c466028</span>

<span class="fragment">implicitly[Functor[Either[String, ?]]]</span>
<span class="fragment">// $anon$1@3c466028</span><span class="fragment"> ← same instance!</span>
</code></pre>


Note:
Here we implement the same Functor instance, except this time instead of making
it polymorphic, we stub in the Nothing type for the Left type param so we can cache
it into a val. We then have the implicit def simply call the val and cast it to
the appropriate type.

---

## Why does this work?

Type parameters are erased at runtime

<pre><code data-noescape data-trim class=scala>
implicit def eitherFunctor[L]: Functor[Either[L, ?]] = new Functor[Either[L, ?]] {
  override def map[A, B](fa: Either[L, A])(f: A =&gt; B): Either[L, B] =
    fa.right.map(f)
}
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public eitherFunctor()LFunctor;
  NEW $anon$1                      // new anon Functor instance
  DUP
  INVOKESPECIAL $anon$1.&lt;init&gt; ()V // Functor constructor &quot;init&quot;
  ARETURN
</code></pre>

Note:

Why does this work? Type parameters are erased at runtime.

---

## Why does this work?

<pre><code data-noescape data-trim class=scala>
private val eitherFunctorCached = new Functor[Either[Nothing, ?]] {
  override def map[A, B](fa: Either[Nothing, A])(f: A =&gt; B): Either[Nothing, B] =
    fa.right.map(f)
}

implicit def eitherFunctor[L]: Functor[Either[L, ?]] =
  eitherFunctorCached.asInstanceOf[Functor[Either[L, ?]]]
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public eitherFunctor()LFunctor; <span class="fragment">// ← Runtime type is still Functor</span>
  ALOAD 0
  INVOKESPECIAL eitherFunctorCached ()LFunctor;
  ARETURN            <span class="fragment">// Same runtime type ↑</span>
</code></pre>

---

## The @cached macro

<pre><code data-noescape data-trim class=scala>
@cached
implicit def eitherFunctor[L]: Functor[Either[L, ?]] = new Functor[Either[L, ?]] {
  override def map[A, B](fa: Either[L, A])(f: A =&gt; B): Either[L, B] =
    fa.right.map(f)
}
</code></pre>

<div class=fragment>
Expands to...

<div class=fragment>
<pre><code data-noescape data-trim class=scala>
implicit def eitherFunctor[L]: Functor[Either[L, ?]] =
  __cached__eitherFunctor.asInstanceOf[Functor[Either[L, ?]]]

private val __cached__eitherFunctor = new Functor[Either[Nothing, ?]] {
  override def map[A, B](fa: Either[Nothing, A])(f: A =&gt; B): Either[Nothing, B] =
    fa.right.map(f)
}
</code></pre>

---

## newtypes

---

## NewTypes: A Motivation

Haskell's `newtype` gives us type safety without the cost

<pre><code data-noescape data-trim class=haskell>
newtype WidgetId = WidgetId String

<span class="fragment">myId = WidgetId &quot;a&quot; </span><span class="fragment"> -- ← Is just a String at runtime</span>

<span class="fragment">upperId :: WidgetId -&gt; WidgetId
upperId (WidgetId s) = WidgetId (map toUpper s)</span>
<span class="fragment">-- The unwrap ↑ and rewrap ↑ do not occur at runtime</span>
</code></pre>

Note:
In Haskell, the `newtype` keyword allows us to define a new type
which incurs no overhead and is just some other type at runtime.

In this case, we're defining a `WidgetId` to improve type safety
so we know that this value is more than just a `String`.

Wrapping and unwrapping newtype values incurs no overhead at runtime; it is
all omitted during compilation.

---

## NewTypes: Specialized type class instances

<pre><code data-noescape data-trim class=haskell>
instance ToJSON WidgetId where
  toJSON (WidgetId s) = object [ &quot;widgetId&quot; .= toJSON s ]
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=haskell>
newtype Sum = Sum Int

instance Monoid Sum where
  mempty = 0
  mappend (Sum x) (Sum y) = Sum (x + y)

newtype Product = Product Int

instance Monoid Product where
  mempty = 1
  mappend (Product x) (Product y) = Product (x * y)
</code></pre>

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

<pre><code data-noescape data-trim class=haskell>
ids :: [String]
ids = [&quot;a&quot;,&quot;b&quot;,&quot;c&quot;,&quot;d&quot;,&quot;e&quot;]

xs :: [WidgetId]
xs = coerce ids :: [WidgetId]
</code></pre>

---

## Value Classes: NewTypes for Scala?

<pre><code data-noescape data-trim class=scala>
final case class WidgetId(value: String) <span class="fragment">extends AnyVal</span>
</code></pre>

<div class=fragment>
Usage
<pre><code data-noescape data-trim class=scala>
def getWidget(id: WidgetId): Widget = ???
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public getWidget(Ljava/lang/String;)LWidget;
</code></pre>

Note:
Here is a value class. This allows us to define a new class which,
at runtime, should be represented as a String.
We can confirm this by looking at the compiled bytecode.

---

## Value Class Pitfalls: Generics

<pre><code data-noescape data-trim class=scala>
def optId = Option(WidgetId(&quot;a&quot;))
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public optId()Lscala/Option;
  GETSTATIC scala/Option$.MODULE$ : Lscala/Option$;
  <span class="fragment highlight-red">NEW WidgetId</span><span class="fragment"> // ← We don&#x27;t like this</span>
  DUP
  LDC &quot;a&quot;
  INVOKESPECIAL WidgetId.&lt;init&gt; (Ljava/lang/String;)V
  INVOKEVIRTUAL scala/Option$.apply (Ljava/lang/Object;)Lscala/Option;
  ARETURN
</code></pre>

Note:
Let's see what's _wrong_ with value classes. One of the big problems with them
is when you try to use them with generics.

Also remember that since generics are erased at runtime,
this method returns Option, not Option of String or WidgetId.

---

## Value Class Pitfalls: Generics

<pre><code data-noescape data-trim class=scala>
def unOptId = optId.get
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public unOptId()Ljava/lang/String;
  ALOAD 0
  INVOKEVIRTUAL optId ()Lscala/Option;
  INVOKEVIRTUAL scala/Option.get ()Ljava/lang/Object;
  <span class="fragment highlight-red">CHECKCAST WidgetId</span> <span class="fragment">// ← Why the allocation occurs</span>
  <span class="fragment highlight-red">INVOKEVIRTUAL WidgetId.value ()Ljava/lang/String;</span> <span class="fragment"> // ← Unwrap</span><span class="fragment"> ლ(ಠ益ಠლ)</span>
  ARETURN
</code></pre>

Note:
* The allocation must occur so the JVM can validate that we actually have
  a WidgetId at runtime.
* Interestingly enough, we then just unwrap the value we had, getting back the
  String we wrapped to begin with.

---

## Value Class Pitfalls: O(n) conversions

<pre><code data-noescape data-trim class=scala>
def convIdList(xs: List[String]): List[WidgetId] =
  xs.map(WidgetId.apply) <span class="fragment">// ← Hey JIT, try to optimize this</span>
</code></pre>

Note:
Any time we need to convert container elements to value class instances,
we'll always incur a performance overhead; this just can't be optimized
away, not even by the JIT.

---

## Tagged Types

<pre><code data-noescape data-trim class=scala>
// Adapted from scalaz
type @@[A, T] = { type Tag = T; type Self = A }
def tag[A, T](a: A): A @@ T = a.asInstanceOf[A @@ T]

<span class="fragment">trait WidgetIdTag</span>
<span class="fragment">type WidgetId = String @@ WidgetIdTag</span>
<span class="fragment">def WidgetId(s: String): WidgetId = tag[String, WidgetIdTag](s)</span>
</code></pre>

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

<pre><code data-noescape data-trim class=scala>
def tag[A, T](a: A): A @@ T = a.asInstanceOf[A @@ T]
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public tag(Ljava/lang/Object;)Ljava/lang/Object;
  ALOAD 1
  ARETURN
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=scala>
def WidgetId(s: String): WidgetId = tag[String, WidgetIdTag](s)
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public WidgetId(Ljava/lang/String;)Ljava/lang/Object;
  ALOAD 0
  ALOAD 1
  INVOKEVIRTUAL tag (Ljava/lang/Object;)Ljava/lang/Object;
  ARETURN
</code></pre>

---

## Tagged Types: Under the hood

<pre><code data-noescape data-trim class=scala>
def optId = Option(WidgetId(&quot;a&quot;))
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public optId()Lscala/Option;
  GETSTATIC scala/Option$.MODULE$ : Lscala/Option$;
  ALOAD 0
  LDC &quot;a&quot;
  INVOKEVIRTUAL WidgetId (Ljava/lang/String;)Ljava/lang/Object;
  INVOKEVIRTUAL scala/Option$.apply (Ljava/lang/Object;)Lscala/Option;
  ARETURN
</code></pre>

---

## Tagged Types: Under the hood

<pre><code data-noescape data-trim class=scala>
def unOptId = optId.get
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public unOptId()Ljava/lang/Object;
  ALOAD 0
  INVOKEVIRTUAL optId ()Lscala/Option;
  INVOKEVIRTUAL scala/Option.get ()Ljava/lang/Object;
  ARETURN
</code></pre>

---

## Tagged Types: Faster conversions

<pre><code data-noescape data-trim class=scala>
def convIdList(ids: List[String]): List[WidgetId] =
  ids.asInstanceOf[List[WidgetId]]
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public convIdList(Lscala/collection/immutable/List;)Lscala/collection/immutable/List;
  ALOAD 1
  ARETURN
</code></pre>

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

<pre><code data-noescape data-trim class=scala>
<span class="fragment" data-fragment-index="4">type WidgetId = WidgetId.Type</span>
object WidgetId {
  <span class="fragment" data-fragment-index="1">type Base = Any { type WidgetId$newtype }</span>
  <span class="fragment" data-fragment-index="2">trait Tag extends Any</span>
  <span class="fragment" data-fragment-index="3">type Type &lt;: Base with Tag</span>

  <span class="fragment" data-fragment-index="5">def apply(value: String): WidgetId = value.asInstanceOf[WidgetId]</span>

  <span class="fragment" data-fragment-index="6">implicit final class Ops(private val me: WidgetId) extends AnyVal {
    def value: String = me.asInstanceOf[String]
  }</span>
}
</code></pre>

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

<pre><code data-noescape data-trim class=scala>
WidgetId(&quot;a&quot;).<span class="fragment"><span class="fragment highlight-red">toUpperCase</span></span> <span class="fragment">// Does not compile</span>
<span class="fragment">WidgetId(&quot;a&quot;).value.toUpperCase</span> <span class="fragment">// Compiles</span>

<span class="fragment">def upperStr(s: String) = s.toUpperCase</span>
<span class="fragment">upperStr(&quot;a&quot;)</span> <span class="fragment">// Compiles</span>
<span class="fragment"><span class="fragment highlight-red">upperStr</span>(WidgetId(&quot;a&quot;))</span> <span class="fragment">// Does not compile</span>

<span class="fragment">def upperWidgetId(id: WidgetId) = WidgetId(id.value.toUpperCase)</span>
<span class="fragment"><span class="fragment highlight-red">upperWidgetId</span>(&quot;a&quot;)</span> <span class="fragment">// Does not compile</span>
<span class="fragment">upperWidgetId(WidgetId(&quot;a&quot;))</span> <span class="fragment">// Compiles</span>

</code></pre>

---

## NewType: Builder trait

<pre><code data-noescape data-trim class=scala>
trait NewType[Repr] {
  <span class="fragment" data-fragment-index="10">type Base = Any { type Repr$newtype = Repr }</span>
  <span class="fragment" data-fragment-index="20">trait Tag extends Any</span>
  <span class="fragment" data-fragment-index="30">type Type &lt;: Base with Tag</span>
  <span class="fragment" data-fragment-index="40">def apply(value: Repr): Type = value.asInstanceOf[Type]</span>
  <span class="fragment" data-fragment-index="50">def repr(x: Type): Repr = x.asInstanceOf[Repr]</span>
}
</code></pre>

<div class=fragment data-fragment-index=70>
Usage
<pre><code data-noescape data-trim class=scala>
<span class="fragment" data-fragment-index="80">type WidgetId = WidgetId.Type</span>
object WidgetId extends NewType[String] <span class="fragment" data-fragment-index="90">{
  implicit final class Ops(private val me: WidgetId) extends AnyVal {
    def value: Repr = repr(me)
  }
}</span>
</code></pre>

Note:

So what does this buy us?

We get a smart constructor and destructure for free.
We get safe casts for collection types.
We get an idiomatic companion object that **just works** for implicit resolution.

---

## NewType: At runtime

<pre><code data-noescape data-trim class=java>
public apply(Ljava/lang/Object;)Ljava/lang/Object;
  ALOAD 0
  ALOAD 1
  INVOKESTATIC <span class="fragment highlight-green">NewType.apply$</span> (LNewType;Ljava/lang/Object;)Ljava/lang/Object;
  ARETURN

<span class="fragment">public static synthetic apply$(LNewType;Ljava/lang/Object;)Ljava/lang/Object;
  ALOAD 0
  ALOAD 1
  INVOKESPECIAL <span class="fragment highlight-green">NewType.apply</span> (Ljava/lang/Object;)Ljava/lang/Object;
  ARETURN</span>

<span class="fragment">public default apply(Ljava/lang/Object;)Ljava/lang/Object;
  ALOAD 1
  ARETURN</span>
</code></pre>

---

## NewType: Extension Methods

<pre><code data-noescape data-trim class=scala>
def widgetIdValue = WidgetId(&quot;a&quot;).value
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public widgetIdValue()Ljava/lang/String;
  GETSTATIC WidgetId$.MODULE$ : LWidgetId$;
  GETSTATIC WidgetId$.MODULE$ : LWidgetId$;
  LDC &quot;a&quot;
  INVOKEVIRTUAL WidgetId$.apply (Ljava/lang/Object;)Ljava/lang/Object;
  INVOKEVIRTUAL WidgetId$.Ops (Ljava/lang/Object;)Ljava/lang/Object;
  INVOKEVIRTUAL <span class="fragment highlight-green">WidgetId$Ops$.value$extension</span> (Ljava/lang/Object;)Ljava/lang/String;
</code></pre>

---

## NewType: Extension Methods

<pre><code data-noescape data-trim class=scala>
implicit final class Ops(private val self: Type) extends AnyVal {
  def value: Repr = repr(self)
}
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public final value$extension(Ljava/lang/Object;)Ljava/lang/String;
  GETSTATIC WidgetId$.MODULE$ : LWidgetId$;
  ALOAD 1
  INVOKEVIRTUAL WidgetId$.repr (Ljava/lang/Object;)Ljava/lang/Object;
  CHECKCAST java/lang/String
  ARETURN
</code></pre>

---

## NewType: Generics

<pre><code data-noescape data-trim class=scala>
def optId = Option(WidgetId(&quot;a&quot;))
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public optId()Lscala/Option;
  GETSTATIC scala/Option$.MODULE$ : Lscala/Option$;
  GETSTATIC WidgetId$.MODULE$ : LWidgetId$;
  LDC &quot;a&quot;
  INVOKEVIRTUAL WidgetId$.apply (Ljava/lang/Object;)Ljava/lang/Object;
  INVOKEVIRTUAL scala/Option$.apply (Ljava/lang/Object;)Lscala/Option;
  ARETURN
</code></pre>

---

## NewType: Generics

<pre><code data-noescape data-trim class=scala>
def unOptId = optId.get
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
public unOptId()Ljava/lang/Object;
  ALOAD 0
  INVOKEVIRTUAL optId ()Lscala/Option;
  INVOKEVIRTUAL scala/Option.get ()Ljava/lang/Object;
  ARETURN
</code></pre>

---

## NewType: Constant-time conversions

<pre><code data-noescape data-trim class=scala>
def convIdList(ids: List[String]): List[WidgetId] = ids.asInstanceOf[List[WidgetId]]
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public convIdList(Lscala/collection/immutable/List;)Lscala/collection/immutable/List;
  ALOAD 1
  ARETURN
</code></pre>

---

## NewTypes: Primitive Problems

<pre><code data-noescape data-trim class=scala>
type Feet = Feet.Type
object Feet extends NewType.Of[Int]
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=scala>
def mkFeet = Feet(12)
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
// Bytecode
public mkFeet()Ljava/lang/Object;
  GETSTATIC Feet$.MODULE$ : LFeet$;
  BIPUSH 12
  INVOKESTATIC scala/runtime/BoxesRunTime.<span class="fragment highlight-red">boxToInteger</span> (I)Ljava/lang/Integer;
  INVOKEVIRTUAL Feet$.apply (Ljava/lang/Object;)Ljava/lang/Object;
  ARETURN
</code></pre>

---

## NewSubType: An Attempt

<pre><code data-noescape data-trim class=scala>
<span class="fragment" data-fragment-index="20">trait NewSubType[Repr] {
  <span class="fragment" data-fragment-index="50">trait Tag extends Any</span>
  <span class="fragment" data-fragment-index="60">type Type &lt;: Repr with Tag</span>

  <span class="fragment" data-fragment-index="70">def apply(x: Repr): Type = x.asInstanceOf[Type]</span>
  <span class="fragment" data-fragment-index="80">def repr(x: Type): Repr = x.asInstanceOf[Repr]</span>
}</span>
</code></pre>

---

## NewSubType


<pre><code data-noescape data-trim class=scala>
<span class="fragment">type Feet = Feet.Type</span>
<span class="fragment">object Feet extends NewSubType[Int] {
  <span class="fragment">implicit final class Ops(private val me: Feet) extends AnyVal {
    def value: Int = repr(me)
  }</span>
}</span>
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=scala>
Feet(1)<span class="fragment">.signum</span> <span class="fragment">// Compiles, returns Int</span>

<span class="fragment">def add(x: Int, y: Int): Int = x + y</span>
<span class="fragment">add(Feet(1), Feet(2))</span> <span class="fragment">// Compiles, returns Int</span>

<span class="fragment">def addFeet(x: Feet, y: Feet): Feet = Feet(x.value + y.value)</span>
<span class="fragment">addFeet(Feet(1), Feet(2))</span> <span class="fragment">// Compiles, returns Feet</span>
<span class="fragment"><span class="fragment highlight-red">addFeet</span>(1, 2)</span> <span class="fragment">// Does not compile</span>
</code></pre>


---

## NewSubType: Is it really zero cost?

<div class=fragment data-fragment-index=1>
<pre><code data-noescape data-trim class=scala>
def mkFeet = Feet(12)
</code></pre>

<div class=fragment data-fragment-index=2>
<pre><code data-noescape data-trim class=java>
// Bytecode
public mkFeet()I
  GETSTATIC Feet$.MODULE$ : LFeet$;
  BIPUSH 12
  INVOKESTATIC scala/runtime/BoxesRunTime.<span class="fragment highlight-red" data-fragment-index="3">boxToInteger</span> (I)Ljava/lang/Integer;
  INVOKEVIRTUAL Feet$.apply <span class="fragment highlight-red" data-fragment-index="6">(Ljava/lang/Object;)Ljava/lang/Object;</span> <span class="fragment" data-fragment-index="7">// oh</span>
  INVOKESTATIC scala/runtime/BoxesRunTime.<span class="fragment highlight-red" data-fragment-index="4">unboxToInt</span> (Ljava/lang/Object;)I <span class="fragment" data-fragment-index="5">// wat</span>
  IRETURN
</code></pre>

---

## NewSubType: Fixing our broken ctor

<pre><code data-noescape data-trim class=scala>
type Feet = Feet.Type
object Feet extends NewSubType[Int] {
  <span class="fragment">override def apply(x: Int): Feet = x.asInstanceOf[Feet]</span>
}
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
public apply(I)I
  ILOAD 1
  <span class="fragment highlight-red">INVOKESTATIC scala/runtime/BoxesRunTime.boxToInteger</span> (I)Ljava/lang/Integer;
  <span class="fragment highlight-red">INVOKESTATIC scala/runtime/BoxesRunTime.unboxToInt</span> (Ljava/lang/Object;)I <span class="fragment">// srsly</span>
  IRETURN
</code></pre>

---

## NewSubType: Giving scalac a hand

<pre><code data-noescape data-trim class=scala>
type Feet = Feet.Type
object Feet extends NewSubType.Of[Int] {
  override def apply(x: Int): Feet = x.asInstanceOf[Feet]
}
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=bash>
% scalac <span class="fragment highlight-green">-opt:l:inline</span> ...
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
public apply(I)I
  ILOAD 1
  IRETURN <span class="fragment">// ...Did we just get it to work?</span>
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=scala>
def mkFeet = Feet(12)
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=java>
public mkFeet()I
  GETSTATIC Feet$.MODULE$ : LFeet$;
  BIPUSH 12
  INVOKEVIRTUAL Feet$.apply (I)I
  IRETURN <span class="fragment">// ...We did!</span>
</code></pre>

---

## Coercible: Abstracting over newtypes

<pre><code data-noescape data-trim class=scala>
trait Coercible[A, B] {
  final def apply(a: A): B = a.asInstanceOf[B]
}
</code></pre>

---

## Generated Coercible instances

<pre><code data-noescape data-trim class=scala>
trait NewType[Repr] {
  type Base = Any { type Repr$newtype = Repr }
  trait Tag extends Any
  type Type &lt;: Base with Tag
  def apply(value: Repr): Type = value.asInstanceOf[Type]
  def repr(x: Type): Repr = x.asInstanceOf[Repr]

  <span class="fragment">implicit val coerceTo:   Coercible[Repr, Type] = Coercible.instance
  implicit val coerceFrom: Coercible[Type, Repr] = Coercible.instance</span>
}
</code></pre>

---

## Coercible in action

<pre><code data-noescape data-trim class=scala>

WidgetId(&quot;foo&quot;)<span class="fragment">.coerce[String]<span class="fragment">.toUpperCase</span></span>

<span class="fragment">def upper(s: String) = s.toUpperCase</span>

<span class="fragment">upper(WidgetId(&quot;foo&quot;).coerce)</span> <span class="fragment">// Compiles, returns String</span>

<span class="fragment">implicit def coercibleListTo[A, B](
  implicit ev: Coercible[A, B]
): Coercible[List[A], List[B]] = Coercible.instance</span>

<span class="fragment">List(&quot;a&quot;,&quot;b&quot;,&quot;c&quot;).coerce[List[WidgetId]]</span>
</code></pre>

---

## Aside on type roles

https://github.com/estatico/scala-newtype/issues/29

<pre><code data-noescape data-trim class=scala>
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
</code></pre>

---

## Shilling for @newtype

https://github.com/estatico/scala-newtype

<pre><code data-noescape data-trim class=scala>
import io.estatico.newtype.macros._

<span class="fragment">@newtype case class WidgetId(value: String)</span>

<span class="fragment">@newsubtype case class Feet(value: Int)</span>
</code></pre>

---

## @newtype: inspecting generated code

<pre><code data-noescape data-trim class=scala>
@newtype(debug = true) case class WidgetId(value: String)
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=scala>
type WidgetId = WidgetId.Type
object WidgetId {
  type Repr = String
  type Base = Any { type WidgetId$newtype }
  abstract trait Tag extends Any
  type Type &lt;: Base with Tag

  def apply(value: String): WidgetId = value.asInstanceOf[WidgetId];

  def deriving[TC[_]](implicit ev: TC[Repr]): TC[Type] = ev.asInstanceOf[TC[Type]]

  // ...
}
</code></pre>

---

## @newtype: deriving type class instances

<pre><code data-noescape data-trim class=scala>
@newtype case class Attributes(toList: List[(String, String)])
object Attributes {
  implicit val monoid: Monoid[Attributes] = <span class="fragment">deriving</span>
}
</code></pre>

---

## @newtype: derivingK

<pre><code data-noescape data-trim class=scala>
@newtype case class Slice[A](toVector: Vector[A])
object Slice {
  implicit val monad: Monad[Slice] = <span class="fragment">derivingK</span>
}
</code></pre>

---

## customized smart constructors

<pre><code data-noescape data-trim class=scala>
@newsubtype class PosInt(val toInt: Int)
<span class="fragment">object PosInt {
  def of(x: Int): Option[PosInt] =
    if (x &lt; 0) None else Some(x.coerce[PosInt])
}</span>
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=scala>
PosInt(1) <span class="fragment">// Compile error: PosInt.type does not take parameters</span>

<span class="fragment">PosInt.of(1)</span>  <span class="fragment">// Some(1): Option[PosInt]</span>
<span class="fragment">PosInt.of(-1)</span> <span class="fragment">// None:    Option[PosInt]</span>
</code></pre>

---

## Unboxed Maybe type

<pre><code data-noescape data-trim class=scala>
@newtype class Maybe[A](val unsafeGet: A) {
  def isEmpty:   Boolean = unsafeGet == Maybe._empty
  def isDefined: Boolean = unsafeGet != Maybe._empty

  def map[B](f: A =&gt; B): Maybe[B] =
    if (isEmpty) Maybe.empty else Maybe(f(unsafeGet))

  def filter(p: A =&gt; Boolean): Maybe[A] =
    if (isEmpty || !p(unsafeGet)) Maybe.empty else this
  ...
}
</code></pre>
<pre><code data-noescape data-trim class=scala>
object Maybe {
  def apply[A](a: A): Maybe[A] = if (a == null) empty else unsafe(a)
  def unsafe[A](a: A): Maybe[A] = a.asInstanceOf[Maybe[A]]
  def empty[A]: Maybe[A] = _empty.asInstanceOf[Maybe[A]]
  private val _empty = new Empty
  private final class Empty { override def toString = &quot;Maybe.empty&quot; }
}
</code></pre>

---

## NewType Transformers

<pre><code data-noescape data-trim class=scala>
@newtype case class OptionT[F[_], A](value: F[Option[A]]) {

  def map[B](f: A =&gt; B)(implicit F: Functor[F]): OptionT[F, B] =
    OptionT(F.map(value)(_.map(f)))

  def flatMapF[B](f: A =&gt; F[Option[B]])(implicit F: Monad[F]): OptionT[F, B] =
    OptionT(F.flatMap(value)(_.fold(F.pure[Option[B]](None))(f)))

  def flatMap[B](f: A =&gt; OptionT[F, B])(implicit F: Monad[F]): OptionT[F, B] =
    flatMapF(a =&gt; f(a).value)

  ...
}
</code></pre>

---

## Generic Programming: Shapeless

<pre><code data-noescape data-trim class=scala>
import shapeless._

case class Foo(a: Int, b: String)

<span class="fragment">val foo = Foo(1, &quot;hey&quot;)
<span class="fragment">val g = Generic[Foo].to(foo)
<span class="fragment">// g: Int :: String :: HNil
<span class="fragment">//  = 1   :: &quot;hey&quot;  :: HNil

<span class="fragment">foo eq g
<span class="fragment">// false
</code></pre>

---

## HList allocations

<pre><code data-noescape data-trim class=scala>
(1 :: &quot;hey&quot; :: HNil) == g
<span class="fragment">// true

<span class="fragment">new ::(1, new ::(&quot;hey&quot;, HNil)) == g
<span class="fragment">// true
</code></pre>

---

## HList: An alternative

<pre><code data-noescape data-trim class=scala>
sealed trait HList
<span class="fragment" data-fragment-index="1">final case class ::[+H, +T &lt;: HList](head : H, tail : T) extends HList
<span class="fragment" data-fragment-index="2">sealed trait HNil extends HList
</code></pre>

<div class=fragment data-fragment-index=3>
Alternatively...
<pre><code data-noescape data-trim class=scala>
<span class="fragment" data-fragment-index="20">sealed trait GList</span>
<span class="fragment" data-fragment-index="30">sealed trait #:[H, T &lt;: GList] extends GList</span>
<span class="fragment" data-fragment-index="40">sealed trait GNil extends GList</span>
<span class="fragment" data-fragment-index="50">object GList {
  @newtype case class Of[A, L &lt;: GList](value: A)
  <span class="fragment" data-fragment-index="70">object Of {</span>           <span class="fragment" data-fragment-index="60">//  ^ phantom type</span>
    <span class="fragment" data-fragment-index="70">// It&#x27;s valid (and desirable) to coerce to the tail of our GList
    implicit def coerceToTail[A, H, T &lt;: GList]: Coercible[Of[A, H #: T], Of[A, T]] =
        Coercible.instance
  }</span>
}</span>
</code></pre>

---

## Generic Products

<pre><code data-noescape data-trim class=scala>
trait GProduct[A] { type Repr &lt;: GList }

object GProduct {

  type Aux[A, R &lt;: GList] = GProduct[A] { type Repr = R }

  def to[A](a: A)(implicit ev: GProduct[A]): GList.Of[A, ev.Repr] = GList.Of(a)
}
</code></pre>

---

## Accessing Fields Generically

<pre><code data-noescape data-trim class=scala>
trait IsGCons[A] {
  <span class="fragment">type Head
  type Tail &lt;: GList</span>
  <span class="fragment">def head(a: GList.Of[A, Head #: Tail]): Head</span>
  <span class="fragment">final def tail(a: GList.Of[A, Head #: Tail]): GList.Of[A, Tail] =
    a.coerce[GList.Of[A, Tail]]</span>
}

<span class="fragment">object IsGCons {
  type Aux[A, H, T &lt;: GList] = IsGCons[A] { type Head = H ; type Tail = T }
}</span>
</code></pre>

---

## Deriving GProduct and IsGCons

<pre><code data-noescape data-trim class=scala>
@DeriveGProduct case class Foo(a: String, b: Int, c: Float, d: Double)
</code></pre>

<div class=fragment>
<pre><code data-noescape data-trim class=scala>
object Foo {
  implicit val isGCons4: IsGCons.Aux[Foo, String, Int #: Float #: Double #: GNil] =
    IsGCons.instance(_.a)
  <span class="fragment">implicit val isGCons3: IsGCons.Aux[Foo, Int, Float #: Double #: GNil] =
    IsGCons.instance(_.b)</span>
  <span class="fragment">implicit val isGCons2: IsGCons.Aux[Foo, Float, Double #: GNil] =
    IsGCons.instance(_.c)</span>
  <span class="fragment">implicit val isGCons1: IsGCons.Aux[Foo, Double, GNil] =
    IsGCons.instance(_.d)</span>
  <span class="fragment">implicit def gProduct: GProduct.Aux[Foo, String #: Int #: Float #: Double #: GNil] =
    GProduct.instance</span>
};
</code></pre>

---

## Deriving new type class instances

<pre><code data-noescape data-trim class=scala>
trait CsvEncoder[A] {
  def encode(a: A): String
}
</code></pre>

---

## Deriving new type class instances

<pre><code data-noescape data-trim class=scala>
implicit def gCons[A, H, T &lt;: GList](
  implicit
  <span class="fragment">hEnc: CsvEncoder[H],</span>
  <span class="fragment">tEnc: CsvEncoder[GList.Of[A, T]],</span>
  <span class="fragment">isGCons: IsGCons.Aux[A, H, T]</span>
): CsvEncoder[GList.Of[A, H #: T]] =
  <span class="fragment">CsvEncoder.instance(a =&gt; hEnc.encode(a.head) + &#x27;,&#x27; + tEnc.encode(a.tail))</span>

<span class="fragment">implicit def gSingle[A, H](
  implicit
  <span class="fragment">hEnc: CsvEncoder[H],</span>
  <span class="fragment">isGCons: IsGCons.Aux[A, H, GNil]</span>
): CsvEncoder[GList.Of[A, H #: GNil]] =
  <span class="fragment">CsvEncoder.instance(a =&gt; hEnc.encode(a.head))</span></span>
</code></pre>

---

## It works!

<pre><code data-noescape data-trim class=scala>
@DeriveGProduct case class Foo(a: String, b: Int, c: Float, d: Double)
object Foo {
  implicit val csvFoo = CsvEncoder[Foo] = CsvEncoder.derive[Foo]
}

<span class="fragment">CsvEncoder.encode(Foo(&quot;ya&quot;, 2, 8.2f, 7.6))</span>
<span class="fragment">// &quot;ya,2,8.2,7.6&quot;</span>
</code></pre>

---

## Thank you!

* NewType Library - https://github.com/estatico/scala-newtype
* `@cached` Implementation - https://github.com/estatico/scala-cached
* GProduct Implementation - https://github.com/carymrobbins/scala-derivable


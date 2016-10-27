# Modelling an RPG in D
For the past year, I've done the majority of my personal coding in a language
called [D](http://dlang.org/). 

This post will explore some of the cool features of D in the context of creating
an RPG.

# Character Stats
Pretty much any RPG worth its salt is going to have some form of 'stats'.

For our RPG, lets say there are three categories of stats on every character:

- Attributes: An `int` value for each of the classic 6 (Strength, Dexterity, ect.)
- Skills:     An `int` value for each of several skills (diplomacy, stealth, ect.).
- Resistance: An `int` value for each 'type' (physical, fire, ect.) of damage

In D, we could represent such a character like so:

```d
struct Character {
  // attributes
  int strength;
  int dexterity;
  int constitution;
  int intellect;
  int wisdom;
  int charisma;

  // skills
  int stealth;
  int perception;
  int diplomacy;

  // resistances
  int resistPhysical;
  int resistFire;
  int resistWater;
  int resistAir;
  int resistEarth;
}
```

So far, pretty typical -- this looks just like it would in C.

However, it would be nicer if we could have each category
(attributes, skills, resistances) represented as a single group of values.

First, let's define some `enum`s:

```d
enum Attribute { strength, dexterity, constitution, intellect, wisdom, charisma }
enum Skill { stealth, perception, diplomacy }
enum Element { physical, fire, water, air, earth }
```

Now we want to map each of these enum 
One option is an [Associative Array](http://dlang.org/spec/hash-map.html), which
would look like this:

```d
struct Character {
  int[Attribute] attributes;
  int[Skill] attributes;
  int[Element] attributes;
}
```

However, associative arrays are heap allocated and don't have a default value
for each key. It seems like overkill for storing a small bundle of values.

Another option is a 
[static array](http://dlang.org/spec/arrays.html#static-arrays). Static arrays
are stack allocated value types and will contain exactly the number of values
that we need.

```d
struct Character {
  int[6] attributes;
  int[3] skills;
  int[5] resistances;
}
```

Our enum values are backed by `int`s, so we can use them directly as indexes
like so:

```d
if (hero.attributes[Attribute.dexterity] < 4) hero.tripOverOwnFeet();
```

This is more efficient for our needs, but nothing enforces using enums as keys.
The following is just as valid:

```d
if (hero.attributes[2] < 4) hero.tripOverOwnFeet();
```

If we acidentally gave an out of bounds index, the compiler wouldn't catch it
and we'd get a runtime error.

Ideally, we want the efficiency of the static array with the syntax of the
associative array. Fortunately, the power of D allows us to achieve this with a
few lines of code, and something even cooler with a few more. Let's call it an
`Enumap`, as it maps enum members to values.

# The Enumap

```d
import std.traits;

struct Enumap(K, V) {
  private enum N = EnumMembers!K.length;
  private V[N] _store;

  auto opIndex(K key)                { return _store[key]; }
  auto opIndexAssign(T value, K key) { return _store[key] = value; }
}
```

Here's a line-by-line breakdown:

```d
import std.traits;
```

We need access to `std.traits.EnumMembers`, a standard-library function that
returns (at compile-time!) the members of an enum.

```d
struct Enumap(K, V)
```

Here, we declare a templated struct. In many other languages,
this would look like `Enumap<K, V>`.
K will be our key type (the enum) and V will be the value.

K and V are known as 'compile-time parameters'. In this case, they are simply
used to create a generic type, but in D such parameters can be used for much
more than just generic types, as we will see later.

```d
private enum N = EnumMembers!K.length;`
private V[N] _store;`
```

Here we leverage `EnumMembers`,  to determine how many entries are in the
provided enum. We use this to declare a static array capable of holding exactly
`N` `V`s.

6: `auto opIndex(K key)                { return _store[key]; }`
7: `auto opIndexAssign(T value, K key) { return _store[key] = value; }`

`opIndex` is a special method that allows us to provide a custom implementation
of the indexing (`[]`) operator. The call `skills[Skill.stealth]` is translated
to `sklls.opIndex(Skill.stealth)`, while the assignment `skills[Skill.stealth] =
5` is translated to `sklls.opIndexAssign(Skill.stealth, 5)`.

Lets use that in our character struct:

```d
struct Character {
  Enumap!(Attribute, int) attributes;
  Enumap!(Skill    , int) skills;
  Enumap!(Element  , int) resistances;
}
```

```d
if (hero.attributes[Attribute.wisdom] < 2) hero.drink(unidentifiedPotion);
```

There! Now the length of each underlying array is figured out for us, and the
values can only be accessed using the enum members as keys.
The underlying array `_store` is statically sized, so it requires no
managed-memory allocation.

Clever, yes?

Wait, that wasn't the clever bit.
**This** is the clever bit:

```d
import std.conv;
//...

struct Enumap(K, V) {
  //...
  auto opDispatch(string s)()      { return this[s.to!K]; }
  auto opDispatch(string s)(V val) { return this[s.to!K] = val; }
}
```

```d
if (hero.attributes.charisma < 5) hero.makeAwkwardJoke();
```

This little bit of magic leverages
[opDispatch](http://dlang.org/spec/operatoroverloading.html#dispatch) to
overload the `.` operator and give us some really nice syntactic sugar.

Here's a quick rundown of what happens for `hero.attributes.charisma = 5`:

1. The compiler sees `attributes.charisma`.
2. It looks for the symbol `charisma` in the type `Enumap!(Attribute, int)`
3. Failing to find this, it tries `attributes.opDispatch!"charisma"`
4. That call resolves to `attributes["charisma".to!Attribute]`
5. And further resolves to `attributes[Attribute.charisma]`

Remember how I mentioned that compile time arguments can be much more than
types? Here, `s` is a compile-time string argument -- in this case, it's value
is whatever symbol followed the `.`.

So, we get the string "charisma", but we what we actually want the enum member 
`Attribute.charisma`. `std.conv.to`, the swiss-army-knife of type conversions,
makes quick work of this; it can, among other things, translate between strings
and enum names.

# A step further: Enumap Arithmetic

Let's suppose we add items to the game, and each item can provide some stat
bonuses:

```d
struct Item {
    Enumap!(Attribute, int) bonuses;
}
```

It would be really nice if we could just add these bonuses to our character's
base stats like so:

```
auto totalStats = character.attributes + item.bonuses;
```

Yet again, D lets us implement this quite concisely, this time by leveraging
[`opBinary`](https://dlang.org/spec/operatoroverloading.html#binary).

```d
struct Enumap(K, V) {
  //...
  auto opBinary(string op)(typeof(this) other) {
    V[N] result = mixin("_store[] " ~ op ~ " other._store[]");
    return typeof(this)(result);
  }
```

Breakdown time again!

```d
auto opBinary(string op)(typeof(this) other)
```

An expression like `enumap1 + enumap2` will get translated (at compile
time!) to `enumap1.opBinary!"+"(enumap2). The operator (in this case `+`) is
passed as a compile time string argument. If passing the operator as a string
sounds weird, read on ...

```d
V[N] result = mixin("_store[]" ~ op ~ "other._store[]");
```

`mixin` is a D keyword that translates a compile-time string into code.
Continuing with our `+` example, we end up with 
`V[N] result = mixin("_store[]" ~ "+" ~ "other._store[]")`, which simplifies to 
`V[N] result = _store[] + other._store[])`

The expression `_store[] + other._store[]` is called an "array-wise operation".
It's a concise way of performing an operation between corresponding elements of
two arrays -- in this case, adding each pair of integers into a resulting array.

In many languages, we'd have to separately define `opAdd`, `opSub`, `opMult`,
and more, most of which would likely contain similar code.
Howerver, thanks to the way `opBinary` allows us to work with a string
representation of the operator at compile time, our single `opBinary`
implementation operators like `-` and `\*` as well.

# Summary

I hope you enjoyed learning a little about D!

There is a full implementation of Enumap available 
[here] (https://github.com/rcorre/enumap).

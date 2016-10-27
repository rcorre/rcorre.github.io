# Modelling an RPG in D
For the past year, I've done the majority of my personal coding in a language
called D. 

Its a really cool languagem, and it pains me a bit to get blank looks whenever I
mention it.

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

This could be easily achieved with an 'associative array', which you may know as
a 'Hash Map' or 'Dictionary' in other languages.

In D, an associative array ('AA for short) is a built-in type, declared with the
syntax `ValueType[KeyType]`.

```d
enum Attribute { strength, dexterity, constitution, intellect, wisdom, charisma }
enum Skill     { stealth, perception, diplomacy }
enum Element   { physical, fire, water, air, earth }

struct Character {
  int[Attribute] attributes;
  int[Skill]     skills;
  int[Element]   resistances;
}
```

Ah, that looks much more organized!

Now we can manipulate the character's stats like so:

```d
hero.attributes[Attribute.dexterity] = 2;
if (hero.attributes[Attribute.dexterity] < 4) hero.tripOverOwnFeet();
```

Unfortunately, the above will fail if we explicitly set a value for each
possible key. The is less than ideal -- it would be nice to have a default value
for each possible key.

AA's are a useful tool, but they aren't quite tailored to our situation here.
We don't need to map some arbitrary set of keys; we know exactly how many keys
we have and want exactly one value for each key.

Lets look at how we could use an array for this:

```d
struct Character {
  int[6] attributes;
  // ...
}
```

The syntax `int[6]` declares a **static array** of size 6 (one entry for each
attribute).

In D, static arrays are _value types_ meaning that no heap allocation is
involved in their creation. Furthermore, we are now guaranteed to have a
default value (0) for each Attribute.

Unfortunately, only convention enforces that we use an `Attribute` as a key:

```d
if (hero.attributes[2] < 4) hero.tripOverOwnFeet();
```

What we'd really like is a type specialized for mapping each entry in an enum to
a single value. Such a type often referred to as an `EnumSet`, and is included
in the standard library of some languages such as 
([Java](TODO) and [Rust](TODO)).

The bad news is that Phobos (D's standard library) provides no such type.
The good news is that we're a few lines of code away from something just as
good, and a few more lines away from something better.

# The EnumSet

Here's a simple definition that supports getting and setting values using enum
members as keys. There are a number of things I'm omitting here, some of which I
will discuss later:

```d
import std.traits : EnumMembers;

struct EnumSet(K, V) {
  private enum N = EnumMembers!K.length;
  private V[N] _store;

  auto opIndex(K key)                { return _store[key]; }
  auto opIndexAssign(T value, K key) { return _store[key] = value; }
}
```

Let's break that down line-by-line:

1: `import std.traits : EnumMembers;`

Here we import something we will need later standard library.
We could have said `import std.traits` to import the whole module,
but instead we used a 'scoped import' as we just need `EnumMembers`.

3: `struct EnumSet(K, V) {`
Here, we declare a templated struct. In many other languages,
this would look like `EnumSet<K, V>`.
K will be our key type (the enum) and V will be the value.

K and V are known as 'compile-time parameters'. In this case, they are simply
used to create a generic type, but in D such parameters can be used for a much
more than just generic types, as we will see later.

4: `private enum N = EnumMembers!K.length;`
5: `private V[N] _store;`

Here we leverage `std.traits.EnumMembers`, (which we imported earlier) to
determine how many entries are in the provided enum. We use this to declare a 
static array capable of holding exactly `N` `V`s (valuesvalues)

6: `auto opIndex(K key)                { return _store[key]; }`
7: `auto opIndexAssign(T value, K key) { return _store[key] = value; }`

opIndex is a special method that allows us to provided a custom implementation
of the indexing (`[]`) operator. If a user of the `EnumSet` calls
`skills[Skill.stealth]`, it is translated to `sklls.opIndex(Skill.stealth)`.
Similarly, the assignment `skills[Skill.stealth] = 5` is translated to 
 `sklls.opIndexAssign(Skill.stealth, 5)`

Now lets use that in our character struct:

```d
struct Character {
  EnumSet!(Attribute, int) attributes;
  EnumSet!(Skill    , int) skills;
  EnumSet!(Element  , int) resistances;
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

Wait, never mind.
That wasn't the clever bit.
**This** is the clever bit:

```d
import std.conv : to;
//...

struct EnumSet(K, V) {
  //...
  auto opDispatch(string s)()      { return this[s.to!K]; }
  auto opDispatch(string s)(V val) { return this[s.to!K] = val; }
}
```

```d
if (hero.attributes.charisma < 5) hero.makeAnAwkwardJoke();
```

What I just added was some totally unnecessary, totally awesome syntactic sugar.

I'm leveraging the [opDispatch](TODO) operator to, in a way, overload the `.`
operator.

Here's a quick rundown of what happens for `hero.attributes.charisma = 5`:

1. The compiler sees `attributes.charisma`.
2. It looks for the symbol `charisma` in the type `EnumSet!(Attribute, int)`
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

# A step further: Resistances
As your game progesses you decide to add the concept of armor.
Each piece of armor has a certain set of resistances.

A character wears multiple pieces of armor, and their total resistance is the
sum of the resistance values of their equipment:

```d
struct Armor { EnumSet!(Element, int) resistance; }

enum BodyPart { head, torso, legs, arms }

struct Character {
  EnumSet!(BodyPart, Armor) armor;

  @property auto resistance() {
    EnumSet!(Element, int) res;

    foreach(piece ; armor) res = res + piece.resistance;

    return res;
  }
}
```

See what I did there? Nobody said `EnumSet` only works for numbers; its pretty
convenient for mapping each body part to a piece of armor too!

Anyways, the new `Character.resistance` property relies on two pieces of
functionality not implemented in `EnumSet` yet:

1. It iterates over the elements with a foreach
2. It invokes the `+` operator between `EnumSet` instances

We can support `foreach` over `EnumSet` elements by forwarding `opSlice` to the
underlying store:

```d
auto opSlice() { return _store[]; }
```

Now lets implement `+` using `opBinary`:

```d
  auto opBinary(string op)(typeof(this) other) if (is(op == "+")) {
    T[N] result = _store[] + other._store[];
    return typeof(this)(result);
  }
```

The expression `_store[] + other._store[]` is called an "array-wise operation".
It's a concise way of performing an operation between corresponding elements of
two arrays -- in this case, adding each pair of integers into a resulting array.

If you're new to D, the syntax for overloading the `+` operator probably seems
strange; `opSum` or `opAdd` would be more obvious than `opBinary!"+"`.

However, this syntax allows an awesome level of flexibility. As a matter of
fact, lets leverage that to add support for the `-` operator:

```d
auto opBinary(string op)(typeof(this) other) {
  V[N] result = mixin("_store[] " ~ op ~ " other._store[]");
  return typeof(this)(result);
}
```

`mixin` is a D keyword that translates a compile-time string into code.

We use the concatenation operator `~` to form a compile-time string that places
our operator ("+", "-", "/", ect.) between the two operands, which becomes the
expression that is assigned to `result`.

For example, subtracting two jk



With `opSlice` and `opBinary` implemented, we can now calculate a character's
total resistance. However, lets come back to that method.
By leveraging `std.algorithm`, we can express our intent very concisely:

```d
import std.algorithm : map, reduce;
struct Character {
  // ...

  @property auto resistance() {
    return armor[]              // for each piece of armor
      .map!(x => x.resistance)  // get its resistance value
      .reduce!((a,b) => a + b); // and add them all together
  }
}
```


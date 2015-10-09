---
layout: post
title:  "A tale of two structs"
date:   2015-08-28 13:00:03
categories: coding D
---

I'd like to tell you a story about two structs, Circle and Square:

{% highlight d %}
struct Square {
  float x, y, size;

  float area() { return size * size; }
  float perimeter() { return 4 * size; }
}

struct Circle {
  float x, y, r;

  float area() { return r * r * PI; }
  float perimeter() { return 2 * PI * r; }
}
{% endhighlight %}

Circle and Square are pretty happy being structs; they are lightweight,
stack-allocated value types. They don't mess around with all that
heap-allocated, garbage-collected nonsense of their bretheren, the classes.

However, noticing that classes get to present common functionality using
something called an interface. They are a bit jealous of this, but don't want to
admit it. Rather than convert to the way of the class, they try to come up with
their own way of doing things:

{% highlight d %}
import std.variant;
alias Shape = Algebraic!(Rect, Circle);
{% endhighlight %}

Now, this is pretty exciting, beacuse they can do things like this:

{% highlight d %}
Shape rect = Rect(1,2,3,4);
Shape circ = Circle(0,0,4);
Shape[] shapes = [ rect, circ ];
{% endhighlight %}

But alas, they find themselves disappointed:

{% highlight d %}
auto totalArea = shapes.map!(x => x.area).sum;
{% endhighlight %}

```no property 'area' for type 'VariantN!(16LU, Rect, Circle)'```

What a bummer! Their so-called `Shape` doesn't act like a shape at all!
`Circle` and `Square` press on, seeking their own form of interface:

{% highlight d %}
struct Shape {
  Algebraic!(Rect, Circle) _shape;

  this(T)(T shape) if (is(typeof(_shape = shape))) {
    _shape = shape;
  }

  auto area() {
    return _shape.visit!((Rect r  ) => r.area(),
                         (Circle c) => c.area());
  }

  auto perimeter() {
    return _shape.visit!((Rect r  ) => r.perimeter(),
                         (Circle c) => c.perimeter());
  }
}
{% endhighlight %}

They sit back for a moment and admire their handiwork. This `Shape` truly feels
like an interface. However, it also reeks of boilerplate.
`Circle` and `Square` refine their approach with some mixin sorcery:

{% highlight d %}
auto visitAny(alias fn, V)(ref V var) {
  foreach(T ; V.AllowedTypes) {
    if (auto ptr = var.peek!T) return fn(*ptr);
  }
  assert(0, "No matching type!");
}
{% endhighlight %}

Their makeshift interface starts to feel somewhat cleaner.

{% highlight d %}
auto area() {
  return _shape.visitAny!(x => x.area());
}

auto perimeter() {
  return _shape.visitAny!(x => x.perimeter());
}
{% endhighlight %}

Still, they are not satisfied. They turn to `opDispatch`, a notorious source of
compile-time shenanigans:

{% highlight d %}
struct Some(T...) {
  Algebraic!T _value;

  this(V)(V value) if (is(typeof(_value = value))) {
    _value = value;
  }

  auto opDispatch(string op, Args...)(Args args) {
    return _value.visitAny!(x => mixin("x." ~ op ~ "(args)"));
  }
}
{% endhighlight %}

Now our humble structs truly have their own interface!

{% highlight d %}
alias Shape = Some!(Rect, Circle);
Shape rect = Rect(1,2,3,4);
assert(rect.area == 12);
{% endhighlight %}

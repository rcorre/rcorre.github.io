---
layout: post
title:  "Stack your States"
date:   2015-08-28 13:00:03
categories: coding gamedev D
---

## Intro
My last two game-development projects were turn-based strategy games.
Both ended up abandoned long before completion, but one particular technique
used in both stood out as being extremely useful: the `StateStack`.

While turn-based games can avoid many of the difficulties that plague real-time
games (physics, networking, tighter performance constraints), they often involve
quite complex game-logic flows. Trying to manage this with a smattering of
state variables and conditionals can quickly become overwhelming.

I'd like to share the `StateStack` approach I took to managing this complexity
in both games.  While a number of poor design decisions lead to each game's
abandonment, the `StateStack` is one technique that proved invaluable in each,
and continues to be useful in my current projects.

I'll start with a basic implementation, then discuss some useful additions I'm
currently using, and finally move on to some musings about how it could be
implemented differently (better?).

If you find this kind of thing interesting, I highly recommend the book
'Programming Game AI by Example', which inspired many of my ideas on this topic.

## What is a State?
A state is a structure that dictates the behavior of some entity in your game.
The meaning of 'entity' here is extremely broad.
It could be anything from a single enemy to the entire game itself.
The former might have states like `SeekPlayer`, and `RunAway`, while the latter
might have states like `TitleScreen` and `PlayMatch`.

The representation of a state might look something like this:

{% highlight d %}
interface State(T) {
  void enter(T);
  void exit(T);
  void run(T);
}
{% endhighlight %}

At any given time, an entity has a single active state. During each update loop
of the game, the `run` method of the active state is executed, and it is passed
the object it is to operate on. For example, the aforementioned `SeekPlayer`
state might direct the subjects movement towards the player's current position
each update.

The `enter` method is called whenever a state becomes active, before the first
call to `run`.
This allows the state to perform any preparation it needs before
it begins its normal flow of logic.
Similarly, `exit` allows a state to perform some sort of tear-down before it
becomes inactive.
Using the `TitleScreen` game state example from before,
`TitleScreen.enter` might initiate a transition that slides the title menu into
place, while `TitleScreen.exit` might trigger a fade-out.

Note that `enter` and `exit` are _not_ equivalent to a constructor and
destructor; we will see later that a single state may `enter` and `exit`
multiple times during its life.

## What is a StateStack?
The state structure mentioned before isn't terribly ingenious on its own -- the
`StateStack` is what makes everything tick.

The `StateStack` itself isn't really that ingenious either; as a matter of fact,
its pretty much exactly what it sounds like, and only needs to support 3
operations:

- `push` : place a state on top of the stack.
- `pop`  : remove the state on top of the stack.
- `run`  : cause the state on top of the stack to process its object.

The only bit of trickiness comes in managing those `enter` and `exit` states
mentioned earlier. The state stack must ensure that the following happens during
a state transition:

- call `enter` once before calling `run` on a state that was previously inactive.
- call `exit` on a state that becomes inactive
- `enter` and `exit` should be called an equal number of times during a state's
  life

We could make `State` a class rather than an interface and stick a
`bool isActive` member on it to track whether each state has been entered.

However, since the `StateStack` only allows a single active state at once, we
can actually track this with a single flag in the `StateStack`.
Let's take a look at a stripped-down implementation:

{% highlight d %}
struct StateStack(T) {
  private {
    bool        _currentStateEntered;
    SList!State _stack;
    T           _subject;

    @property auto top()   { return _stack.front; }
    @property bool empty() { return _stack.empty; }
  }

  void push(State!T state) {
    if (_currentStateEntered) {
      // we had a state that was previously entered()
      // our first order of business is to exit()
      top.exit(_subject);

      // our new top state has no longer been entered(), so unset this flag
      _currentStateEntered = false;
    }

    // note that we push the new state, but do _not_ call enter() yet
    // if multiple states are pushed at once, we only want to enter() the one
    // that ends up getting run during the next pass.
    _stack.insertFront(state);
  }

  void pop() {
    // get ref to current state, top may change during exit
    auto state = top;
    _stack.removeFront;

    if (_currentStateEntered) {
      // the state we are popping had been entered(), so we need to exit() it
      _currentStateEntered = false;
      state.exit(_params);
    }
  }

  void run(T subject) {
    // cache obj for calls to exit() that are triggered by pop().
    _subject = subject;

    // top.enter() could push/pop, so keep going until the top state is entered
    while (!_currentStateEntered) {
      _currentStateEntered = true;
      top.enter(params);
    }

    // finally, our stack has stabilized
    top.run(params);
  }
}
{% endhighlight %}

It may not be rocket science, but it is quite a bit more complicated than I
imagined when I first set out to implement this.

Much of the complexity comes from the fact that it is valid (and useful) for a
state to push() and pop() states during its `enter`.
It would be lovely to implement `StateStack.run` like so:

{% highlight d %}
if (!_currentStateEntered) {
  _currentStateEntered = true;
  top.enter(params);
}
top.run(params);
{% endhighlight %}

However, this is too naive; after we call `top.enter`, `top` may be a completely
different state!

This is the cause for the while loop seen in `StateStack.run`.

Similarly, a state may `push/pop` states during its `exit` call.
While I find this much less useful, I decided to support it just to avoid nasty
surprises. To support this, we need to cache the subject that is passed into
`run` so it can be used by `pop`.

## Dissolving Complex Logic Flows
In my previous game [Terra Arcana](https://github.com/rcorre/terra-arcana), the
`StateStack` proved invaluable in making the flow of combat manageable.

Here's a quick description of the rules regarding attacks:

- The attacker launches one or more strikes against the defender
- Each strike may hit (dealing damage or some effect) or miss
- If the defender's health has dropped to 0, they are destroyed
- The defender may get a chance to counter-attack if:
  - They were not destroyed by the initial attack
  - They have an attack that is in range of the attacker
  - They have enough AP (action points) to use said attack
- The counter-attack, like the initial attack, may have multiple strikes
- The counter-attack may destroy the attacker
- You cannot counter-attack a counter-attack

Now consider that an AOE attack may hit multiple defenders, each of which gets a
chance to counter-attack!

Now, computing the _result_ of this isn't so bad -- you can probably imagine a
series of if/else statements that could do the job in a single pass.

The difficulty comes in controlling the sounds and animations that depict this
event to the player. We need to play animations for attacks and unit
destruction, pop up text to indicate damage and status effects (or lack thereof),
manipulate health/AP bars on the ui, and play sound effects at various points
throughout the process.

This all happens over the course of multiple update cycles rather than a single
function call, so managing it with a single function would involve a whole mess
of state variables (`strikesLeft`, `isStrikeAnimating`, `isDefenderDestroyed`,
`isCounterAttackInProgress`, ect.).

Using the `StateStack` means we can separate each chunk of logic (applying an
attack effect, destroying a unit, initiating a counter attack) into its own
independent state, push a whole bunch of these onto the stack at once, and then
let everything play out.

Here's an excerpt of code from Terra Arcana that handles the initiation of an
attack action (note that `b` stands for 'battle', I guess I was feeling
particularly lazy with my variable names at the time):

{% highlight d %}
b.states.popState();

foreach(unit ; unitsAffected) {
  b.states.pushState(new PerformCounter(unit, _actor));
}

foreach(unit ; unitsAffected) {
  b.states.pushState(new CheckUnitDestruction(unit));

  for(int i = 0 ; i < _action.hits ; i++) {
    b.states.pushState(new ApplyEffect(_action, unit));
  }
}
{% endhighlight %}

Remember that we are dealing with a stack, so states pushed later end up at the
top (`ApplyEffect` happens before `CheckUnitDestruction`).

This logic resides in the `PerformAction` state, so the first call removes this
state from the stack before pushing the rest on.

To understand this a bit better, consider the following scenario:
A unit launches an attack that hits twice. The target is not destroyed, and is
capable of countering with an attack that hits 3 times.
The states on the stack would progress like so:

PerformAction
PerformCounter | CheckUnitDestruction | ApplyEffect | ApplyEffect
PerformCounter | CheckUnitDestruction | ApplyEffect
PerformCounter | CheckUnitDestruction
PerformCounter
CheckUnitDestruction | ApplyEffect | ApplyEffect | ApplyEffect
CheckUnitDestruction | ApplyEffect | ApplyEffect
CheckUnitDestruction | ApplyEffect
CheckUnitDestruction

Note that when `PerformCounter` becomes the active state, it replaces itself
with 3 `ApplyEffects` and a `CheckUnitDestruction`. The states nicely
encapsulate specific chunks of game logic, so we get to reuse the same states in
`PerformAction` and `PerformCounter`.

## Isolation
Another advantage of states is the ability to isolate chunks of game logic from
one another. My [current project](https://github.com/rcorre/damage-control)
is a game reminiscent of the SNES title Rampart.
In it, a match is divided into rounds, and each round passes through a series of
phases. First you place some turrets in your territory, then you fire at your
opponent, and then you try to repair the damage done during the firing phase.
There are some transitional states in-between and a score summary at the end of
each round.

Here's the logic that sets up a new round:

{% highlight d %}
battle.states.push(
    new BattleIntroduction(cannonsTitle, MusicLevel.moderate, battle.game),
    new PlaceTurrets(battle),

    new BattleIntroduction(fightTitle, MusicLevel.intense, battle.game),
    new FightAI(battle, _currentRound),

    new BattleIntroduction(rebuildTitle, MusicLevel.basic, battle.game),
    new PlaceWalls(battle),

    new StatsSummary(battle, _currentRound));

++_currentRound;
{% endhighlight %}

Because all of the states can be stacked up at once within a single function,
none of the states have to be aware what state comes next.
For example, `PlaceWalls` doesn't have to know to show a stats summary when it
ends; it just pops itself off the stack when done and lets the next state kick
in.

This logic resides in the `StartRound` state, which sits at the bottom of the
state stack. Once all the phases for the current round are popped, we once again
`enter` the `StartRound` state, which increments `_currentRound` and pushes a
new set of states.

Those of you still paying attention may notice that I pushed the states in a
single call, in the order in which they occur (rather than pushing one at a time
in reverse order).

This is one of those useful extensions to `StateStack` I mentioned earlier.

My current implementation actually defines `push` like so:

{% highlight d %}
void push(State!T[] states ...) {
  if (_currentStateEntered) {
    top.exit(_params);
    _currentStateEntered = false;
  }

  foreach_reverse(state ; states) {
    _stack.insertFront(state);
  }
{% endhighlight %}

Admittedly, this could be slightly non-obvious compared to pushing states one at
a time in reverse order, but I find it much easier to visualize the flow of
logic when pushing multiple states this way.

## Hierarchy
The above code resides in the `enter` method of `StartRound`, which is of type
`State!Battle`. So, `StartRound` is a 'Battle State', but what is a `Battle`?

{% highlight d %}
class Battle : State!Game { ... }
{% endhighlight %}

This provides a nice way of dividing you game logic into tiers.
`Game` has very general states like `TitleScreen` and `Battle`, and each of
those states has a more specific set of states.
Within the `Battle` state, each enemy has its own `StateStack` to manage things
like hovering, circling targets, shooting, and crashing after being hit.

## Improvements
As I've worked with this structure over the course of 3 games (admittedly, 2
abandoned and 1 in-progress), I've gradually refined and added functionality to
the `State Stack`. Here are a few useful additions:

### Push(...)
As mentioned earlier, I modified push to take multiple arguments which are
pushed in reverse order. This allows the caller to list states in the order in
which they occur, which I find easier to read.

### Replace
This one is a gimme. It is quite common for one state to simply pop itself and
push a single other state. Replace is a convenience wrapper to do just that:

{% highlight d %}
void replace(State!T state) {
  if (!_stack.empty) pop();
  push(state);
}
{% endhighlight %}

### Context, Please!
Sometimes, a state might rely on information that the entity itself doesn't have
access to. Consider the following:

{% highlight d %}
class SeekPlayer : State!Enemy {
  void run(Enemy self) {
    // what is playerPos?
    self.moveToward(playerPos);
  }
}
{% endhighlight %}

The state operates on an `Enemy`, but doesn't have all of the information it
needs to behave intelligently. We could place this information on the `Enemy`
itself, but another option is to allow a `State` to take multiple arguments.

{% highlight d %}
interface State(T...)
{% endhighlight %}

Thanks to D's tuple mechanics, this is actually the only change required!
Everything else stays the same, but we can now define states to operate on
multiple args:

{% highlight d %}
struct EnemyContext {
  Vector2f playerPos;
  // other useful info for updating an enemy goes here
}

class SeekPlayer : State!(Enemy, EnemyContext) {
  void run(Enemy self, EnemyContext context) {
    self.moveTowards(context.playerPos);
  }
}
{% endhighlight %}

## Different Approaches
The approach I just described has worked pretty well for me at a macro scale
(describing the flow of logic through various game states and sub-states).

However, how well would it scale if every member of a group of 1000s of entities
were to have its own `StateStack`, each with a constantly churning set of states?
Can we use this technique without the bloat of allocating a new class instance
for every state?

Here are a few approaches that have been bouncing around in my head:

### The minimalist
The nice bit about using inheritance is that every type of `State` can maintain
its own set of data. Consider some states from 'Damage Control':

{% highlight d %}
class StartRound : State!Battle {
  private int _currentRound;
  // ...
}

// note: TimedPhase is a State!Battle
abstract class Fight : TimedPhase {
  private {
    // ...
    private ProjectileList _projectiles;
    private ExplosionList  _explosions;
    private ParticleList   _particles;
  }
  // ...
}
{% endhighlight %}

At the time of writing these, it made sense to keep these values in their
respective states -- after all, projectiles are only flying around in the
`Fight` state, and so on.

However, do these values _really_ belong in the states themselves?
Or are they really members of the subject the states operate on (in this case,
the `Battle`)?

If we assume that the only context a `State` needs is that which is passed to its
`enter`/`exit`/`run` methods, we can reduce a `State` to a bundle of function
pointers:

{% highlight d %}
struct State(T...) { void function(T) enter, exit, run; }
{% endhighlight %}

Now a state is defined by sticking a few function pointers to a struct:

{% highlight d %}
auto seekPlayer() {
  State!Enemy state;
  state.enter = (self, context) { }
  state.exit  = (self, context) { }
  state.run   = (self, context) { self.moveToward(context.playerPos); }
  return state;
}

enemy.states.push(seekPlayer());
{% endhighlight %}
The nice bit about this approach is that it fits nicely into more minimal
languages like C.

Now that we updated our definition of `State`, lets look at how we need to
change `StateStack` to support this.

Oh, wait. We don't! `StateStack` can remain exactly the same.

It will fail if we leave a null function pointer, but we can avoid that with
some do-nothing defaults:

{% highlight d %}
auto doNothing(T...)(T) { }

struct State(T...) {
  void function(T) enter = &doNothing!T;
  void function(T) exit  = &doNothing!T;
  void function(T) run   = &doNothing!T;
}
{% endhighlight %}

We could also come to a compromise between the class and struct approaches by
using delegates:

{% highlight d %}
struct State(T...) { void delegate(T) enter, exit, run; }

auto moveToPosition(Vector2f pos) {
  State!Enemy state;
  state.enter = (self, context) { }
  state.exit = (self, context) { }
  state.run = (self, context) { self.moveToward(pos); }
  return state;
}

enemy.states.push(moveToPosition(Vector2f(120, 240)));
{% endhighlight %}

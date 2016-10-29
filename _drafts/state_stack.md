# Intro

Structuring the flow of logic in a game can be challenging.  If you're not
careful, you quickly end up with a scattered collection of state variables and
conditionals that is difficult to wrap your head around.

In my past two game projects, I found it helpful to structure my game flow as a
stack of states. In this article, I'll give a quick overview of this technique
and some examples of what makes it useful. The example code is written in
[D](http://dlang.org/), but it should be pretty easy to apply in any language.

# Stacking States for Isolation
Stacking states provides a nice way isolate chunks of game logic from
one another. I leveraged this while making
[damage-control](https://github.com/rcorre/damage-control), a game reminiscent
of the SNES title Rampart.

In it, a match is divided into rounds, and each round passes through a series of
phases. First you place some turrets in your territory, then you fire at your
opponent, and then you try to repair the damage done during the firing phase.
Before each phase, a banner scrolls across the screen telling the player what
phase they are in.

Here's the logic that sets up a new round (simplified from the original source
for clarity:

```d
game.states.push(
    new ShowBanner("Place Turrets", game),
    new PlaceTurrets(game),

    new ShowBanner("Fire!", game),
    new Fire(game, _currentRound),

    new ShowBanner("Rebuild!", game),
    new PlaceWalls(game),

    new StatsSummary(game));
```

Because all of the states can be stacked up at once within a single function,
none of the states have to be aware what state comes next.

For example, `PlaceWalls` doesn't have to know to show a stats summary when it
ends; it just pops itself off the stack when done and lets the next state kick
in.

The code shown above resides in the `StartRound` state, which sits at the bottom
of the state stack. Once all the phases for the current round are popped, we
once again enter `StartRound` and push a new set of states.

The flow of states looks like this (the right side represents the top of the
stack, or the active state):

```
StartRound
StartRound | StatsSummary | PlaceWalls | ShowBanner | Fire | ShowBanner | PlaceTurrets | ShowBanner
StartRound | StatsSummary | PlaceWalls | ShowBanner | Fire | ShowBanner | PlaceTurrets
StartRound | StatsSummary | PlaceWalls | ShowBanner | Fire | ShowBanner
StartRound | StatsSummary | PlaceWalls | ShowBanner | Fire
StartRound | StatsSummary | PlaceWalls | ShowBanner
StartRound | StatsSummary | PlaceWalls
StartRound | StatsSummary
StartRound
StartRound | StatsSummary | PlaceWalls | ShowBanner | Fire | ShowBanner | PlaceTurrets | ShowBanner
... and so on ...
```

I'll provide another example at the end of the article, but first I'll discuss
the implementation.

# The State

```d
interface State(T) {
  void enter(T);
  void exit(T);
  void run(T);
}
```

`T` is a generic type here, and represents whatever kind of object the states
will operate on. For example, it might be a `Game` object that provides access
to game entities, resources, input devices, and more.

At any given time, your has a single active state. `run` is executed once for
each update loop of the game.

`enter` is called whenever a state becomes active, before the first call to
`run`. This allows the state to perform any preparation it needs before it
begins its normal flow of logic.  Similarly, `exit` allows a state to perform
some sort of tear-down before it becomes inactive.  Note that `enter` and `exit`
are _not_ equivalent to a constructor and destructor; we will see later that a
single state may `enter` and `exit` multiple times during its life.

As an example, lets take the `PlaceWalls` state from earlier. `enter` might
start a timer for how long the state should last, `run` would process input from
the player to move and place pieces, and `exit` would mark off areas that the
player had enclosed.

# The Stack

The `StateStack` itself is pretty straightforward as well.
It only needs to support 3 operations:

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

```d
struct StateStack(T) {
  private {
    bool        _entered;
    SList!State _stack;
    T           _object;
  }

  void push(State!T[] states ...) {
    if (_entered) {
      // we had a state that was previously entered()
      // our first order of business is to exit()
      _stack.top.exit(_object);

      // our new top state has no longer been entered(), so unset this flag
      _entered = false;
    }

    // note that we push the new state, but do _not_ call enter() yet
    // if multiple states are pushed at once, we only want to enter() the one
    // that ends up getting run during the next pass.
    foreach_reverse(state ; states) {
      _stack.insertFront(state);
    }
  }

  void pop() {
    // get ref to current state, top may change during exit
    auto popped = _stack.top;
    _stack.removeFront;

    if (_entered) {
      // the state we are popping had been entered, so we need to exit it
      _entered = false;
      popped.exit(_object);
    }
  }

  void run(T obj) {
    // cache obj for calls to exit() that are triggered by pop().
    _object = obj;

    // top.enter() could push/pop, so keep going until the top state is entered
    while (!_entered) {
      _entered = true;
      top.enter(obj);
    }

    // finally, our stack has stabilized
    top.run(obj);
  }
}
```

The implementation is mostly straightforward, but there are a few caveats.

It is valid (and useful) for a state to push() and pop() states during its
`enter`. In the previous example, `StartRound` pushes a number of states during
`enter`. Therefore, implementing `StateStack.run` like so would be incorrect:

```d
if (!_entered) {
  _entered = true;
  top.enter(obj);
}
top.run(obj);
```

After pushing `StartRound` and calling `StateStack.run`, it could call
`StartRound.enter`, which would push more states onto the stack. It would then
call `top.run(obj)` on whatever state was last pushed, which hasn't been entered
yet!

For this reason, `run` uses the `while (!_entered)` loop to call `enter` until
the stack 'stabilizes'

Similarly, a state may `push` or `pop` states during its `exit` call.
To support this, we need to cache the object that get passed in
to `run` so it can be used by `pop`.

# Dissolving Complex Logic Flows

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

The difficulty depicting the result to the player. We need to play animations
for attacks and unit destruction, pop up text to indicate damage and status
effects (or lack thereof), manipulate health/AP bars on the ui, and play
sound effects at various points throughout the process.

This all happens over the course of multiple update cycles rather than a single
function call, so managing it with a single function would involve a whole mess
of state variables (`attackCount`, `isAnimating`, `isDefenderDestroyed`,
`isCounterAttackInProgress`, ect.).

With a `StateStack` we can separate chunks of logic like applying damage or
status effects, destroying a unit, and initiating a counter attack into its own
independent state. When an attack begins, you push a whole bunch of these onto
the stack at once, and then let everything play out.

Here's an excerpt of code from that initiates an attack:

```d
battle.states.popState();

foreach(unit ; unitsAffected) {
    battle.states.push(new PerformCounter(unit, _actor));
}

foreach(unit ; unitsAffected) {
    battle.states.push(new CheckUnitDestruction(unit));

    for(int i = 0 ; i < _action.hits ; i++) {
        battle.states.push(new ApplyEffect(_action, unit));
    }
}
```

Remember that we are dealing with a stack, so states pushed later end up at the
top (`ApplyEffect` happens before `CheckUnitDestruction`).

This logic resides in the `PerformAction` state, so the first call removes this
state from the stack before pushing the rest on.

To understand this a bit better, consider the following scenario:
A unit launches an attack that hits twice. The target is not destroyed, and is
capable of countering with an attack that hits 3 times.
The states on the stack would progress like so (where the right side represents
the top of the stack):

```
PerformAction
PerformCounter | CheckUnitDestruction | ApplyEffect | ApplyEffect
PerformCounter | CheckUnitDestruction | ApplyEffect
PerformCounter | CheckUnitDestruction
PerformCounter
CheckUnitDestruction | ApplyEffect | ApplyEffect | ApplyEffect
CheckUnitDestruction | ApplyEffect | ApplyEffect
CheckUnitDestruction | ApplyEffect
CheckUnitDestruction
```

Note that when `PerformCounter` becomes the active state, it replaces itself
with 3 `ApplyEffects` and a `CheckUnitDestruction`. The states nicely
encapsulate specific chunks of game logic, so we get to reuse the same states in
`PerformAction` and `PerformCounter`.

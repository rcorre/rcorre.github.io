## Intro
My last two game projects were turn-based strategy games.
While turn-based games can avoid many of the difficulties that plague real-time
games (physics, networking, tighter performance constraints), they often involve
quite complex game-logic flows. Trying to manage this with a smattering of
state variables and conditionals can quickly become overwhelming.

Thinking about the game flow in terms of a stack of states made organizing this
logic manageable.

If you find this kind of thing interesting, I highly recommend the book
'Programming Game AI by Example', which inspired many of my ideas on this topic.

## The State

```d
interface State(T) {
  void enter(T);
  void exit(T);
  void run(T);
}
```

At any given time, your has a single active state. `run` is executed once for
each update loop of the game.

`enter` is called whenever a state becomes active, before the first call to
`run`. This allows the state to perform any preparation it needs before it
begins its normal flow of logic.  Similarly, `exit` allows a state to perform
some sort of tear-down before it becomes inactive.

Using the `TitleScreen` game state example from before,
`TitleScreen.enter` might initiate a transition that slides the title menu into
place, while `TitleScreen.exit` might trigger a fade-out.

Note that `enter` and `exit` are _not_ equivalent to a constructor and
destructor; we will see later that a single state may `enter` and `exit`
multiple times during its life.

## The Stack
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

```d
struct StateStack(T...) {
  private {
    bool        _entered;
    SList!State _stack;
    T           _params;
  }

  void push(State!T[] states ...) {
    if (_entered) {
      // we had a state that was previously entered()
      // our first order of business is to exit()
      _stack.top.exit(_params);

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
      popped.exit(_params);
    }
  }

  void run(T params) {
    // cache params for calls to exit() that are triggered by pop().
    _params = params;

    // top.enter() could push/pop, so keep going until the top state is entered
    while (!_entered) {
      _entered = true;
      top.enter(params);
    }

    // finally, our stack has stabilized
    top.run(params);
  }
}
```

It may not be rocket science, but it is quite a bit more complicated than I
imagined when I first set out to implement this.

Much of the complexity comes from the fact that it is valid (and useful) for a
state to push() and pop() states during its `enter`.

It would be lovely to implement `StateStack.run` like so:

```d
if (!_entered) {
  _entered = true;
  top.enter(params);
}
top.run(params);
```

However, this is too naive; after we call `top.enter`, `top` may be a completely
different state!

This is the cause for the while loop seen in `StateStack.run`.

Similarly, a state may `push/pop` states during its `exit` call.
While I find this much less useful, I decided to support it just to avoid nasty
surprises. To support this, we need to cache the parameters that get passed in
to `run` so it can be used by `pop`.

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

Here's an excerpt of code from Terra Arcana that initiates an attack action:

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

## Isolation
Stacking states provides a nice way isolate chunks of game logic from
one another. I leveraged this while making
[damage-control](https://github.com/rcorre/damage-control), a game reminiscent
of the SNES title Rampart.

In it, a match is divided into rounds, and each round passes through a series of
phases. First you place some turrets in your territory, then you fire at your
opponent, and then you try to repair the damage done during the firing phase.
There are some transitional states (called `BattleIntroduction`s) between and a
score summary at the end of each round.

Here's the logic that sets up a new round:

```d
battle.states.push(
    new BattleIntroduction(cannonsTitle, MusicLevel.moderate, battle.game),
    new PlaceTurrets(battle),

    new BattleIntroduction(fightTitle, MusicLevel.intense, battle.game),
    new FightAI(battle, _currentRound),

    new BattleIntroduction(rebuildTitle, MusicLevel.basic, battle.game),
    new PlaceWalls(battle),

    new StatsSummary(battle, _currentRound));
```

Because all of the states can be stacked up at once within a single function,
none of the states have to be aware what state comes next.

For example, `PlaceWalls` doesn't have to know to show a stats summary when it
ends; it just pops itself off the stack when done and lets the next state kick
in.

This logic resides in the `StartRound` state, which sits at the bottom of the
state stack. Once all the phases for the current round are popped, we once again
enter `StartRound` and push a new set of states.

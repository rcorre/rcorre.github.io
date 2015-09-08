---
layout: default
title: Projects
permalink: /projects/
---

# Damage Control

**State**  : Pre-Alpha (Alpha coming soon!)

**Source** : [On Github](https://github.com/rcorre/damage_control)

Damage control is a game reminiscent of one of my old favorite titles:
[Rampart](https://en.wikipedia.org/wiki/Rampart_(video_game)).

The gameplay involves two players squaring off, each destroying the others walls
and then trying frantically to repair their own.

The mechanics are pretty simple, yet the strategy can get surprisingly deep.
Currently, it only supports single-player (where AI drones destroy your walls),
but I plan on adding multiplayer (both local and remote!) eventually, as the
competitive aspect is where it really shines.

The game is written in [D](http://dlang.org/) and uses the
[Allegro5](http://liballeg.org/) game library.

# DTiled

**State**  : v0.2

**Source** : [On Github](https://github.com/rcorre/dtiled)

DTiled is a library for creating tile-mapped games in D.

Over the course of working on several turn-based stragegy/rpg games, I noticed I
was repeating a lot of the same logic over and over. Regardless of the game's
mechanics, things like mapping between screen coordinates and grid coordinates,
iterating through groups of tiles, loading a map from a file, and figuring out
what tiles were covered by a certain pattern (e.g. an area-of-effect attack).

DTiled aims to consolidate this logic into a game-engine-agnostic library.
It also has built-in support for loading maps created with
[Tiled](http://www.mapeditor.org/).
It currently only supports orthogonal maps, but isometric and hexagonal map
support is planned.

I am currently using DTiled in Damage Control, and will try to release a stable
version of the library once the game is finished.

# Terra Arcana

**State**  : Beta (Abandoned)

**Source** : [On Github](https://github.com/rcorre/terra-arcana)

Terra Arcana is a sci-fi turn-based strategy game that I worked on for a good
portion of 2014.

I have not _entirely_ given up on it, but I simply haven't found the time to
work on it in the midst of other things demanding my attention.

There are playable demos for Linux and Windows under the **Releases** section of
the github repo.

The game is written in [D](http://dlang.org/) and uses the
[Allegro5](http://liballeg.org/) game library.

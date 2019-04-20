---
layout: post
title:  "Reactive Game of Life with Akka"
date:   2019-04-19 18:56:42 +0200
categories: akka actors reactive
---

Quite for some time I was thinking about refreshing Akka skills and providing some simple demo of what one can do with Akka and [Actor Model][actor-model]. Finally, the ~~winter~~ time has come, I will show the [Conway's Game of Life][game-of-life] reactive implementation with [Akka Actors][akka-actors].

TL;DR: [code][the-code]

The rules are very simple, the current state of a cell on a two-dimentional grid depends only on previous state of the call and a number of "alive" cells among surrounding 8 neighbors cells (immediate vertical, horizontal and diagonal adjacent cells). The implementation in Scala looks like this:

```scala
object Cell {
    def apply(alive: Boolean, around: Int): Boolean = {
        assert(around >= 0)
        (alive, around) match {
            case (true, n) if n < 2 => false
            case (true, n) if n == 2 || n == 3 => true
            case (true, n) if n > 3 => false
            case (false, n) if n == 3 => true
            case _ => false
        }
    }
}
```

The initial idea I had was to keep each cell's state in a dedicated actor, thus allowing "reactive" and consistent transitions to the next state. The only events happening are cells becoming "alive" or "empty", thus once cell becomes "alive", the surrounding 8 cells receive a message that number of neighbors increased, and respectfully, when cell becomes "empty" - decreased. Scaling this out (to suppot grids that don't fit single instance) might be tricky, as messages between adjacent cells must still be able to find their way to addressed cell. [Location transparency][location-transparency] FTW.

For this post, I will limit the scope to small grids, but the code doesn't need to be changed much for a distributed case, *just* using [Akka Remoting][akka-remoting] looks enough. The distributed option will remain on my to-do list, as it might be interesting to run. Tricky to visualize/observe though - how to observe million by million grid? It's 1E12 cells, but it might be interesting to play with some statistics about it.

Back to small grids, but keep scaling out in mind. I need to split a grid in chunks, perfect use case for a [Quad Tree][quad-tree]. The quad tree nodes are *forks* and *leaves*, with *forks* keeping list of quads that belongs a level down, and *leaves* keeping the sub-grid of cells and mapping from cell's position to a `CellActor`. So for sub-quad border-line cells, some adjacent cells lay in different quads, and it makes sense to forward updates to such "remote" cells via parent quad tree node. Then quad tree node can decide if an update should go to respective quad or go to a higher level of a quad tree. Eventually such event will land respective cell, and the longet such path (for central 4 cells) will go up to the quad tree root and then down to respective cell through quad tree nodes. It scales though, and each actor keeps doing their thing: `CellActor` stores state and handles updates from neighbors, `GridActor` stores either split quads or references to actual cells and handles routing of updates for "remote" cells.

This is all very nice, but how do I actually *see* what is really happening on the grid? I isolated 2 abstractions: `Contol` to let user provide some input (click on specific cell) and `Display`, that is responsible for rendering specific cell in a given state. I believe the signatures below are self-explanatory enough:

```scala
// The position of the cell on the grid
case class Pos(row: Int, col: Int)

trait Control {
  def click(pos: Pos): Unit
  def terminate(): Unit
}

trait Display {
  def fill(pos: Pos): Unit
  def clear(pos: Pos): Unit
}
```

These two abstractions are more than enough to keep the show going! The actuall execution of input and state updates is performed in scope of `DisplayActor`, as this is essentially all what can happen: user can click or a cell gets rendered on a screen.

The actual rendering is implemented via [SimGraf][ui-lib], nice and small UI library (built on top of Akka as well). The result looks like this:

![Conway's Game of Life]({{ site.url }}/assets/2019-04-19-akka-reactive-game-of-life/grid.png)

The unit-tests of the [code][the-code] are left as an excercise for a reader.

[game-of-life]: https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life
[actor-model]: https://en.wikipedia.org/wiki/Actor_model
[akka-actors]: https://doc.akka.io/docs/akka/current/actors.html
[location-transparency]: https://doc.akka.io/docs/akka/current/general/remoting.html
[quad-tree]: https://en.wikipedia.org/wiki/Quadtree
[the-code]: https://github.com/sergey-melnychuk/akka-reactive-game-of-life
[akka-remoting]: https://doc.akka.io/docs/akka/current/remoting.html
[ui-lib]: https://gitlab.com/h2b/SimGraf


---
layout: post
title:  "Reactive Game of Life with Akka"
date:   2019-04-19 18:56:42 +0200
categories: akka actors reactive
---

Quite for some time I was thinking about refreshing Akka skills and provide some simple demo of what one can do with Akka and [Actor Model][actor-model]. Finally, the ~~winter~~ time has come, I will show the [Conway's Game of Life][game-of-life] reactive imlementation with [Akka Actors][akka-actors].

TL;DR: [code][the-code]

The rules are very simple, state of a cell on a two-dimentional grid depends only on number of "alive" cells amongh surrounding 8 neighbors cells (immediate vertical, horizontal and diagonal adjacent cells). The implementation of Scala looks like this:

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

The initial idea I had is to keep each cell's state in a dedicated actor. The only events happening are cells becoming "alive" or "empty", thus once cell becomes "alive", the surrounding 8 cells receive a message that number of neighbors increased, and respectfully, when cell becomes "empty" - decreased. Scaling this out might be tricky for the case when the grid is very large (doesn't fit single node), as messages between adjacent cells must still be able to find their way to required cell. Did someone just say [location transparency][location-transparency]?

For this post, I will limit the scope to small grids, but the code doesn't need to be changed much for a distributed case, just using [Akka Remoting][akka-remoting] looks enough. The distributed option will remain on my to-do list, as it might be interesting to run. Tricky to visualize/observe though - how to observe million by million grid? It's 1E12 cells, but it might be interesting to play with some statistics about it.

Back to small grids, but keep scaling out in mind. I need to split a grid in chunks, perfect use case for a [Quad Tree][quad-tree]. The quads tree nodes are forks and leaves, with forks keeping either list of quads below and leaves keeping the sub-grid of cells. So for sub-quad border-line cells, some adjacent cells lay in different quads, so it makes sense to forward updates to such "remote" cells via path of quads up, and such will even touch root of the tree for the 4 central cells of the grid. It scales though, and each actor keeps doing their thing: `CellActor` stores state, `GridActor` stores either quads or references to actual cells.

This is all very nice, but how do I actually *see* what is really happening on the Grid? So I isolated 2 abstractions: `Contol` to let user provide some input (click on specific cell to) and `Display`, to actually change cell state. I believe the signatures below are self-explanatory enough:

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

And those thow abstractions are more than enough to keep the show going! The actuall execution of input and state updates is performed in scope of `DisplayActor`, as this is essentially all what can happen: user can click, cell can flip.

The unit-tests of the [code][the-code] are left as an excercise for a reader.

[game-of-life]: https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life
[actor-model]: https://en.wikipedia.org/wiki/Actor_model
[akka-actors]: https://doc.akka.io/docs/akka/current/actors.html
[location-transparency]: https://doc.akka.io/docs/akka/current/general/remoting.html
[quad-tree]: https://en.wikipedia.org/wiki/Quadtree
[the-code]: https://github.com/sergey-melnychuk/akka-reactive-game-of-life
[akka-remoting]: https://doc.akka.io/docs/akka/current/remoting.html


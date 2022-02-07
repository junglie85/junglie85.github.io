---
title: "Game Programming Maths Cheat Sheet"
date: 2022-01-26T20:52:10Z
draft: false
tags: ["games", "maths"]
categories: []
---

I don't use these maths functions often enough for them to stick in my head when I need them. I'm
developing this cheat sheet to help me have a handy reference when needed.

## Linear colour blending for RGB colours

Linear colour blending for RGB colours uses linear interpolation between each colour.
To blend colour A (`fg`) onto colour B (`bg`) using colour A's alpha channel value (`a`):

> _blended colour = bg + (fg - bg) * a_

Or with the same formula rearranged:

> _blended colour = bg * (1.0 - a) + fg * a_

For example, assuming two colours A (255, 0, 0, 102) and B (0, 255, 255, 255) in RGBA format):

> a = 102 / 255 = 0.4
>
> red   = 0   * (1.0 - 0.4) + 255 * 0.4 = 102
> green = 255 * (1.0 - 0.4) + 0 * 0.4   = 153
> blue  = 255 * (1.0 - 0.4) + 0 * 0.4   = 153
>
> blended colour = (102, 153, 153, 255) 
>
> _n.b.: alpha value of 255/1.0 means fully opaque and 0/0.0 means fully transparent._

## Topics to cover

1. Movement (position, velocity, acceleration - derivatives and integrals)
1. Vectors
1. Matrices
1. Collision detection

## References

1. [http://higherorderfun.com/blog/2012/06/03/math-for-game-programmers-05-vector-cheat-sheet/](http://higherorderfun.com/blog/2012/06/03/math-for-game-programmers-05-vector-cheat-sheet/)
1. [https://www.youtube.com/watch?v=DPfxjQ6sqrc](https://www.youtube.com/watch?v=DPfxjQ6sqrc)
1. [https://www.youtube.com/playlist?list=PLW3Zl3wyJwWOpdhYedlD-yCB7WQoHf-My](https://www.youtube.com/playlist?list=PLW3Zl3wyJwWOpdhYedlD-yCB7WQoHf-My)
1. [https://thatfrenchgamedev.com/1026/game-programmers-handy-maths-formulas/](https://thatfrenchgamedev.com/1026/game-programmers-handy-maths-formulas/)
1. [https://gist.github.com/xem/99930986c5333125a13b0ea50600391f](https://gist.github.com/xem/99930986c5333125a13b0ea50600391f)

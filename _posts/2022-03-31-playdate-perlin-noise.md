---
layout: post
title:  Generating Perlin Noise on Playdate
date:   2022-03-31 08:00:00 -0700
author: vanessa
category: software
tags: playdate lua
---
<!-- cspell:ignore Perlin -->
The Playdate SDK provides native methods for generating Perlin noise. Let's take a look at how to use them.

## Better than Random
Perlin Noise is a means of generating "randomness" that looks organic. In games, this can be used for terrain or visual effects, like smoke.

> If you're more math-inclined, [Understanding Perlin Noise](http://adrianb.io/2014/08/09/perlinnoise.html) is a great article on the math behind the algorithm.

Crucially, Perlin Noise **is not** random, but deterministic. This means that when you generate a grid of noise, you only have to remember the parameters used to generate it in order to recreate it. Especially if you're generating a whole organic world, this is a huge space-saver.

Let's look at how to use it.

## Points in 3D Space
To get a single value of Perlin noise, you'll need to provide a point in three-dimensional space. This will be a number between 0 and 1, and it will always be the same value for the same coordinate.

```lua
local sample = playdate.graphics.perlin(x, y, z)
```

What X-, Y-, and Z-values should you use? That's entirely up to you - experiment and choose values that look good for your application. However, it's probably a good idea to use values that aren't integers.

Points on integer-aligned values on all three axes will always be `0.5`. If you want to have your values vary at all, choose at least one non-integer position along at least one axis. For the greatest effect, do it on all 3 axes.

### Moving Through Space
The fact that this is modeled as points in 3D space has a particularly interesting application for games. If you're using Perlin to generate spatial data - like moving through a 2D tile map - then moving through that space is as easy as moving by 1 in any axis-aligned direction.

You can see this in action with a tool like [Perlin Explorer](https://github.com/downie/playdate-perlin-explorer).

![A gif of perlin noise moving as X and Y increase](/assets/img/2022-04-01-perlin-move.gif)

Notice:
- We're using non-integer values for X and Y to start
- As we increase X by 1, it looks like the entire tile map moves left.
- Similarly, as we increase Y by 1, it looks like the entire tile map moves up.

## Advanced Generation - Repeat, Octaves & Persistence
The X, Y, and Z coordinates aren't the only arguments we can use to change the shape of the returned Perlin noise.

The `repeat` value can be used if you want to have the noise repeat as you traverse the space.

```lua
playdate.graphics.perlin(0.5, 0.5, 0, 5)
```

![A repeating noise pattern in a larger grid](/assets/img/2022-04-01-perlin-repeat-5.png)

If you want more organic values, you can achieve that by retrieving and mixing multiple Perlin values. This is so common, it's become part of the API. You can specify `octaves` to say how many values to combine, and `persistence` to specify how much to weight successive Perlin values.

```lua
local repeatValue = 0 -- Don't repeat
local octaves = 5 -- Combine 5 Perlin values
local persistence = 0.5 -- Each value is weighed as half as much as the previous
playdate.graphics.perlin(0.5, 0.5, 0, repeatValue, octaves, persistence)
```

![Less noisy perlin data shown with 5 octaves](/assets/img/2022-04-01-perlin-five-octaves.png)

## Better Performance with `perlinArray`
So far, we've talked about generating single values. As we saw in the examples above, generating a lot of values is a much more common use case. If you find yourself generating a lot of Perlin values in a loop, you can generate those values faster by generating them all at once.

There's a bit of a mental shift that comes with this. Rather than specify a single point in 3D space, you specify:

- A starting point
- How much that point moves with each sequential value (the delta)
- How many times that point should move -- also the number of Perlin values it'll return (the count)

See this in action below:

```lua
local valueCount = 100
-- Start at (0.5, 0.5, 0)
local x = 0.5
local y = 0.5
local z = 0
playdate.graphics.perlinArray(
	valueCount,
	x, 1, -- Move X by 1 with each later value
	y, 0, -- But don't move Y
	z, 0 -- and don't move Z
)
```

This method also supports all the advanced generation arguments that the single `.perlin` method does:

```lua
playdate.graphics.perlinArray(
	count,
	x, 1, -- Move X by 1 with each later value
	y, 0, -- But don't move Y
	z, 0, -- and don't move Z
	repeatValue,
	octaves,
	persistence
)
```


## Links
- [Playdate SDK for perlin noise APIs](https://sdk.play.date/1.9.3/Inside%20Playdate.html#_perlin_noise)
- [Understanding Perlin Noise](http://adrianb.io/2014/08/09/perlinnoise.html)
- [Perlin Explorer for Playdate](https://github.com/downie/playdate-perlin-explorer)

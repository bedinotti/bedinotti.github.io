---
layout: post
title:  Advanced Debugging with the Playdate Simulator
date:   2022-04-08 06:00:00 -0700
author: vanessa
category: software
tags: playdate lua
---

While the actual [Playdate](https://play.date) hardware may not be here yet, the
[Playdate SDK](https://https://sdk.play.date/) is. With it comes a pretty
thorough simulator to help you build and run Playdate games.

Not only that, but the Playdate Simulator comes with a few super-powers not
present in the hardware that really help you debug while you build your games.
Let's take a look.

## More Buttons
While a handful of keys are used to simulate button inputs in the simulator,
that leaves a lot of unused inputs on your keyboard. Fortunately, you can
capture these key presses in your code and tie them to special actions while
debugging.

There are two callbacks you can implement to accomplish this:

- [`playdate.keyPressed(key)`](https://sdk.play.date/1.9.3/#c-keyPressed) is
called whenever you press a key on the keyboard.
- [`playdate.keyReleased(key)`](https://sdk.play.date/1.9.3/#c-keyReleased) is
called whenever you release a key on the keyboard.

```lua
function playdate.keyPressed(key)
    print("Pressed " .. key .. " key")
end

function playdate.keyReleased(key)
    print("Released " .. key .. " key")
end
```

How you use these methods is a bit open-ended. You could use them to:
- Quickly reset the level you're playing
- Tweak settings in your app to see how they feel
- Log details of the game's state to the console.

## More Drawing
The Simulator also gives you another super power: the ability to draw in a color
on top of the Simulator's black-and-white display. You can use this to draw
bounding boxes, highlight areas of interest, or show private variables to the
screen.

To do that, you'll need these two methods:
- Use [`playdate.setDebugDrawColor`](https://sdk.play.date/1.9.3/#f-setDebugDrawColor)
to set the color you'd like to display the debug overlay in. Each color component (red, green, blue, and alpha) is a value from 0 to 1, inclusive, indicating how much of that color to use.
- [`playdate.debugDraw`](https://sdk.play.date/1.9.3/#c-debugDraw) is where you
can do your custom drawing. This is called immediately after `playdate.update`
on each frame.

### Color Rules
Normally, when drawing an image to the screen, you've got 3 "colors" to deal
with: black, white, and transparent. This limited color palette doesn't change
in `debugDraw`. How do we achieve color drawing?

- First of all, we only get one more color. Later calls to `setDebugDrawColor`
supersede previous ones. This is a debug view, not a full screen display.
- Second, black pixels are treated as transparent. They won't be rendered with
the debug color.

If you're just drawing with the `playdate.graphics` API, you might not notice
that second part. Even if you set the draw color to `kColorBlack` in your
`playdate.update` callback, it gets reset to `kColorWhite` in the graphics
context before `debugDraw` is called.

However, your images and text might not render. Let's solve that.

### Drawing Text & Images
Drawing images doesn't use the graphics context's current color. Instead, it
uses whatever color is present in the image. This sound perfectly reasonable:
if we've gone through the trouble to produce a black & white image, then ask
the system to draw it, we'd expect it to be drawn just as we made it.

What you might _not expect_: those same rules apply to text. Text is rendered
with a font, which is just - you guessed it - a series of images. Most fonts
define their glyphs as black-on-transparent images, which works great for
rendering to a white screen. But it makes one big problem:
**black-on-transparent images are totally transparent in `debugDraw`**.

Fortunately, there's a way to solve this without making custom white-on-transparent
debug fonts. And that's to change the fill mode.

```lua
gfx.setImageDrawMode(gfx.kDrawModeFillWhite)
```

[`playdate.graphics.setImageDrawMode`](https://sdk.play.date/1.9.3/Inside%20Playdate.html#f-graphics.setImageDrawMode)
changes how all images are drawn to the current graphics context. Choosing
`kDrawModeFillWhite` here says to treat any non-transparent pixels as white.
Now, any text or images you render in `debugDraw` will show up in your
`debugDrawColor`!

## Bring It All Together
Let's see it in action. Here, we'll
1. Draw some normal in-game text in `playdate.update`
1. Draw some geometry & text in `playdate.debugDraw`
1. Change the debug color used when you press `r`, `g`, or `b`.

```lua
function playdate.update()
    -- Show normal text drawing working
    gfx.drawText("update", 10, 60)
end

function playdate.debugDraw()
    gfx.pushContext()

    -- All drawing, including text, should be drawn in the	debug color
    gfx.setImageDrawMode(gfx.kDrawModeFillWhite)

    -- Show geometric drawing working
    gfx.drawLine(10, 20, 10, 30)
    gfx.drawPixel(20, 20)
    gfx.fillRect(30, 20, 30, 30)

    -- Show text drawing working
    gfx.drawText("debug", 10, 90)

    gfx.popContext()
end

function playdate.keyPressed(key)
    -- Toggle the color palette when "B" tapped
    if key == "r" then
        playdate.setDebugDrawColor(1.0, 0.0, 0.0, 1.0)
    elseif key == "g" then
        playdate.setDebugDrawColor(0.0, 1.0, 0.0, 1.0)
    elseif key == "b" then
        playdate.setDebugDrawColor(0.0, 0.0, 1.0, 1.0)
    end
end
```

## Final Simulator Caveats
These debug methods are super powerful, but their power is limited.
Specifically:
- You can only register key presses for printed characters. You won't get called
when someone taps "Shift".
- You can only register key presses for characters the Simulator doesn't already
use.
- You only get one debug color to draw with. No debug rainbows here.

Given that though, these are still fantastic, fun tools to use to help build your Playdate game.

---
layout: post
title: "Zelda Screen Transitions are Undefined Behaviour"
date: 2019-06-20 21:30 +1000
categories: emulation
permalink: /zelda-screen-transitions-are-undefined-behaviour
excerpt_separator: <!--more-->
---
<style>
.nes-screenshot img {
    width: 512px;
    height: 512px;
    image-rendering: crisp-edges;
}
</style>

The vertical-scrolling effect in the original "The Legend of Zelda" relies on
manipulating the NES graphics hardware in a manor likely unintended by its
designers.

<div class="nes-screenshot">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/title.png">
</div>

Not having access to any official documentation for the NES Picture Processing Unit
(PPU), this is somewhat speculative. I've been relying on the
[NesDev Wiki](https://wiki.nesdev.com/w/index.php/PPU) for a specification of how
the graphics hardware behaves. The PPU is controlled by writing to a series of memory-mapped
registers, and using these registers for their (seemingly!) intended purpose,
the following effect should not be possible:

<!--more-->

<div class="nes-screenshot">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/example.gif">
</div>

Specifically, scrolling part of the background vertically, while another part of the background
(the heads up display in the upper quarter of the screen) remains stationary, can't be done by
interacting with the PPU in the expected way.

In contrast, keeping part of the screen stationary and scrolling the rest of the screen *horizontally* is
completely well-defined.

<div class="nes-screenshot">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/horizontal-scrolling.gif">
</div>

## Types of Graphics

Graphics on the NES are split into 2 types:
 - sprites, which are tiles that can be placed at arbitrary positions on the screen and independently move around
 - the background, which is a grid of tiles which can be scrolled smoothly as a single image

To highlight the difference, here's a scene made up of sprites and background:
<div class="nes-screenshot">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/sprites-and-background.gif">
</div>

Here's the same scene with only the sprites visible:
<div class="nes-screenshot">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/only-sprites.gif">
</div>

And here's the scene with only the background visible:
<div class="nes-screenshot">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/only-background.gif">
</div>

## Scrolling

The NES Picture Processor supports scrolling background graphics.
Video memory contains a grid of tiles 2x2 times the size of the screen.
The screen displays a screen-sized window into this grid, and the position of the window can
be precisely controlled. Gradually moving this window within the grid produces a smooth scrolling effect.

The video output from the NES is 256x240 pixels. The in-memory tile grid represents a 512x480
pixel area, and is broken up into 4 screen-sized quadrants called "name tables". Games can configure the
Picture Processing Unit (PPU), to specify the position of the visible screen-sized window
by selecting a pixel coordinate within the in-memory tile grid.

Choosing the coordinate (0, 0) will display the entire top-left name table:

<img src="/images/zelda-screen-transitions-are-undefined-behaviour/0,0.png" style="width:50%">

Scrolling to (125, 181) shows a bit of each name table:

<img src="/images/zelda-screen-transitions-are-undefined-behaviour/125,181.png" style="width:50%">

The visible window wraps around to the far side of the in-memory tile grid.
Scrolling to (342, 290) will place the top-left corner of the visible screen inside the
bottom-right name table, and parts of each name table will be visible due to wrapping:

<img src="/images/zelda-screen-transitions-are-undefined-behaviour/342,290.png" style="width:50%">

### The Catch

Each name table is 1kb in size, but the NES only dedicates 2kb of its video memory to name tables,
so there are only 2 name tables worth of memory.

How does it hold 4 screens worth of tile grid?

Video memory is connected to the PPU in such a way that when the PPU renders a tile from one of the 4
apparent name tables, one of the 2 real name tables is selected, and read from instead. This effectively
means that the 4 apparent name tables are made up of 2 identical pairs of name tables.

Why not just have 2 name tables then?

Fortunately, the precise mapping between apparent name table and real name table can be configured
at runtime. If a game wants to scroll horizontally, it will configure graphics hardware such that
the top-left and top-right name tables are different, so it can scroll between them without any visible duplication.
In this configuration, the top-left and bottom-left name tables will refer to the same
real name table, and likewise the top-right and bottom-right. This configuration is named "Vertical Mirroring".

<img src="/images/zelda-screen-transitions-are-undefined-behaviour/vertical-mirroring.png" style="width:50%">

The other possible configuration is "Horizontal Mirroring", which games use when they want to scroll vertically.

<img src="/images/zelda-screen-transitions-are-undefined-behaviour/horizontal-mirroring.png" style="width:50%">

Games usually don't scroll diagonally, as it produces artifacts around the edge of the screen due to name table
mirroring.

### Cartridges

Each game's cartridge contains hardware which allows name table mirroring to be configured.

<img src="/images/zelda-screen-transitions-are-undefined-behaviour/cart.jpg" style="width:50%">

Some games don't ever need to change mirroring, so their cartridges are hardwired to either horizontal
or vertical mirroring. Other games need to dynamically switch between the two modes, so their cartridge
can be configured by software to mirror horizontally or vertically. The Legend of Zelda falls into this
category. Finally, some really fancy games come with
extra video memory in the cartridge, which means they don't need to mirror at all, and can scroll
horizontally and vertically at the same time without any visible duplication.

### A real example

On the left is an example of vertical scrolling as it would appear on the screen.
On the right is a recording of the name tables, with horizontal mirroring, and the currently-visible
window highlighted.

<div class="nes-screenshot">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/scroll-demo.gif" style="width:50%;height:50%;float:left">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/scroll-demo-name-table.gif" style="width:50%;height:50%">
</div>

## Screen Splitting

Each frame of video produced by the NES is drawn from top to bottom, one row of pixels at a time.
Within each row, pixels are drawn one at a time, left to right.
Mid way through drawing a frame, the game can reconfigure the PPU, which effects how the yet-to-be
drawn pixels are displayed. One common mid-frame change is to update the horizontal scroll position.

<div class="nes-screenshot">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/horizontal-scroll-demo.gif" style="width:50%;height:50%;float:left">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/horizontal-scroll-demo-name-table.gif" style="width:50%;height:50%">
</div>

When scrolling horizontally between rooms, The Legend of Zelda always starts with scroll set
to (0, 0), and renders the heads up display at the top of the screen.
After the final row of pixels of the heads up display has been drawn to the screen,
the horizontal scroll is changed by a value which increases slightly each frame,
causing the camera to pan smoothly.

The name table view shows how the game is changing from horizontal mirroring to vertical mirroring before
it starts scrolling, then back to horizontal mirroring once the transition is complete. Also, while the scroll
is in progress, the top-left (and bottom-left) name table is updated to contain a copy of the room being entered.
Once the scroll is finished, the game stops splitting the screen, and renders entirely from the top-left name table
again.

### Measuring Draw Progress

In order to split the screen at the correct position, the game needs a way of finding out how much of the
current frame has been drawn.
Pixel rows are rendered at a known rate, so it's possible to tell which row of pixels is currently being
drawn by counting the number of CPU cycles that have passed since the start of the frame.

There is another, more accurate technique, called "Sprite Zero Hit".

The NES can draw 64 sprites at a time. The first sprite in video memory is referred to as "Sprite Zero".
Each frame, the first time an opaque pixel of sprite zero overlaps with an opaque pixel of the background,
an event called "Sprite Zero Hit" occurs. This has the effect of setting a bit in one of the memory-mapped
PPU registers, which can be checked by the CPU.

To use Sprite Zero Hit to split the screen, games place sprite zero at a vertical position near the boundary
of the split, and during rendering, repeatedly check whether a Sprite Zero Hit has occurred.
When Sprite Zero Hit occurs, the game changes the horizontal scroll to effect the split.

This shows a horizontal room transition with and without the background.

<div class="nes-screenshot">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/horizontal-scroll-demo.gif" style="width:50%;height:50%;float:left">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/horizontal-scroll-demo-sprites.gif" style="width:50%;height:50%">
</div>

The brown circle
which appears at the start of the transition, and vanishes at the end, is sprite zero.
Looking closer at the HUD with and without the background:

<img src="/images/zelda-screen-transitions-are-undefined-behaviour/hud.png" style="width:512px;image-rendering:crisp-edges">
<img src="/images/zelda-screen-transitions-are-undefined-behaviour/hud-sprites.png" style="width:512px;image-rendering:crisp-edges">

Sprite zero is a discoloured bomb sprite, lined up exactly with the regular bomb sprite in the game's HUD.
Sprite zero is configured to appear behind the background, but since the black pixels in the HUD are considered
transparent, the sprite zero bomb would be visible if it wasn't strategically positioned behind the HUD bomb.

Note that the sprite zero hit occurs several pixel rows before the bottom row of the HUD.
It occurs at the top pixel of the fuse of the bomb, which is 16 pixels from the bottom of the HUD.
When sprite zero hit happens, the game starts counting CPU cycles, and sets the horizontal scroll
after a specific number of cycles have passed.

It's not clear to me whether screen splitting was the original purpose of sprite zero hit.
It seems like there should be a simpler way for the hardware to report that it's reached
a certain part of the screen than requiring the game to draw a sprite. A possibility is
it was intended to be used for collision detection, but it would only be able to detect
collisions with a single sprite and the background.

## Anatomy of a Frame

The majority of the time, the NES PPU is drawing pixels to the screen.
There is a brief period of time in between frames during which no drawing
is taking place. This is known as the "Vertical Blank" or "Vblank".

![](/images/zelda-screen-transitions-are-undefined-behaviour/frame-anatomy.png)

The precise duration of vblank differs between regional TV standards.
The NTSC NES sold in North America and the PAL NES sold in Europe have different hardware
to handle the different requirements for frame timing.

## The PPU Interface

Programs running on the NES interact with graphics hardware via memory-mapped registers.
These are special memory addresses which can be read and written like normal memory,
except instead of loading and storing data, properties of the graphics hardware
can be configured and queried.

This table describes the relevant registers for this story.
Each register is 8 bits wide.
Some registers can only be safely accessed during vblank.

<table>
<tr><th>Name</th><th>Address</th><th>Description</th></tr>
<tr><td>PPUCTRL</td><td>0x2000</td>
<td>
<p>
When this register is written, each bit of the written value
specifies some property of how the PPU should behave.
In particular, the low 2 bits are treated as a number from 0 to 3
and select one of the 4 name tables which will contain the top-left
corner of the visible screen-sized window.
</p><p>
Writing this register outside vblank can cause graphical anomalies.
</p>
</td></tr>
<tr><td>PPUSTATUS</td><td>0x2002</td>
<td>
Contains various status flags, including whether the PPU is currently
in vblank, and whether sprite zero hit has occurred this frame.
</td></tr>
<tr><td>PPUSCROLL</td><td>0x2005</td>
<td>
<p>
Sets the position of the visible screen-sized window within the name table
selected in PPUCTRL.
</p><p>
During vblank, writing to this register alternates between setting
the X and Y position of the window.
</p><p>
Outside of vblank, setting the X position works as expected,
but the change will only take effect on the following pixel row.
Setting the Y position outside of vblank has no effect on the position of the visible window.
This is the reason why it <em>shouldn't</em> be possible to keep part of the screen stationary
while the rest scrolls vertically.
</p>
</td></tr>
<tr><td>PPUADDR</td><td>0x2006</td>
<td>
</td></tr>
<tr><td>PPUDATA</td><td>0x2007</td>
<td>
</td></tr>
</table>
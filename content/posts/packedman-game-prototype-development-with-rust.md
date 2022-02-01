---
title: "Packedman game prototype development with Rust"
date: 2022-01-27T15:12:00Z
draft: false
tags: ["rust", "games"]
categories: ["packedman"]
---

I'm developing a [Pac-Man][1] inspired game in Rust called Packedman (think Pac-Man with guns).
I'm not using any game engines or frameworks like Bevy or Amythest.
Instead, I'm developing the game from scratch so that I can learn about how game engines work.
In this and following articles I'll be talking about how I approached creating this game.
There are many other ways - mine is not the best way, hopefully it's not the worst either!

## Current progress

I'm working towards completing a prototype that is somewhat of a clone of the original Pac-Man.
I've currently implemented a game loop with fixed framerate, basic graphics, movement and maze loading.
The screenshot below shows an example of what I have implemented so far.

![Packedman development screenshot](/packedman/01/img/development-screenshot.png)

To get the prototype to this point I've had to implement a number of features, including graphics, input, fixed framerate and maze loading.
I'm going to give an overview of each of these key features below.
[View the game at the related commit on GitHub][4].

### Graphics

I wanted to be able to see something on my screen quite quickly.
Some advice I've read suggests this is a bad approach but my general software development experience has been that workign towards concrete goals is best.
I didn't want to spend ages grappling with OpenGL, Vulkan or [wgpu][3], so I chose to use a library called [minifb][2].
This allows me to write pixels to a framebuffer and present that to the screen without GPU acceleration.

The framebuffer data is a `Vec<u32>` with each element representing a single pixel.
The vec has capacity for the required number of pixels (calculated as _screen width * screen height_).
I'm using a `FrameBuffer` struct to encapsulate the relevant functionality:

```rust
pub(crate) struct FrameBuffer {
    pub(crate) data: Vec<u32>,
    pub(crate) width: usize,
    pub(crate) height: usize,
}

impl FrameBuffer {
    fn new(width: usize, height: usize) -> Self {
        Self {
            data: vec![0x0; width * height],
            width,
            height,
        }
    }
}
```

The `FrameBuffer` struct's fields are public for other modules within the crate for now, since it was the simplest solution during an earlier refactoring.

Computing the index for a given pixel coordinate is relatively simple given the _x_ and _y_ coordinates and the _width_ of the framebuffer.
The index is calculated as the number of pixels before the required row offset by the number of pixels within the required row.
Or `framebuffer.width * y + x`.
I make use of this in my `Gfx` struct when drawing a rectangle into the framebuffer:

```rust
pub fn draw_rect(&mut self, p1: Vec2, p2: Vec2, color: Color) {
    let x1 = (p1.x * self.pixels_per_meter as f32).round() as i32;
    let x2 = (p2.x * self.pixels_per_meter as f32).round() as i32;
    let y1 = self.height as i32 - (p2.y * self.pixels_per_meter as f32).round() as i32;
    let y2 = self.height as i32 - (p1.y * self.pixels_per_meter as f32).round() as i32;

    let x1 = clamp(0, x1, self.width as i32);
    let x2 = clamp(0, x2, self.width as i32);
    let y1 = clamp(0, y1, self.height as i32);
    let y2 = clamp(0, y2, self.height as i32);

    for y in y1..y2 {
        for x in x1..x2 {
            let pixel = y * self.width as i32 + x;
            self.framebuffer.data[pixel as usize] = color.as_argb();
        }
    }
}
```

I wanted my screen coordinates origin to be in the bottom left hand corner.
This means that the pixel representing _(0, 0)_ is actually at index _(0, height).
It also means that `y1` and `y2` need to be computed by subtracting the _y_ value of `p2` and `p1` respectively from the height.
Doing so reverses the cardinality of the y-axis, enabling me to represent coordinates increaseing upwards from the bottom left corner.

I've also decoupled the size of my game world from the size of a pixel.
In future, it will be simple to scale the game based on the size of the window that is being displayed into.

Within the framebuffer, each pixel is represented as `u32` and layed out as `alpha | red | green | blue`.
Each of the 4 components is represented as `u8`.
My `Color` struct handles the storage and manipulation of these bytes, meaning the necessary bitwise operations are nicely encapsulated.
Each 8-bit component is left-shifted into it's correct position and or'd with all other components:

```rust
fn as_argb(self) -> u32 {
    let (r, g, b, a) = (self.r as u32, self.g as u32, self.b as u32, self.a as u32);
    (a << 24) | (r << 16) | (g << 8) | b
}
```

Updating the display is handled by minifb:

```rust
let framebuffer = self.gfx.get_buffer();
window.update_with_buffer(&framebuffer.data, framebuffer.width, framebuffer.height)?;
```

I'd like to use GPU acceleration as a future enhancement; most likely I'll use the [wgpu][3] crate.
I suspect I'll choose a half-way house approach and continue to write my own framebuffer and load that as a tecture for the GPU to display.

### User input

For handling user input, I wanted to be able to pass the current input state to the game when it is updated each frame.
This allows me to provide an `Input` abstraction over minifb's input mechanism.
Whilst not essential, it provides all the usual benefits that encapsulation and code ownership bring.

```rust
#[derive(PartialEq, Eq, Hash)]
pub enum Key {
    Up,
    Down,
    Left,
    Right,
    Space,
}

pub struct Input {
    pub(crate) keys: HashMap<Key, bool>,
}

impl Input {
    pub(crate) fn new() -> Self {
        Self {
            keys: HashMap::new(),
        }
    }

    pub fn is_key_down(&self, key: Key) -> bool {
        match self.keys.get(&key) {
            Some(&is_pressed) => is_pressed,
            None => false,
        }
    }
}
```

Since Packedman has a simple set of controls, I chose to limit the keys I will handle to those that can move the player and fire the weapon.
During the game loop, before the game is updated, input is processed and the state updated to record whether a key is down.
There's not really anything more complicated to it that that.

### Fixed framerate

I've chosen to implement Packedman as 60 frames per second (FPS), which is quite a common framerate.
I could have chosen 30 FPS, which is also common, but decided to start with the higher and reduce if necessary.
At the moment, getting 60 FPS means making the game loop pause between frames so that updates don't happen too frequently.
I achieve this with the following game loop:

```
start clock

while window is open:
    get input

    if elapsed time is less than target frame duration:
        sleep for remaining frame duration minus some sleep tolerance
        while elapsed time is less than target frame duration:
            eat CPU cycles for remaining duration

    update clock

    update game
    render
    update window
```

I chose to put my elapsed time checks and sleep window before the game is updated and rendered.
This is because I want my delta time between update calls to always be 16.7ms.
I could also have put the logic for this after rendering, making the framebuffer presentation the start of a frame.
I've also used a hardcoded sleep tolerance, which I've found is needed to ensure the sleep duration does not exceed that which was requested.
I don't know how effective this approach will be in the longer term but it serves it's purpose for now:

```rust
let target_frame_micros = (ONE_SECOND_AS_MICROS as f32 / self.fps as f32) as u64;
let target_frame_duration = Duration::from_micros(target_frame_micros);
let mut clock = Clock::new();

while window.is_open() {
    let input = process_input(&mut window);

    if clock.split() < target_frame_duration {
        let sleep = target_frame_duration - clock.split();
        let sleep_tolerance = Duration::from_millis(2);
        if sleep >= sleep_tolerance {
            std::thread::sleep(sleep - sleep_tolerance);
        }

        while clock.split() < target_frame_duration {
            // Eat CPU cycles for remaining duration.
        }
    }

    clock.update();

    game.on_update(&input, clock.delta());
    game.on_render(&mut self.gfx);

    let framebuffer = self.gfx.get_buffer();
    window.update_with_buffer(&framebuffer.data, framebuffer.width, framebuffer.height)?;
}
```

The clock I use is relatively simple.

```rust
pub struct Clock {
    last_update: Instant,
    delta: Duration,
}

impl Clock {
    pub fn new() -> Self {
        Default::default()
    }

    pub fn update(&mut self) {
        let instant = Instant::now();
        self.delta = instant - self.last_update;
        self.last_update = instant;
    }

    pub fn split(&self) -> Duration {
        Instant::now() - self.last_update
    }

    pub fn delta(&self) -> Duration {
        self.delta
    }
}
```

It needs to be updated each frame with a call to `Clock::update()`, although I may rename this to `tick` in the future.
The currently elapsed time since the last call to `update` is available from the `split` method.
The time between the last two calls to `update` is available from the `delta` method.

### Maze loading

I'd like to be able to generate random mazes for Packedman to navigate.
However, I'm starting with a single maze that can be loaded from a string:

```
┌──────────────────────────┐
│............║║............│
│.╔══╗.╔═══╗.║║.╔═══╗.╔══╗.│
│.║  ║.║   ║.║║.║   ║.║  ║.│
│.╚══╝.╚═══╝.╚╝.╚═══╝.╚══╝.│
│..........................│
│.╔══╗.╔╗.╔══════╗.╔╗.╔══╗.│
│.╚══╝.║║.╚══╗╔══╝.║║.╚══╝.│
│......║║....║║....║║......│
└────┐.║╚══╗ ║║ ╔══╝║.┌────┘
     │.║╔══╝ ╚╝ ╚══╗║.│     
     │.║║          ║║.│     
     │.║║ ╔══DD══╗ ║║.│     
─────┘.╚╝ ║HHHHHH║ ╚╝.└─────
.......   ║HHHHHH║   .......
─────┐.╔╗ ║HHHHHH║ ╔╗.┌─────
     │.║║ ╚══════╝ ║║.│     
     │.║║          ║║.│     
     │.║║ ╔══════╗ ║║.│     
┌────┘.╚╝ ╚══╗╔══╝ ╚╝.└────┐
│............║║............│
│.╔══╗.╔═══╗.║║.╔═══╗.╔══╗.│
│.╚═╗║.╚═══╝.╚╝.╚═══╝.║╔═╝.│
│...║║................║║...│
└─┐.║║.╔╗.╔══════╗.╔╗.║║.┌─┘
┌─┘.╚╝.║║.╚══╗╔══╝.║║.╚╝.└─┐
│......║║....║║....║║......│
│.╔════╝╚══╗.║║.╔══╝╚════╗.│
│.╚════════╝.╚╝.╚════════╝.│
│..........................│
└──────────────────────────┘
```

The string is processed character by character to build a programmatic representation.
I've used a pretty beefy `match` expression to construct a maze tile for the appropriate topology.
For example:

```rust
let mut y = height * tile_size.y;
for row in map_rows {
    let mut x = 0.0;
    let mut tiles = row.chars();
    while let Some(tile) = tiles.next() {
        match tile {
            '│' => walls.push(Wall {
                pos: Vec2::new(x, y),
                size: Vec2::new(tile_size.x, tile_size.y),
            }),
            ...
            _ => (),
        }
        x += tile_size.x;
    }
    y -= tile_size.y;
}
```

There are definitely some improvements I can make to the maze loading code as I go forward.
However, it serves a purpose for now and allows me to progress on to other aspects of the initial prototype.

## What's next?

My next steps are going to be to add collision detection so that I can keep Packedman within the maze passageways.
Then I want to add a basic debug UI to show FPS and collision boxes.
To do that I'll need to add the ability to display text, but I don't want to deal with font loading right now.
With those items of functionality in place, I hope to add the ability for Packedman to fire projectiles and collect treasure.


[1]: https://en.wikipedia.org/wiki/Pac-Man "Pac-Man"
[2]: https://github.com/emoon/rust_minifb "rust_minifb"
[3]: https://github.com/gfx-rs/wgpu "wgpu"
[4]: https://github.com/junglie85/packedman/tree/4ab3ce475abb93e77c0b85db1b117b3f19f8ca98 "Packedman at current git commit"

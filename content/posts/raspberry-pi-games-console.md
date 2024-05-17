---
title: "Raspberry Pi Games Console Project Intro"
date: 2024-05-17T09:04:20+01:00
draft: false
tags: ["raspberry pi", "games", "console"]
projects: ["pi console"]
summary: "Raspberry Pi games console project introduction"
---

I'm finally taking the plung to embark on a reasonably lengthy project to create a Raspberry Pi games console.
Now, I know what you're thinking: [that's already been done][1]. Yes, that's true, but I want to embark on
a slightly different challenge, which I'll outline below.

## The goal

The aim of this project is to develop, from scratch, a simple games console using a Raspberry Pi. This is more
than just creating a game and running it on a Pi. Instead, I want to develop a minimal development environment
to support creating games on the Pi. The following is a not-complete list of the things that I think I want
to include in the project:

- Cross-compile from Windows to whatever OS is running on the Pi.
- Custom and lightweight OS on the Pi, either Linux or FreeBSD.
- No windowing system on the Pi (i.e. no X11 or Wayland), just using the DRM (Direct Rendering Manager).
- Use Vulkan as the provided graphics API.
- It should be capable of playing sound and receiving gamepad input for 2, maybe 4 pads.
- Some level of integration with an editor, probably VSCode.

There are some things I am undecided on yet, such as:

- Whether to use OpenGL instead of Vulkan to make things a bit easier.
- How games will be distributed:
  - Embedded on a Pi.
  - Via an "online store".
  - Via a "cartridge", i.e. SD card or USB.
- How to handle saves, temporary file system, etc.
- Support for other development OS's, e.g. cross-compiling from Linux.
- If I should have "development" and "consumer" versions of the console.

## The approach

As with any endeavour, as I learn more about the problem domain, I'll probably change my requirements, add some
features, remove others, etc. This initial set was chosen because it's more taxing than just using the Pi to
compile code and is, I think, similar to how development on current and previous generation consoles happens.
Plus, I think it's going to be fun and educational, and I'm going to do my best to document my progress as
I go.

There are quite a few things I need to figure out before I can start pulling all of these threads together,
so I'm going to do some smaller projects as I research what is needed:

- Exploring DRM and OpenGL/Vulkan on Linux and FreeBSD in a VM.
- Building a custom Linux distro or FreeBSD for the target OS.
- Creating a cross-toolchain from Windows to Linux.
- Creating a cross-toolchain from Windows to target OS.
- Writing a simple VSCode extension.

I'm not necessarily going to do these in this order and I may well jump into the main project as I'm working through
these.

## Next steps

First things first, I need to make sure some of these ideas are actually valid. Specifically, I want to make sure
that I can get OpenGL and probably Vulkan working in a FreeBSD installation on a VM using just the DRM. If I can't
do that then the project needs to be rescoped.

[1]: https://retropie.org.uk/ "RetroPie"

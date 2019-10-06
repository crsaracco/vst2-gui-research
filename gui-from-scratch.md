# Notes on creating a new GUI crate from the ground up

If no already-existing GUI crates work well out-of-the-box for our purposes, what about making our own? We can get very fine-grained control over the [two main problems in VST GUI creation](vst2-gui-background.md), and we can tailor it to [our exact requirements](vst2-gui-requirements.md); no more, no less.

Unfortunately, "our requirements" actually encompass a *lot* of work:

 - Window creation / attaching to a parent window
 - Cross-platform GUI context creation (OpenGL, Vulkan/Metal, etc)
 - Event handling
 - Wrapping all of that up in a platform-agnostic crate for others to use

And that's before we try to create *another* crate for drawing on the GUI context -- circles, rectangles, menu systems, etc.

**TL;DR: Creating our own GUI crate from scratch is a lot of work**, and the tradeoffs between it and trying to use an existing GUI crate should definitely be considered. Even if we get the basics down, there's still a lot of hidden issues like "how does DPI scaling work" and many others, that an existing GUI crate would already have solved out-of-the-box. Even if all of the other crates might not work for our use-case, we're still duplicating *a lot* of effort and fragmenting the already-thin Rust GUI ecosystem further. Definitely consider if it's worth it to contribute to some other upstream crate to try to fulfill our use-case, or at least fork one and mold it to suit our needs.

## The basic idea

The basic idea is to [define a set of requirements](vst2-gui-requirements.md), get each requirement working in a platform-specific module, and then wrap everything together in a cross-platform interface module/trait/whatever so that people can use the crate without worrying about the platform-specific implementation details.

 - Working with the platform-specific APIs for managing windows is tedious, but doable.
 - Setting up a cross-platform GUI context (OpenGL, etc.) is possible, with some googling.
 - The harder part is figuring out the intricacies of each platform's event system, and trying to design an umbrella API that can handle each one in a platform-agnostic way.

Once we have those things sorted, we should be able to create a window that connects to the window that the DAW gives us, and handle events like mouse clicks and keyboard presses in a way that our VST plugin can understand. Additionally, the user should be able to target one GUI paradigm (OpenGL, etc.) and have it work on all three platforms.

As mentioned before, it would be nice to also have another crate on top of that for GUI primitives (creating circles/rectangles, etc) in the chosen system, so that it's easier to draw potentiometers, levels, menus, and whatever else people would like to create.

## Past attempts

#### [rtb-rs](https://github.com/rust-dsp/rtb-rs)

`rtb-rs` is (was?) an attempt to create a library built around the "basic idea" above. The project got to the point where you could spawn a window that connects to the DAW on all three platforms through a platform-agnostic API; OpenGL and event support is a little more sparse.

The project suffered from two issues:

 1. A lot of the "platform-agnostic" API was built up before anyone knew too much about the platforms themselves, which led to a lot of churn as developers discovered that parts of the API wouldn't work on all three platforms and resulted in some platform-specific code having to undergo major reworks or being thrown out entirely.
 2. There wasn't a concentrated effort in the development of each of the individual platforms (and Windows didn't have a dedicated developer at all), so when churn did happen, it took *quite a long time* to resolve.

These are mostly project management issues; no technical/software issues were uncovered that stopped us from continuing this project to completion. I think if we had more devs actively working on the project (in particular, at least one active dev for each platform), this project could actually work with enough elbow grease.

#### [vst2-window](https://github.com/crsaracco/vst2-window)

Taking note of problem #1 from `rtb-rs`, I decided to try my hand at creating my own crate from the ground up. I didn't define a platform-agnostic API to start with; instead, I opted to let the platform-agnostic API grow organically as I observed what each platform needed to do to fill the requirements of a VST GUI.

In my research phase, I decided to [create some prototype VSTs that each only support one platform](https://github.com/crsaracco/vst2-gui-prototypes).

 - I got a [Linux (X11)-specific VST](https://github.com/crsaracco/vst2-gui-prototypes/tree/master/linux-opengl-vst) working with OpenGL and simple events.
 - I got a [Windows-specific VST](https://github.com/crsaracco/vst2-gui-prototypes/tree/master/windows-opengl-vst) working with OpenGL and simple events.
 - I even purchased a Mac Mini and [did some prototype OpenGL work](https://github.com/crsaracco/vst2-gui-prototypes/tree/master/osx-opengl), without the VST part.

Once I did all three of those, I felt reasonably confident that I could get everything working and started to create [`vst2-window`](https://github.com/crsaracco/vst2-window). My idea was to define one chunk of the platform-agnostic interface, then implement it for all three platforms, then define one more platform-agnostic chunk, implement, and iterate like that.

Around here, I started to burn out -- I don't really like GUI programming in the first place, and I felt like I'd rather be coding other things, so I shelved the project for the time being.

This project ultimately suffered from one issue:

 1. The project had one programmer, who doesn't really like GUI programming in the first place :). Since I'm not really familiar with GUI programming in general, I had to go research how to do the same thing three different times, once for each platform. It's mind-numbing.

With more (active) programmers, each focused on one particular platform, I think this method is the best way to do it and is very likely to succeed.

## Ideas for future attempts

Given the successes and failures of the previous two projects, here's how I would go about trying to implement our own cross-platform GUI crate from scratch:

 1. Conjure up some developers who are willing to dedicate the good part of the next few months' worth of weekends to programming such a GUI crate, while being active on Discord to help keep the project going smoothly.
    - Ideally, each dev working on their own favorite platform, so that dumping some time into research doesn't seem like such a waste of time to them.
    - At least one dev per platform, ideally two or more.
 2. Brainstorm what an ideal end-goal would be like.
    - What's in scope, and what's out-of-scope?
    - Which cross-platform GUI context to use? OpenGL? Vulkan + Metal?
    - Which events to handle?
    - What should the graphics and events API look like to a VST programmer using this crate?
 3. Go out and develop platform-specific VSTs (like the [prototypes](https://github.com/crsaracco/vst2-gui-prototypes) repo I made) to get a feel for the problem space in that particular platform. If there's more than one developer for a given platform, they can either work together to do this, or split up and each make their own. Either way, try to implement a majority of the end-goal requirements for each platform before reconvening and working on the next step.
 4. Make a new shared repo that defines a small chunk of the platform-agnostic API, and have each platform-specific team implement it completely before moving on. Then, define another small chunk of the platform-agnostic API, and implement that. Iterate until complete.
    - You'll probably want to start with window creation / connecting to the DAW window as the first major step. Then, maybe getting an OpenGL/Vulkan+Metal context and proving that you can draw to it with platform-agnostic APIs? Then event handling?
    - Ideally, you constantly figure out which part has the biggest risk of not fitting into the platform-agnostic API and therefore messing everything up, and work on that next. The big goal here is to minimize the amount of churn that might happen if/when something does mess everything up. Try to make as few assumptions as possible.
    - Since each team already made their own platform-specific VST, this phase should *hopefully* be relatively easy; implementing each iteration should be only slightly harder than copy-pasting your previous work into this new repo.

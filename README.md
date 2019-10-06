# VST2 GUI Research

This repo is intended to be a brain-dump for anyone interested in working on GUIs for [vst-rs](https://github.com/RustAudio/vst-rs).

There have been a few efforts in the past for trying to get this working, but what always ends up happening is that someone dumps a considerable amount of time working on it, learns a bunch of new information, then drops the project for whatever reason, slowly losing the information they've gained. My goal for this repo is to instead retain that information, so that we can eventually converge on a solution.

If you're interested in doing some GUI work in this space, read through this repo to get a feeling for what the requirements are, what has worked, and what hasn't.

If you've been doing some GUI work and you have *any* bit of information that isn't in this repo, please file a PR or let us know what you've found on [the RustAudio discord server](https://discord.gg/QPdhk2u) (channel #vst2-gui)!

## Goals

Ideally, we would like to have a GUI system that works on all three of the "major" platforms that might use `vst-rs`: Windows, Mac OS X, and Linux (X11). A programmer who uses the solution we come up with should be able to just *use it* to create a VST GUI: they shouldn't have to worry about coding platform-specific stuff, and any given feature we have should work equally well on all three platforms.

Information that only applies to one of those specific platforms is still appreciated, because it still helps us scope out the overall requirements. Other platforms are considered "out of scope" for this project, but I will happily collect that information as well.

I will list out "harder" requirements on a separate page.

## Approaches

There are two general approaches to getting GUIs working for Rust VSTs:

 1. Try to use a cross-platform crate that already exists, figure out it isn't *quite* what you need, and decide to hack around with it trying to get something working.
 2. Build something new from the ground up that fits our exact requirements, and realize there's quite a bit of work ahead of you.

Each of these approaches will have their own page to document findings.

## Information

 - [Background on how GUIs work in VST2](vst2-gui-background.md)
 - Requirements specification for a Rust VST2 GUI crate **(TODO)**
 - Notes on using an already-existing Rust GUI crate **(TODO)**
 - Notes on creating a new GUI create from the ground up **(TODO)**
 - Resources that might be worth looking into **(TODO)**
    - example: `rutabaga`, my GUI prototype repo, `rtb-rs`, etc.

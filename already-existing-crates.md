# Notes on using already-existing crates

Document anything you've found by trying to use crates from the greater Rust ecosystem here.

What works already? What's preventing us from using it with `vst-rs` (parent window handle / child window creation, event loop, etc.)? Are there any GitHub issues tracking those things? How hard would it be to hack it to get it to work for our needs? How hard would it be for us to maintain that fork?

Also, put a date next to your findings so we can judge how fresh the information is.

## Conrod

*(TODO)*

## Piston

*(TODO)*

## Druid

*(TODO)*

## gfx-rs

*(TODO)*

## Iced

Repo: https://github.com/hecrj/iced

[12/12/2019]

There is a [very early vst-rs gui example](https://github.com/hatoo/vst-rs-example-iced) with an associated [iced issue](https://github.com/hecrj/iced/issues/118) tracking the ability to pass state into the GUI application. 

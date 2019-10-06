# Background on how GUIs work in VST2

*(**TODO:** might break this up so that each platform gets its own section, depending on how this page evolves.)*

*(**TODO:** at the time of writing, I haven't worked on GUIs in a while, so certain things are a bit generalized and hand-wavey. More precise information is always good!)*

## Host window creation

The DAW handles the lifecycle of the actual GUI window: creating it, opening it, closing it, etc.

When a VST plugin wants to have a GUI, the DAW creates this window and passes a handle to it to the plugin, so the plugin has access to it.

The type of window that is created is **platform-dependent**, which means that the API you use to interact with the window is also platform-dependent.

Your job as a VST2 GUI developer is always the same, though: "connect" your GUI to this parent window, using the handle that the DAW gave you, so that the DAW can manage the window (open, close, get events, etc). The "connecting" part is usually done by passing the handle as an argument to your platform's "native" window creation API (Windows' `CreateWindowEx`, X11's `xcb::create_window`, etc).

#### Specifics for Rust and `vst-rs`

When using `vst-rs`, the handle will always be a `*mut std::os::raw::c_void`, which is given to you through [`Editor::open`](https://github.com/RustAudio/vst-rs/blob/c1e29953a80946c47987180bec8b3c26c494941a/src/editor.rs#L32):

```rust
fn open(&mut self, parent: *mut c_void) -> bool;
```

 - In Windows, that parent pointer is really a `winapi::HWND__`. You can convert the parent pointer by doing `parent as *mut winapi::HWND__`, and connecting to the window through [`CreateWindowExW`](https://github.com/crsaracco/vst2-gui-prototypes/blob/master/windows-opengl-vst/src/editor/window/win32_window.rs#L23-L36).
 - In Linux, that parent pointer is actually just an ID number to the parent window. You can convert the parent pointer by doing `parent as u32` and then connecting to the window through [`xcb::create_window`](https://github.com/crsaracco/vst2-gui-prototypes/blob/master/linux-opengl-vst/src/editor/window/mod.rs#L112-L114).
 - **TODO:** Mac OS X

## Events and threads

*(**TODO:** Super hazy information ahead. Please contribute if you know more!)*

VST2 expects the DAW to have complete control over the application, including its threads. Generally, the DAW will have one or more threads dedicated for each running plugin, and will send calls over FFI when an event is ready to be dealt with (window closing, audio buffer ready to be processed, etc). VST2 expects that function call to give control back to the DAW, so it shouldn't run forever.

This generally means that you should use the platform's native event handling system instead of spawning your own thread, which is true on Windows and Mac OS X. Linux, however, doesn't have a "native event handling system" to use, so you *have* to spawn your own thread. The joys of platform-dependent development!

The problem, though, is that this is **incompatible** with *most (all?)* of the GUI crates in the Rust ecosystem. They almost always expect you to be building a standalone GUI application, so they assume that you want your own event loop and event processing thread, so they make it for you. Check out the ["Notes on using an already-existing Rust GUI crate"](already-existing-crates.md) page for more info.

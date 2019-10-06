# Rutabaga notes

Information about Rutabaga-specific implementation details, from @wrl himself. Light editing for formatting reasons.

Editor's note: I'd like to eventually take this information and generate some requirements that each specific platform has to implement, but for now I'm just leaving it in "brain dump" format. Also, "implementation details" probably doesn't describe this page too accurately, since a lot of this probably applies to *any* VST-specific GUI crate you'll make. Regardless, here's the deets:

## Brain dump

 - On linux, since the connection to the window server (under x11) is just a file descriptor, you're free to spin up a new thread and run your own event loop from there, and this is what I do in rtb. Sort of. rtb assumes that it owns the event loop on linux, so in the plugin I spin up a new thread and run rtb from it.
 - On linux, you must spin up a thread to own the event loop, for there is no other event loop to which you can subscribe. On windows and mac, you *absolutely cannot* create a new thread. Everything will break in very sludgy unfortunate ways.
    - The reason why it breaks things is generally deep inside the platform library and "I'm not really sure".
    - I know on windows that the event dispatching that goes from parent window to child window can't cross threads, or the events just... don't show up.
    - On mac, NSView is marked "main thread only". So I assume there are assumptions in cocoa related to that -- such as some data structures being insufficiently (or not at all) protected by mutexes or whatever.
 - Nobody has decided how plugin UIs should work natively on wayland yet, so that's all WIP and rtb has no support for it presently.
 - On windows and macos, the connection to the window system is a platform-specific abstraction and is not exposed to client code in a reliable and accessible fashion.
    - On windows it's `HANDLE hEventQueueClient` in `THREADINFO`
    - On macOS it's a mach port but i don't know where to find it. this way lies dragons, you will drive yourself mad, your code will break. perish the thought.
 - Roughly: in `open_editor()`, you create the platform window, hook the events you want (on windows you implement this in your `windowproc`, on macOS you subclass `NSView` and override the handler methods) and then return from `open_editor()`. this is the hard part – UI toolkits, as we've established, aren't used to working like this.
 - You'll also want to create some way of doing redraws. VST2 has an "editor idle" opcode but it's up to the host how often it gets called and other plugin APIs (AUv2 for one) don't provide the facility.
    - Your easiest route here is to create a timer (`WM_TIMER` on windows, `CFRunLoopTimer` on mac) and hook it into the event loop. This is your redraw clock. Life gets more difficult while handling resize (especially on windows) but in a plugin UI you generally don't have to deal with that.
    - I am also interested in using `CVDisplayLink` on mac for better vblank and fps sync but it complicates locking and also you need to catch notifications for when the view moves to a different monitor... blegh. WIP. Timer method works fine.
 - **You will absolutely want a communication channel of some sort for the UI.**
    - At the bare minimum you will want DSP -> UI for things like knowing to redraw widgets if the parameter changes. `set_parameter()` *will* get called on the DSP thread and you cannot mess with the UI at all from it. This is **vital**, you **will** cause underruns [if you do].
    - Have a little SPSC message queue. Bounded, so you're not allocating. In my plugin I have my own circular buffer for the message storage and then I use a tiny signaling primitive (pipe on mac, eventfd on linux, I think EVENT on windows, but it's a libuv thing regardless) for notifying the UI event loop. Strictly speaking you could get away without a signaling primitive -- just use the buffer and check it in your redraw timer.
 - I really want to get multithreaded drawing working on windows and mac because openGL vblank usually works by blocking in `glSwapBuffers()` until the vblank is complete. Which means that the buffer swap is holding up the whole event loop, and that's just no bueno.
 - The name of the game with all of this deep platform-specific work is "sometimes what you're trying to do just won't work and you have to hack around it".

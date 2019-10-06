# Miscellaneous notes

## Brain dump by @wrl on Discord, 2019-10-05

here's a little bit of background on the event loop thing:

most UI toolkits assume that they own the top-level event loop (that is, they own `main()` and never return from it), and for plugins that doesn't work at all.

on linux, since the connection to the window server (under x11) is just a file descriptor, you're free to spin up a new thread and run your own event loop from there, and this is what I do in rtb. sort of. rtb assumes that it owns the event loop on linux, so in the plugin i spin up a new thread and run rtb from it.

nobody has decided how plugin UIs should work natively on wayland yet, so that's all WIP and rtb has no support for it presently.
on windows and macos, the connection to the window system is a platform-specific abstraction and is not exposed to client code in a reliable and accessible fashion. (on windows it's `HANDLE hEventQueueClient;` in `THREADINFO` and on macOS it's a mach port but i don't know where to find it. this way lies dragons, you will drive yourself mad, your code will break. perish the thought.)

roughly – in open_editor(), you create the platform window, hook the events you want (on windows you implement this in your windowproc, on macOS you subclass NSView and override the handler methods) and then *return from open_editor()*. this is the hard part – UI toolkits, as we've established, aren't used to working like this.
you'll also want to create some way of doing redraws. VST2 has an "editor idle" opcode but it's up to the host how often it gets called and other plugin APIs (AUv2 for one) don't provide the facility. your easiest route here is to create a timer (WM_TIMER on windows, CFRunLoopTimer on mac) and hook it into the event loop. this is your redraw clock. life gets more difficult while handling resize (especially on windows) but in a plugin UI you generally don't have to deal with that.

i am also interested in using CVDisplayLink on mac for better vblank and fps sync but it complicates locking and also you need to catch notifications for when the view moves to a different monitor... blegh. WIP. timer method works fine.

i might turn this into a blog post or sth also

oh RIGHT one last bit of detail

*you will absolutely want a communication channel of some sort for the UI*

at the bare minimum you will want DSP -> UI for things like knowing to redraw widgets if the parameter changes
`set_parameter()` *will* get called on the DSP thread and you cannot mess with the UI at all from it

this is **vital**, you **will** cause underruns

have a little spsc message queue. bounded, so you're not allocating

in my plugin I have my own circular buffer for the message storage and then i use a tiny signalling primitive (pipe on mac, eventfd on linux, i think EVENT on windows, but it's a libuv thing regardless) for notifying the UI event loop

strictly speaking you could get away without a signalling primitive – just use the buffer and check it in your redraw timer

fun additional musing: on linux, you must spin up a thread to own the event loop, for there is no other event loop to which you can subscribe. on windows and mac, you *absolutely cannot* create a new thread. everything will break in very sludgy unfortunate ways.

the reason why it breaks things is generally deep inside the platform library and "i'm not really sure"

i know on windows that the event dispatching that goes from parent window to child window can't cross threads, or the events just... don't show up

on mac, NSView is marked "main thread only"

so i assume there are assumptions in cocoa related to that – such as some data structures being insufficiently (or not at all) protected by mutexes or whatever

i *really* want to get multithreaded drawing working on windows and mac because openGL vblank usually works by blocking in `glSwapBuffers()` until the vblank is complete

which means that the buffer swap is *holding up the whole event loop*

and that's just no bueno

but i haven't really sat down to grind on that yet

the name of the game with all of this deep platform-specific work is "sometimes what you're trying to do just won't work and you have to hack around it"

toolkit developers don't buy each other beer when we go out, we buy each other whiskey

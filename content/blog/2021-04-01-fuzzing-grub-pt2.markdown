Title: Fuzzing grub, part 2: going faster
Date: 2021-06-14 17:10:00
Authors: Daniel Axtens
Category: Development
Tags: testing

Recently a set of 8 vulnerabilities were disclosed for the [grub bootloader](https://wiki.ubuntu.com/SecurityTeam/KnowledgeBase/GRUB2SecureBootBypass2021). I
found 2 of them (CVE-2021-20225 and CVE-2021-20233), and contributed a number of
other fixes for crashing bugs which we don't believe are exploitable. I found
them by applying fuzz testing to grub. Here's how.

This is a multi-part series: I think it will end up being 4 posts. I'm hoping to
cover:

 * [Part 1: getting started with fuzzing grub](/fuzzing-grub-part-1)
 * Part 2 (this post): going faster by doing lots more work
 * Part 3: fuzzing filesystems and more
 * Part 4: potential next steps and avenues for further work

Previously, we talked about some issues building grub with AFL++'s
instrumentation:


```
:::text
./configure --with-platform=emu --disable-grub-emu-sdl CC=$AFL_PATH/afl-cc
...
checking whether target compiler is working... no
configure: error: cannot compile for the target
```

It also doesn't work with `afl-gcc`.

We tried to trick configure:

```
:::shell
./configure --with-platform=emu --disable-grub-emu-sdl CC=clang CXX=clang++
make CC="$AFL_PATH/afl-cc" 
```

Sadly, things still break:

```
:::text
/usr/bin/ld: disk.module:(.bss+0x20): multiple definition of `__afl_global_area_ptr'; kernel.exec:(.bss+0xe078): first defined here
/usr/bin/ld: regexp.module:(.bss+0x70): multiple definition of `__afl_global_area_ptr'; kernel.exec:(.bss+0xe078): first defined here
/usr/bin/ld: blocklist.module:(.bss+0x28): multiple definition of `__afl_global_area_ptr'; kernel.exec:(.bss+0xe078): first defined here
```

The problem is the module linkage that I talked about in [part 1](/fuzzing-grub-part-1).
There is a link stage of sorts for the kernel (`kernel.exec`) and each module
(e.g. `disk.module`), so some AFL support code gets linked into each of
those. Then there's another link stage for `grub-emu` itself, which also tries
to bring in the same support code. The linker doesn't like the symbols being in
multiple places, which is fair enough.

There are (at least) 3 ways you could solve this. I'm going to call them the new
way, the hard way, and the ugly way.

The hard way: messing with makefiles
------------------------------------

We've been looking at fuzzing `grub-emu`. Building `grub-emu` links
`kernel.exec` and almost every `.module` file that grub produces into the
final binary. Maybe we could avoid our duplicate symbol problems entirely by
changing how we build things?

I didn't do this in my early work because, to be honest, I don't like working
with build systems and I'm not especially good at it. grub's build system is
based on autotools but is even more quirky than usual: rather than just having a
`Makefile.am`, we have `Makefile.core.def` which is used along with other things
to generate `Makefile.am`. It's a pretty cool system for making modules, but
it's not my idea of fun to work with.

But, for the sake of completeness, I tried again.

It gets unpleasant quickly. The generated `grub-core/Makefile.core.am` adds each module to `platform_PROGRAMS`, and then each is built with `LDFLAGS_MODULE = $(LDFLAGS_PLATFORM) -nostdlib $(TARGET_LDFLAGS_OLDMAGIC) -Wl,-r,-d`.

Basically, in the makefile this ends up being (e.g.):

```
:::make
tar.module$(EXEEXT): $(tar_module_OBJECTS) $(tar_module_DEPENDENCIES) $(EXTRA_tar_module_DEPENDENCIES) 
	@rm -f tar.module$(EXEEXT)
	$(AM_V_CCLD)$(tar_module_LINK) $(tar_module_OBJECTS) $(tar_module_LDADD) $(LIBS)
```

Ideally I don't want them to be linked at all; there's no benefit if
they're just going to be linked again.

You can't just collect the sources and build them into `grub-emu` - they all
have to built with different `CFLAGS`! So instead I spent some hours messing
around with the build system. Given some changes to the python script that
converts the `Makefile.*.def` files into `Makefile.am` files, plus some other
bits and pieces, we can build `grub-emu` by linking the object files rather than
the more-processed modules.

The build dies immediately after linking `grub-emu` in other components, and it
requires a bit of manual intervention to get the right things built in the right
order, but with all of those caveats, it's enough. It works, and you can turn on
things like ASAN, but getting there was hard, unrewarding and unpleasant. Let's
consider alternative ways to solve this problem.


The ugly way: patching AFL
--------------------------

What I did when finding the bugs was to observe that we only wanted AFL to link
in its extra instrumentation at certain points of the build process. So I
patched AFL to add an environment variable `AFL_DEFER_LIB` - which prevented AFL
adding its own instrumentation library when being called as a linker. I combined
this with the older CFG instrumentation, as the PCGUARD instrumentation brought
in a bunch of symbols from LLVM which I didn't want to also figure out how to
guard.

I then wrapped this in a horrifying script that basically built bits and pieces
of grub with the environment variable on or off, in order to at least get the
userspace tools and `grub-emu` built. Basically it set `AFL_DEFER_LIB` when
building all the modules and turned it off when building the userspace tools
and `grub-emu`.

This worked and it's what I used to find most of my bugs. But I'd probably not
recommend it, and I'm not sharing the source: it's extremely fragile and
brittle, the hard way is more generally applicable, and the new way is nicer.


The new way: AFL++ enhancements
-------------------------------

After posting part 1 of this series, I had a fascinating twitter conversation
with [@hackerschoice](https://twitter.com/hackerschoice), who pointed me to some
new work that had been done in AFL++ between when I started and when I published
part 1.

AFL++ now has the ability to dynamically detect some of the duplicate symbols -
which allows it to support plugins and modules better. We also need a linker
flag which instructs the linker to ignore the duplication rather than error
out. Together, they provide a significantly simpler way to instrument
`grub-emu`, avoiding all the issues I'd previously been fighting so hard to
address.

Now, with a modern AFL++, and the patch from part 1, you can sort out this
entire process like this:

```
:::shell
./bootstrap
./configure --with-platform=emu CC=clang CXX=clang++ --disable-grub-emu-sdl
make CC=/path/to/afl-clang-fast LDFLAGS="-Wl,--allow-multiple-definition"
```

The build will fail eventually, but you should have a `./grub-core/grub-emu`
binary which is all you need.


Going extra fast: `__AFL_INIT`
==============================

Now that we can compile with instrumentation, we can use `__AFL_INIT`. I'll
leave the precise details of how this works to the AFL docs, but in short it
allows us to do a bunch of early setup only once, and just fork the process
after the setup is done.

There's a patch that inserts a call to `__AFL_INIT` in the `grub-emu` start path
in [my GitHub repo](https://github.com/daxtens/grub/tree/fuzzing-pt2).

All up, this can lead to a 2x-3x speedup over the figures I saw in my part 1:

![afl-fuzz fuzzing grub, showing fuzzing happening](/images/dja/grub-fuzzing-pt2.png)


Finding more bugs with sanitisers
=================================

This also makes it really easy to compile with ASAN, which can often find more
issues:

```
env AFL_USE_ASAN=1 make CC=/path/to/afl-clang-fast LDFLAGS="-Wl,--allow-multiple-definition"
```

You can also try `AFL_USE_MSAN` if you want to find uses of uninitialised
memory.

That's all for part 2. In part 3 we'll look at fuzzing filesystems and
more. Hopefully there will be a quicker turnaround between part 2 and part 3
than there was between part 1 and part 2!

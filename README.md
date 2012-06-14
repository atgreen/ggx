How To Retarget the GNU Toolchain in 21 Patches
===============================================

The text below originally came from a series of blog posts I wrote
about porting the GNU Toolchain to a new target.  The original blog
site is down, so I have collected the contents of the articles and
have posted them here, along with the patches.

[This is not complete yet]

Patch 1: Naming the Target
-------------------------

The GNU tools are maintained in two separate repositories: src and
gcc. The src tree contains the GNU binutils, gdb, instruction set
simulators, cgen, newlib (a C runtime library), winsup (cygwin runtime
support) and more. The gcc tree contains the GNU compiler collection
and related utilities. These trees are designed to be merged into a
single tree (sometimes called a Cygnus tree) so you can configure and
build the entire toolchain in one go.

Those of you interested in some history might want to read the mostly
accurate article and thread here:
http://www.sourceware.org/ml/gdb/2000-09/msg00009.html.

I'm not going to build everything from a merged tree. The unfortunate
truth is that shared top level files in the two trees often get out of
sync with one another. For instance, sometimes the trees depend on
incompatible versions of the autotools and keeping everything happy
together is more work that I care to do at this point.

Now that we've decided on two trees, I'm going to start in the src
tree.

The only thing we need to decide at this point is the architecture's
name. I'm calling this "ggx" for now.

The patch after the jump adds the top level configury for our port. It
just tells the build system to recognize our target name, and to not
configure/build any of the subdirectories requiring target specific
work.

Once patched, you should be able to configure and build the toolchain
like so:

    $ ./configure --target=ggx-elf
    $ make

The "ggx-elf" target simply tells the build system that we want a
toolchain to generate ELF object files for the ggx architecture.

The only thing this builds are some support libraries for the host
system. It's not much, but it's a start!


Patch 2: BFD!
-------------

Today we'll build BFD, which is a library for reading and writing
object files. We're mostly concerned with ELF, but BFD handles a
number of other formats as well.

BFD is a backronym for Binary File Descriptor, but we all know it
originally stood for Big F'ing Deal. The BFD manual puts it thusly:

    The name came from a conversation David Wallace was having with
    Richard Stallman about the library: RMS said that it would be
    quite hard--David said "BFD". Stallman was right, but the name
    stuck.

David Henkel-Wallace (aka Gumby) was one of Cygnus' founders, and, at
least as I recall, his California license plate at one time was "GNU
BFD".

Now we have to make some real decisions about the ggx architecture,
but not very difficult ones. The most significant features we're
committing to in this patch are 32-bit words, 32-bit addresses and
8-bit bytes (see cpu-ggx.c). Two reasons for picking these values: (1)
the GNU tools are really good at targeting 32-bit systems and (2) 8-
and 16-bit systems won't run any interesting Free Software.

Also note that we're defining the ELF machine number for ggx (0xFEED
in include/elf/common.h). This number is encoded in all ggx object
files and executables so other tools (objdump, gdb, etc) can identify
them as such. There's a standards body somewhere that maintains the
master list of ELF machine numbers. I just picked a random number
right now, but it's something you need to worry about if you start
working with other tool vendors for your processor (JTAG hardware
debuggers, for instance).

Apply this patch to your src tree, rebuild, and you'll have a
libbfd. There's really not much to it. Most of this patch is made up
of configury changes. Click on the link below to see the patch.

We're very close to having a working assembler for ggx. There's just
one more infrastructure step to take care of first.

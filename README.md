<TriforceAFL_new>

A tool for simulation, dynamic analsis and fuzz of IoT firmware.
Combination of TriforceAFL, firmadyne and DECAF.

Firmadyne: we use its custom kernel and libnvram. 
  cd firmadyne
  See README in firmadyne and do as it says.
  Here, we test DIR-815_FIRMWARE_1.01.ZIP, a router firmware image based on mipsel cpu arch.
  Finally, we replace the run.sh in scratch/(num)/ with our modified one.

TriforceAFL: 
  run make
  
DECAF: upgraded to the newest qemu version 2.10.1
   It is included in qemu_mode/qemu dir.
   If there is something wrong with sleuthkit, plese comment or not comment the following code.
	LIBS="\$(SRC_PATH)/shared/sleuthkit/lib/libtsk.a -lbfd $LIBS"

   In our case, run ./configure --target-list=mipsel-softmmu
   run make

</TriforceAFL_new>

<ProjectTriforce>

New: For those looking to play with TriforceAFL and TLSF, Richard Johnson created a Dockerfile which installs both (and even builds a Linux kernel for you). It's available here <https://hub.docker.com/r/moflow/afl-triforce/tags/>.

Also new: afl-tmin now works with the forkserver!


https://github.com/nccgroup/TriforceAFL
Jesse Hertz <jesse.hertz@nccgroup.trust>
Tim Newsham <tim.newsham@nccgroup.trust>

This is a patched version of AFL that supports full-system
fuzzing using QEMU. The included QEMU has been updated to allow tracing
of branches when running a system emulator for x86_64.
Extra instructions have been added to start AFL's forkserver,
make fuzz settings, and mark the start and stop of test cases.

Note: not all of the AFL tools have been tested with the
new changes.  These tools have seen some testing:

   - afl-fuzz - patched to support -QQ 
   - afl-showmap - patched to support -QQ and forkserver (with batching)
   - afl-cmin - patched to support -QQ and use forkserver, 
                stdin no longer supported
   - afl-analyze - patched to support -QQ
   - afl-tmin - patched to support -QQ,  but does not support forkserver!

---
To build:

  make

---
To get a coverage map:

  echo hello > /tmp/hello
  ./afl-showmap -o coverage.txt -QQ -- \
      ./afl-qemu-system-trace -kernel ../bzImage \
      -initrd ../initramfs.cpio.gz -m 1G -nographic \
      -append "console=ttyS0" -aflFile /tmp/hello
  cat coverage.txt

---
To fuzz:

  # figure out what addrs to use below...
  egrep ' (panic|log_store)$' ../mykern/kallsyms
    ffffffff8108e570 t log_store
    ffffffff8181064b T panic

  mkdir inputs
  echo hello > inputs/hello
  ./afl-fuzz -i inputs -o outputs -QQ -- \
        afl-qemu-system-trace -kernel bzImage -initrd root.cpio.gz \
        -m 1G -nographic -append "console=ttyS0" \
        -aflPanicAddr ffffffff8181064b -aflDmesgAddr ffffffff8108e570 \
        -aflFile @@

(Note: unlike when using the "-Q" option, you must specify the
full command line to afl-qemu-system-trace when using the "-QQ" option).

For more details on how to use this modified version of AFL,
see our Linux syscall fuzzer at 
https://github.com/nccgroup/TriforceLinuxSyscallFuzzer.
  
---
New AFL flags:
   -QQ  - use qemu in full-system emulation rather than user-mode (-Q)

New QEMU flags:
   -aflFile - The name of the file containing fuzzer inputs
   -aflPanicAddr - A address of a kernel panic address for panic detection
   -aflDmesgAddr - Linux kernel address of dmesg logging function for
                   detecting logging and intercepting log messages

New QEMU instructions:
   0f 24 - aflCall
      edi=1 startForkserver(esi=enableTicks)
            Start AFL's fork server.  After this point each test
            will run in a separate forked child.  If enableTicks is
            non-zero, QEMU will re-enable the CPUs timer after
            forking a child, otherwise it will not be enabled.
      edi=2 getWork(esi=ptr, edx=sz)
            Fill ptr[0..sz] with the next input test case. Returns
            the actual size filled (<= sz).
      edi=3 startWork(esi=ptr)
            Tell AFL to start tracing.  The argument points to a
            buffer with two quadwords giving the start and end
            address of the code to trace.  Instructions outside
            of this range are not traced.
      edi=4 doneWork(esi=exitCode)
            Tell AFL that the test case has completed.  If a panic
            is detected, AFL will stop the test case immediately.
            Otherwise it will run until doneWork is called.
            The exitCode specified is returned to AFL.  (The
            code can, but currently does not, OR in the value 64
            to all exit codes if any dmesg logs were detected during
            the test case.)

New QEMU block driver:
   -drive filename=privmem:<name> 
      This block driver keeps the drive's image in copy-on-write memory
      so that changes are never persisted to disk.  Changes made by
      on test case are isolated from other test cases.

</ProjectTriforce>

==================
american fuzzy lop
==================

  Written and maintained by Michal Zalewski <lcamtuf@google.com>

  Copyright 2013, 2014, 2015, 2016 Google Inc. All rights reserved.
  Released under terms and conditions of Apache License, Version 2.0.

  For new versions and additional information, check out:
  http://lcamtuf.coredump.cx/afl/

  To compare notes with other users or get notified about major new features,
  send a mail to <afl-users+subscribe@googlegroups.com>.

  ** See QuickStartGuide.txt if you don't have time to read this file. **

1) Challenges of guided fuzzing
-------------------------------

Fuzzing is one of the most powerful and proven strategies for identifying
security issues in real-world software; it is responsible for the vast
majority of remote code execution and privilege escalation bugs found to date
in security-critical software.

Unfortunately, fuzzing is also relatively shallow; blind, random mutations
make it very unlikely to reach certain code paths in the tested code, leaving
some vulnerabilities firmly outside the reach of this technique.

There have been numerous attempts to solve this problem. One of the early
approaches - pioneered by Tavis Ormandy - is corpus distillation. The method
relies on coverage signals to select a subset of interesting seeds from a
massive, high-quality corpus of candidate files, and then fuzz them by
traditional means. The approach works exceptionally well, but requires such
a corpus to be readily available. In addition, block coverage measurements
provide only a very simplistic understanding of program state, and are less
useful for guiding the fuzzing effort in the long haul.

Other, more sophisticated research has focused on techniques such as program
flow analysis ("concolic execution"), symbolic execution, or static analysis.
All these methods are extremely promising in experimental settings, but tend
to suffer from reliability and performance problems in practical uses - and
currently do not offer a viable alternative to "dumb" fuzzing techniques.

2) The afl-fuzz approach
------------------------

American Fuzzy Lop is a brute-force fuzzer coupled with an exceedingly simple
but rock-solid instrumentation-guided genetic algorithm. It uses a modified
form of edge coverage to effortlessly pick up subtle, local-scale changes to
program control flow.

Simplifying a bit, the overall algorithm can be summed up as:

  1) Load user-supplied initial test cases into the queue,

  2) Take next input file from the queue,

  3) Attempt to trim the test case to the smallest size that doesn't alter
     the measured behavior of the program,

  4) Repeatedly mutate the file using a balanced and well-researched variety
     of traditional fuzzing strategies,

  5) If any of the generated mutations resulted in a new state transition
     recorded by the instrumentation, add mutated output as a new entry in the
     queue.

  6) Go to 2.

The discovered test cases are also periodically culled to eliminate ones that
have been obsoleted by newer, higher-coverage finds; and undergo several other
instrumentation-driven effort minimization steps.

As a side result of the fuzzing process, the tool creates a small,
self-contained corpus of interesting test cases. These are extremely useful
for seeding other, labor- or resource-intensive testing regimes - for example,
for stress-testing browsers, office applications, graphics suites, or
closed-source tools.

The fuzzer is thoroughly tested to deliver out-of-the-box performance far
superior to blind fuzzing or coverage-only tools.

3) Instrumenting programs for use with AFL
------------------------------------------

When source code is available, instrumentation can be injected by a companion
tool that works as a drop-in replacement for gcc or clang in any standard build
process for third-party code.

The instrumentation has a fairly modest performance impact; in conjunction with
other optimizations implemented by afl-fuzz, most programs can be fuzzed as fast
or even faster than possible with traditional tools.

The correct way to recompile the target program may vary depending on the
specifics of the build process, but a nearly-universal approach would be:

$ CC=/path/to/afl/afl-gcc ./configure
$ make clean all

For C++ programs, you'd would also want to set CXX=/path/to/afl/afl-g++.

The clang wrappers (afl-clang and afl-clang++) can be used in the same way;
clang users may also opt to leverage a higher-performance instrumentation mode,
as described in llvm_mode/README.llvm.

When testing libraries, you need to find or write a simple program that reads
data from stdin or from a file and passes it to the tested library. In such a
case, it is essential to link this executable against a static version of the
instrumented library, or to make sure that the correct .so file is loaded at
runtime (usually by setting LD_LIBRARY_PATH). The simplest option is a static
build, usually possible via:

$ CC=/path/to/afl/afl-gcc ./configure --disable-shared

Setting AFL_HARDEN=1 when calling 'make' will cause the CC wrapper to
automatically enable code hardening options that make it easier to detect
simple memory bugs.

PS. ASAN users are advised to review notes_for_asan.txt file for important
caveats.

4) Instrumenting binary-only apps
---------------------------------

When source code is *NOT* available, the fuzzer offers experimental support for
fast, on-the-fly instrumentation of black-box binaries. This is accomplished
with a version of QEMU running in the lesser-known "user space emulation" mode.

QEMU is a project separate from AFL, but you can conveniently build the
feature by doing:

$ cd qemu_mode
$ ./build_qemu_support.sh

For additional instructions and caveats, see qemu_mode/README.qemu.

The mode is approximately 2-5x slower than compile-time instrumentation, is
less conductive to parallelization, and may have some other quirks.

5) Choosing initial test cases
------------------------------

To operate correctly, the fuzzer requires one or more starting file that
contains a good example of the input data normally expected by the targeted
application. There are two basic rules:

  - Keep the files small. Under 1 kB is ideal, although not strictly necessary.
    For a discussion of why size matters, see perf_tips.txt.

  - Use multiple test cases only if they are functionally different from
    each other. There is no point in using fifty different vacation photos
    to fuzz an image library.

You can find many good examples of starting files in the testcases/ subdirectory
that comes with this tool.

PS. If a large corpus of data is available for screening, you may want to use
the afl-cmin utility to identify a subset of functionally distinct files that
exercise different code paths in the target binary.

6) Fuzzing binaries
-------------------

The fuzzing process itself is carried out by the afl-fuzz utility. This program
requires a read-only directory with initial test cases, a separate place to
store its findings, plus a path to the binary to test.

For target binaries that accept input directly from stdin, the usual syntax is:

$ ./afl-fuzz -i testcase_dir -o findings_dir /path/to/program [...params...]

For programs that take input from a file, use '@@' to mark the location in
the target's command line where the input file name should be placed. The
fuzzer will substitute this for you:

$ ./afl-fuzz -i testcase_dir -o findings_dir /path/to/program @@

You can also use the -f option to have the mutated data written to a specific
file. This is useful if the program expects a particular file extension or so.

Non-instrumented binaries can be fuzzed in the QEMU mode (add -Q in the command
line) or in a traditional, blind-fuzzer mode (specify -n).

You can use -t and -m to override the default timeout and memory limit for the
executed process; rare examples of targets that may need these settings touched
include compilers and video decoders.

Tips for optimizing fuzzing performance are discussed in perf_tips.txt.

Note that afl-fuzz starts by performing an array of deterministic fuzzing
steps, which can take several days. If you want quick & dirty results right
away, akin to zzuf or honggfuzz, add the -d option to the command line.

7) Interpreting output
----------------------

See the status_screen.txt file for information on how to interpret the
displayed stats and monitor the health of the process. Be sure to consult this
file especially if any UI elements are highlighted in red.

The fuzzing process will continue until you press Ctrl-C. At minimum, you want
to allow the fuzzer to complete one queue cycle, which may take anywhere from a
couple of hours to a week or so.

There are three subdirectories created within the output directory and updated
in real time:

  - queue/   - test cases for every distinctive execution path, plus all the
               starting files given by the user. This is the synthesized corpus
               mentioned in section 2.

               Before using this corpus for any other purposes, you can shrink
               it to a smaller size using the afl-cmin tool. The tool will find
               a smaller subset of files offering equivalent edge coverage.

  - crashes/ - unique test cases that cause the tested program to receive a
               fatal signal (e.g., SIGSEGV, SIGILL, SIGABRT). The entries are 
               grouped by the received signal.

  - hangs/   - unique test cases that cause the tested program to time out. Note
               that when default (aggressive) timeout settings are in effect,
               this can be slightly noisy due to latency spikes and other
               natural phenomena.

Crashes and hangs are considered "unique" if the associated execution paths
involve any state transitions not seen in previously-recorded faults. If a
single bug can be reached in multiple ways, there will be some count inflation
early in the process, but this should quickly taper off.

The file names for crashes and hangs are correlated with parent, non-faulting
queue entries. This should help with debugging.

When you can't reproduce a crash found by afl-fuzz, the most likely cause is
that you are not setting the same memory limit as used by the tool. Try:

$ LIMIT_MB=50
$ ( ulimit -Sv $[LIMIT_MB << 10]; /path/to/tested_binary ... )

Change LIMIT_MB to match the -m parameter passed to afl-fuzz. On OpenBSD,
also change -Sv to -Sd.

Any existing output directory can be also used to resume aborted jobs; try:

$ ./afl-fuzz -i- -o existing_output_dir [...etc...]

If you have gnuplot installed, you can also generate some pretty graphs for any
active fuzzing task using afl-plot. For an example of how this looks like,
see http://lcamtuf.coredump.cx/afl/plot/.

8) Parallelized fuzzing
-----------------------

Every instance of afl-fuzz takes up roughly one core. This means that on
multi-core systems, parallelization is necessary to fully utilize the hardware.
For tips on how to fuzz a common target on multiple cores or multiple networked
machines, please refer to parallel_fuzzing.txt.

9) Fuzzer dictionaries
----------------------

By default, afl-fuzz mutation engine is optimized for compact data formats -
say, images, multimedia, compressed data, regular expression syntax, or shell
scripts. It is somewhat less suited for languages with particularly verbose and
redundant verbiage - notably including HTML, SQL, or JavaScript.

To avoid the hassle of building syntax-aware tools, afl-fuzz provides a way to
seed the fuzzing process with an optional dictionary of language keywords,
magic headers, or other special tokens associated with the targeted data type
- and use that to reconstruct the underlying grammar on the go:

  http://lcamtuf.blogspot.com/2015/01/afl-fuzz-making-up-grammar-with.html

To use this feature, you first need to create a dictionary in one of the two
formats discussed in testcases/README.testcases; and then point the fuzzer to
it via the -x option in the command line.

There is no way to provide more structured descriptions of the underlying
syntax, but the fuzzer will likely figure out some of this based on the
instrumentation feedback alone. This actually works in practice, say:

  http://lcamtuf.blogspot.com/2015/04/finding-bugs-in-sqlite-easy-way.html

PS. Even when no explicit dictionary is given, afl-fuzz will try to extract
existing syntax tokens in the input corpus by watching the instrumentation
very closely during deterministic byte flips. This works for some types of
parsers and grammars, but isn't nearly as good as the -x mode.

10) Crash triage
----------------

The coverage-based grouping of crashes usually produces a small data set that
can be quickly triaged manually or with a very simple GDB or Valgrind script.
Every crash is also traceable to its parent non-crashing test case in the
queue, making it easier to diagnose faults.

Having said that, it's important to acknowledge that some fuzzing crashes can be
difficult quickly evaluate for exploitability without a lot of debugging and
code analysis work. To assist with this task, afl-fuzz supports a very unique
"crash exploration" mode enabled with the -C flag.

In this mode, the fuzzer takes one or more crashing test cases as the input,
and uses its feedback-driven fuzzing strategies to very quickly enumerate all
code paths that can be reached in the program while keeping it in the
crashing state.

Mutations that do not result in a crash are rejected; so are any changes that
do not affect the execution path.

The output is a small corpus of files that can be very rapidly examined to see
what degree of control the attacker has over the faulting address, or whether
it is possible to get past an initial out-of-bounds read - and see what lies
beneath.

Oh, one more thing: for test case minimization, give afl-tmin a try. The tool
can be operated in a very simple way:

$ ./afl-tmin -i test_case -o minimized_result -- /path/to/program [...]

The tool works with crashing and non-crashing test cases alike. In the crash
mode, it will happily accept instrumented and non-instrumented binaries. In the
non-crashing mode, the minimizer relies on standard AFL instrumentation to make
the file simpler without altering the execution path.

The minimizer accepts the -m, -t, -f and @@ syntax in a manner compatible with
afl-fuzz.

Another recent addition to AFL is the afl-analyze tool. It takes an input
file, attempts to sequentially flip bytes, and observes the behavior of the
tested program. It then color-codes the input based on which sections appear to
be critical, and which are not; while not bulletproof, it can often offer quick
insights into complex file formats. More info about its operation can be found
near the end of technical_details.txt.

11) Common-sense risks
----------------------

Please keep in mind that, similarly to many other computationally-intensive
tasks, fuzzing may put strain on your hardware and on the OS. In particular:

  - Your CPU will run hot and will need adequate cooling. In most cases, if
    cooling is insufficient or stops working properly, CPU speeds will be
    automatically throttled. That said, especially when fuzzing on less
    suitable hardware (laptops, smartphones, etc), it's not entirely impossible
    for something to blow up.

  - Targeted programs may end up erratically grabbing gigabytes of memory or
    filling up disk space with junk files. AFL tries to enforce basic memory
    limits, but can't prevent each and every possible mishap. The bottom line
    is that you shouldn't be fuzzing on systems where the prospect of data loss
    is not an acceptable risk.

  - Fuzzing involves billions of reads and writes to the filesystem. On modern
    systems, this will be usually heavily cached, resulting in fairly modest
    "physical" I/O - but there are many factors that may alter this equation.
    It is your responsibility to monitor for potential trouble; with very heavy
    I/O, the lifespan of many HDDs and SSDs may be reduced.

    A good way to monitor disk I/O on Linux is the 'iostat' command:

    $ iostat -d 3 -x -k [...optional disk ID...]

12) Known limitations & areas for improvement
---------------------------------------------

Here are some of the most important caveats for AFL:

  - AFL detects faults by checking for the first spawned process dying due to
    a signal (SIGSEGV, SIGABRT, etc). Programs that install custom handlers for
    these signals may need to have the relevant code commented out. In the same
    vein, faults in child processed spawned by the fuzzed target may evade
    detection unless you manually add some code to catch that.

  - As with any other brute-force tool, the fuzzer offers limited coverage if
    encryption, checksums, cryptographic signatures, or compression are used to
    wholly wrap the actual data format to be tested.

    To work around this, you can comment out the relevant checks (see
    experimental/libpng_no_checksum/ for inspiration); if this is not possible,
    you can also write a postprocessor, as explained in
    experimental/post_library/.

  - There are some unfortunate trade-offs with ASAN and 64-bit binaries. This
    isn't due to any specific fault of afl-fuzz; see notes_for_asan.txt for
    tips.

  - There is no direct support for fuzzing network services, background
    daemons, or interactive apps that require UI interaction to work. You may
    need to make simple code changes to make them behave in a more traditional
    way. Preeny may offer a relatively simple option, too - see:
    https://github.com/zardus/preeny

    Some useful tips for modifying network-based services can be also found at:
    https://www.fastly.com/blog/how-to-fuzz-server-american-fuzzy-lop

  - AFL doesn't output human-readable coverage data. If you want to monitor
    coverage, use afl-cov from Michael Rash: https://github.com/mrash/afl-cov

Beyond this, see INSTALL for platform-specific tips.

13) Special thanks
------------------

Many of the improvements to afl-fuzz wouldn't be possible without feedback,
bug reports, or patches from:

  Jann Horn                             Hanno Boeck
  Felix Groebert                        Jakub Wilk
  Richard W. M. Jones                   Alexander Cherepanov
  Tom Ritter                            Hovik Manucharyan
  Sebastian Roschke                     Eberhard Mattes
  Padraig Brady                         Ben Laurie
  @dronesec                             Luca Barbato
  Tobias Ospelt                         Thomas Jarosch
  Martin Carpenter                      Mudge Zatko
  Joe Zbiciak                           Ryan Govostes
  Michael Rash                          William Robinet
  Jonathan Gray                         Filipe Cabecinhas
  Nico Weber                            Jodie Cunningham
  Andrew Griffiths                      Parker Thompson
  Jonathan Neuschfer                    Tyler Nighswander
  Ben Nagy                              Samir Aguiar
  Aidan Thornton                        Aleksandar Nikolich
  Sam Hakim                             Laszlo Szekeres
  David A. Wheeler                      Turo Lamminen
  Andreas Stieger                       Richard Godbee
  Louis Dassy                           teor2345
  Alex Moneger                          Dmitry Vyukov
  Keegan McAllister                     Kostya Serebryany
  Richo Healey                          Martijn Bogaard
  rc0r                                  Jonathan Foote
  Christian Holler                      Dominique Pelle
  Jacek Wielemborek                     Leo Barnes
  Jeremy Barnes                         Jeff Trull
  Guillaume Endignoux                   ilovezfs
  Daniel Godas-Lopez                    Franjo Ivancic

Thank you!

14) Contact
-----------

Questions? Concerns? Bug reports? The author can be usually reached at
<lcamtuf@google.com>.

There is also a mailing list for the project; to join, send a mail to
<afl-users+subscribe@googlegroups.com>. Or, if you prefer to browse
archives first, try:

  https://groups.google.com/group/afl-users

PS. If you wish to submit raw code to be incorporated into the project, please
be aware that the copyright on most of AFL is claimed by Google. While you do
retain copyright on your contributions, they do ask people to agree to a simple
CLA first:

  https://cla.developers.google.com/clas

Sorry about the hassle. Of course, no CLA is required for feature requests or
bug reports.

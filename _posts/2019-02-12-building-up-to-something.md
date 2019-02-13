---
layout: default
title: Building up to something
---

The other day I needed to build a [conda][conda] package. If you're not
familiar, it's a language-agnostic package manager with good support for Python
and R.  For packaging R packages, it's got significant advantages over the
built-in mechanisms: compiled code can be distributed, users can have multiple
environments with different packages and different versions of R on the same
machine without interfering.

Thanks to [Ray Donnelly][Ray Donnelly], `conda-build` has great support for
generating recipes for conda packages based on R packages from [CRAN][CRAN].
There's also a standard [Docker image][docker image] that we can use to build
packages that will be compatible with rest of the Anaconda distribution.

```bash
docker run -it -v $PWD:/code conda/c3i-linux-64
conda skeleton cran sdmtools
conda build r-sdmtools/
```

It starts out looking good: it loads the library, byte-compiles it, and turns
it into a conda package.

```
** R
** byte-compile and prepare package for lazy loading
** help
*** installing help indices
** building package indices
** testing if installed package can be loaded
* creating tarball
packaged installation of ‘SDMTools’ as ‘SDMTools_1.1-221_R_x86_64-conda_cos6-linux-gnu.tar.gz’
* DONE (SDMTools)

...

> library('SDMTools')
Error: package or namespace load failed for ‘SDMTools’ in dyn.load(file, DLLpath = DLLpath, ...):
 unable to load shared object '/opt/conda/conda-bld/r-sdmtools_1544077144220/_test_env_.../lib/R/library/SDMTools/libs/SDMTools.so':
  /opt/conda/conda-bld/r-sdmtools_1544077144220/_test_env_.../lib/R/library/SDMTools/libs/SDMTools.so: undefined symbol: X
Execution halted
```

Aww, crumb. This is not a totally new problem to have, sometimes the generated
conda recipe is missing some dependencies. Generally just searching the
`undefined symbol: X` error message is good enough to get going: you can find
whatever library the symbol should be coming from, and add that to the
dependencies.  Let's see what I can find on Google...

<blockquote class="blockquote">
I am using conda as a package manager for R packages that are used. I am currently having trouble getting one of the packages to build properly...
<footer class="blockquote-footer">
A <a href="https://groups.google.com/a/continuum.io/forum/#!topic/conda/LHeifquVyOo">Google groups thread</a> from a few years ago
</footer>
</blockquote>

That's pretty interesting, someone who has the same problem. The initial poster
has done some good investigation, but it seems they ran into a dead-end.
There's no solution posted, so I'll have to keep looking.

<blockquote class="blockquote">
I’m a bit stuck with this issue, though. It’s coming from an R package that is already in Nix:
...
<footer class="blockquote-footer">
A <a href="https://discourse.nixos.org/t/shared-object-error-in-rpackages-seurat/211">Discourse thread</a> in the NixOS forums, from earlier this year
</footer>
</blockquote>

Even more interesting! Someone who is not using `conda`, but is running into exactly
the same issue!

They seem to have got a little deeper - there's something wrong about the
symbol table.  Let's see if I have the same problem.  I can take a look 
at the build artifacts that `conda-build` has left over:

```bash
mkdir broken-package
mv /opt/conda/conda-bld/broken/r-sdmtools-1.1_221-r351h96ca727_0.tar.bz2 .
mkdir source-dir
mv /opt/conda/conda-bld/r-sdmtools_1544077144220/work_moved_r-sdmtools-1.1_221-r351h96ca727_0_linux-64/* source-dir/
tar -xf r-sdmtools-1.1_221-r351h96ca727_0.tar.bz2 -C broken-package/
```

There's a great Python library called [Lief][lief] that I can use to examine
the shared libraries: 

```bash
conda install py-lief
```

```py
import lief

broken = lief.parse('broken-package/lib/R/library/SDMTools/libs/SDMTools.so')
working = lief.parse('source-dir/src/SDMTools.so')

diff = set(working.symbols).symmetric_difference(broken.symbols)
for symbol in diff:
    print(symbol)
```

The output confirms, my symbol table is different:

```text
b                             OBJECT    GLOBAL    8020      8         * Global *
X                             OBJECT    GLOBAL    8020      8         * Global *
```

The two different results give a good hint on where to look: what do conda and
NixOS have in common? A little investigation shows that conda uses
[patchelf][patchelf] (a NixOS project) to make libraries
[relocatable][conda-build relocatable]. It uses a field called [RPATH][Rpath]
to change where the runtime linker looks for shared libraries, so libraries
that are in conda's environment are preferred over system libraries of the same
name. I can use `lief` to compare the values in the original and the broken
library:

```py
import itertools
for entry in itertools.chain(working.dynamic_entries, broken.dynamic_entries):
    if entry.tag == lief.ELF.DYNAMIC_TAGS.RPATH:
        print(entry)
```

```text
RPATH               249       /opt/conda/conda-bld/r-sdmtools_1544077144220/_build_env/lib:/opt/conda/conda-bld/r-sdmtools_1544077144220/_h_env_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_placehold_pl/lib
RPATH               249       $ORIGIN/../../../..:$ORIGIN/../../../lib
```

Could that change be what's corrupting my symbol table? This [line from
patchelf][patchelf memset] seems like a good candidate! It's replacing the 
previous RPATH value with `X`, the name of my new symbol.

```c
    /* Zero out the previous rpath to prevent retained dependencies in
       Nix. */
    unsigned int rpathSize = 0;
    if (rpath) {
        rpathSize = strlen(rpath);
        memset(rpath, 'X', rpathSize);
    }
```

But why is the RPATH change affecting the symbol table? To explain that, I
need to dig into the [ELF format][ELF] a bit. That's the binary format used
for executables and shared libraries on Linux. An ELF file is composed of multiple
sections, so I should try to determine which sections to look at. Lief can tell
us what sections exist in my library:

```py
for section in working.sections:
    print(section.name)
```
```text
.hash
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.init
.plt
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.ctors
.dtors
.dynamic
.got
.data
.bss
.comment
.symtab
.strtab
.shstrtab
```

The `.dynsym` section sounds good - short for dynamic symbol. Unfortunately,
Lief hides some of the implementation details here that turn out to be important.
So I'll have to dig into the dynamic symbols myself.

I have a 64-bit library, so I can use the definition of a symbol for ELF64.
```py
assert broken.type == working.type == lief.ELF.ELF_CLASS.CLASS64
```

I'm borrowing this description from the Lief C++ [source code][ELF64 struct].

```cpp
/** Symbol table entries for ELF64. */
struct Elf64_Sym {
  Elf64_Word      st_name;  /**< Symbol name (index into string table) */
  unsigned char   st_info;  /**< Symbol's type and binding attributes */
  unsigned char   st_other; /**< Must be zero; reserved */
  Elf64_Half      st_shndx; /**< Which section (header tbl index) it's defined in */
  Elf64_Addr      st_value; /**< Value or address associated with the symbol */
  Elf64_Xword     st_size;  /**< Size of the symbol */
};
```

`st_name` is an interesting field here. It looks like there's some indirection
involved - the symbol name is not embedded directly in the `.dynsym` section.
I'm guessing that the `string table` being referred to is `.dynstr`.

The `struct` module in Python provides an easy way to do parsing of simple
binary formats like this.

```py
import struct
Symbol = struct.Struct('IBBHQQ')
```

And I should do a quick sanity-check that the format looks correct:

```py
symtable = bytes(broken.get_section('.dynsym').content)
assert len(symtable) == len(broken.dynamic_symbols) * Symbol.size
```

The `st_size` field doesn't look like the size of the symbol name, so I'm
assuming the strings are stored as C-style null-terminated strings. I can now
look through the symbol table and show the offset into the string table for
the corrupted symbol.

```py
strtable = bytes(broken.get_section('.dynstr').content)
for name, info, other, shndx, value, size in Symbol.iter_unpack(symtable):
    if name > 0:
        cname = strtable[name:strtable.find(b'\0', name)].decode('ascii')
        if cname in ('X', 'b'):
            print(f'{name:04d}    {cname}')
```

```text
0904    X
```

And if I compare that against the content of the corrupted string table:

```py
def print_strtab(section):
    strdata = section.content
    chunklen = 64
    for i in range(0, len(strdata), chunklen):
        values = bytes(strdata[i:i + chunklen]).replace(b'\0', b' ').decode('ascii')
        print(f'{i:04d}    {values}')

print_strtab(broken.get_section('.dynstr'))
```
```
0000     __gmon_start__ _init _fini _ITM_deregisterTMCloneTable _ITM_reg
0064    isterTMCloneTable __cxa_finalize Tracer out nrow ncol R_NaInt Co
0128    ntourTracing ccl Rf_coerceVector Rf_protect INTEGER R_DimSymbol
0192    Rf_getAttrib Rf_allocMatrix ans Rf_unprotect getmin REAL R_NaRea
0256    l R_IsNA movewindow Rf_length projectedPS Rf_allocVector __stack
0320    _chk_fail geographicPS pip epsilon atan2 TWOPI Slope sqrt Aspect
0384     small_num Dest sincos f Dist atan __isnan writeascdata STRING_E
0448    LT R_CHAR fopen __fprintf_chk fwrite fputc fclose R_NilValue lib
0512    R.so libc.so.6 _edata __bss_start _end GLIBC_2.3.4 GLIBC_2.4 GLI
0576    BC_2.2.5 $ORIGIN/../../../..:$ORIGIN/../../../lib XXXXXXXXXXXXXX
0640    XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
0704    XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
0768    XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
0832    XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
0896    XXXXXXXXX
```

And the working string table:
```py
print_strtab(working.get_section('.dynstr'))
```
```
0000     __gmon_start__ _init _fini _ITM_deregisterTMCloneTable _ITM_reg
0064    isterTMCloneTable __cxa_finalize Tracer out nrow ncol R_NaInt Co
0128    ntourTracing ccl Rf_coerceVector Rf_protect INTEGER R_DimSymbol
0192    Rf_getAttrib Rf_allocMatrix ans Rf_unprotect getmin REAL R_NaRea
0256    l R_IsNA movewindow Rf_length projectedPS Rf_allocVector __stack
0320    _chk_fail geographicPS pip epsilon atan2 TWOPI Slope sqrt Aspect
0384     small_num Dest sincos f Dist atan __isnan writeascdata STRING_E
0448    LT R_CHAR fopen __fprintf_chk fwrite fputc fclose R_NilValue lib
0512    R.so libc.so.6 _edata __bss_start _end GLIBC_2.3.4 GLIBC_2.4 GLI
0576    BC_2.2.5 /opt/conda/conda-bld/r-sdmtools_1544077144220/_build_en
0640    v/lib:/opt/conda/conda-bld/r-sdmtools_1544077144220/_h_env_place
0704    hold_placehold_placehold_placehold_placehold_placehold_placehold
0768    _placehold_placehold_placehold_placehold_placehold_placehold_pla
0832    cehold_placehold_placehold_placehold_placehold_placehold_placeho
0896    ld_pl/lib
```

I can see the RPATH is in there, along with all my symbols! But where's
the definition of the `b` symbol? It looks like the index is pointing to
the end of the RPATH - which before `patchelf` got involved, ended with
`b`.

Whatever is creating my library is smart enough to share strings in this
string table between different symbols and entries - so long as the suffixes
are shared. I want to find out where that happens! 

On Linux, these `.so` (shared object) files are created by the linker. The
compiler turns each source code file into a `.o` (object file), and the files
are combined together into a shared object. The linker gathers up all 
the symbols exported in all the object files, and creates a symbol table
representing them.

If I look in the source for the GCC binutils, I can find this comment
describing exactly what I'm seeing:

```c
      /* Loop over the sorted array and merge suffixes.  Start from the
	 end because we want eg.
	 s1 -> "d"
	 s2 -> "bcd"
	 s3 -> "abcd"
	 to end up as
	 s3 -> "abcd"
	 s2 _____^
	 s1 _______^
	 ie. we don't want s1 pointing into the old s2.  */
```

Why bother doing this? How often do symbols share suffixes? I'm curious to find
out how much space savings there would be in a typical library, and on a
typical system.

The other question I have is why RPATH is included in this `.dynstr` section.
I think a hint is that I found it in the list of `dynamic_entries` in the
library, so probably everything in that list uses this string table.

So what now? I know this library got corrupted because it's declaring a dynamic
symbol called `b`. That seems kind of egregious, is that really needed? Perhaps
it doesn't need to be exported at all. And indeed, looking at the source code,
it's clear it is just a constant, so we should declare it as such:

```diff
diff -u source-dir/src/vincenty.geodesics.c patched-dir/src/vincenty.geodesics.c
--- source-dir/src/vincenty.geodesics.c	2014-08-05 02:29:35.000000000 -0700
+++ patched-dir/src/vincenty.geodesics.c	2019-02-12 23:09:25.000000000 -0800
@@ -7,7 +7,7 @@
 #include <math.h>
 
 //define some global constants
-double a = 6378137, b = 6356752.3142,  f = 1/298.257223563;  // WGS-84 ellipsiod
+static const double a = 6378137, b = 6356752.3142,  f = 1/298.257223563;  // WGS-84 ellipsiod
 
 /*
  * Calculate destination point given start point lat/long (numeric degrees), 
```

After applying this patch in our conda recipe, our package successfully builds,
and runs!


TLDR:

The GCC linker combines strings in string tables together that share a common
suffix. This breaks `patchelf` which naively replaces entries in the string table,
causing our conda build to fail.

[conda]: https://github.com/conda/conda
[docker image]: https://hub.docker.com/r/conda/c3i-linux-64
[CRAN]: https://cran.r-project.org/
[lief]: https://lief.quarkslab.com/
[ELF]: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
[Rpath]: https://en.wikipedia.org/wiki/Rpath
[ELF64 struct]: https://github.com/lief-project/LIEF/blob/406115c8d097da0b61f00b2bb7b2442322ffc5d1/include/LIEF/ELF/structures.inc#L106-L115
[Ray Donnelly]: https://github.com/mingwandroid
[conda-build relocatable]: https://github.com/conda/conda-build/blob/7dac1cbad5195f719bde1eaeb0d5795186dc0eb0/docs/source/make-relocatable.rst
[wheel PEP]: https://www.python.org/dev/peps/pep-0427/
[patchelf]: https://github.com/NixOS/patchelf
[patchelf memset]: https://github.com/NixOS/patchelf/blob/27ffe8ae871e7a186018d66020ef3f6162c12c69/src/patchelf.cc#L1261-L1267
[binutils merge]: https://github.com/bminor/binutils-gdb/blob/01c7ae818bd6c0b5d797091ec1664bcaed705d40/bfd/elf-strtab.c#L390-L403


# BurrTools Notes

Notes I've taken on BurrTools internals for the purposes of understanding the
code and algorithms.

You can find the latest version of BurrTools at [github.com/burr-tools/burr-tools](https://github.com/burr-tools/burr-tools).
These notes reference line numbers from commit [29701df](https://github.com/burr-tools/burr-tools/tree/29701dfefc71f686d4e928725de4edf06040d382).
A copy of the code is committed in `burr-tools/` for easy linking.

In my quest for understanding the assembler I've done some substantial code
cleanup in a fork, plus other various improvements of the code. You can find
that work at [github.com/mbrown1413/burr-tools](https://github.com/mbrown1413/burr-tools).

## Topics

* [Data Structures](data-structures.md)
* [Assemblers](assemblers.md)
* [Assembly Algorithm](assembly-algorithm.md)
* [Grids](grids.md)
* [Symmetry](symmetry.md)

## Terminology

See also: [BurrTools User Guide: Concepts and Definitions](https://burrtools.sourceforge.net/gui-doc/ConceptsandDefinitions.html)

* **Voxel**: A unit of volume in 3D space.
* **Fixed Voxel**: A voxel in a solution shape which must be filled in a solution. Also referred to as "filled voxels".
* **Variable Voxel**: A voxel in a solution shape which is only optionally filled in a solution.

-----

* **Shape**: The actual shapes created in the UI for individual puzzle pieces including the solution shape.
    * Stored in [puzzle_c::shapes](burr-tools/src/lib/puzzle.h#L61).
    * Often passed around as a shape ID, referencing the shape index in the currently opened puzzle.
* **Part**: Stores how many of a shape is used in a problem.
    * Stored in [problem_c::parts](burr-tools/src/lib/problem.h#L84) and data structure defined in [class part_c](burr-tools/src/lib/problem.cpp#L48).
    * The XML for a problem stores a part using the "shape" tag (see [problem_c::save](burr-tools/src/lib/problem.cpp#L129)).
* **Piece**: An individual use of a shape in a problem.
    * Part of a problem but derived from a problem's list of parts, for example in [problem_c::getNumberOfPieces](burr-tools/src/lib/problem.cpp#L761).
    * For example: If a problem has one part with `shapeId=1, min=1, max=3`, that problem has 3 pieces with `shapeId=1`.

## Build Notes

Initial build:

    autoreconf -fvi
    ./configure
    make

Debug build:

    $ ./configure CPPFLAGS="-g"
    $ make clean
    $ make
    $ gdb ./src/burrTxt2 ...

Profiling (TOOD: How to make this work with threading):

    $ ./configure CPPFLAGS="-g -pg" LDFLAGS="-g -pg"
    $ make clean
    $ make
    $ ./src/burrTxt2 ...
    $ gprof src/burrTxt2

Reset to default build:

    $ ./configure
    $ make clean
    $ make

## Resources

* [BurrTools User Guide](https://burrtools.sourceforge.net/gui-doc/toc.html)
* [BurrTools Library documentation](https://burrtools.sourceforge.net/lib-doc/index.html)
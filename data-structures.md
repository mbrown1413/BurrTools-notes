# Data Structures

This isn't an exhaustive overview. Just read the code for that! Here we'll
highlight the main parts you'll need to understand how things are put together.

The XML save functionality starts at `puzzle_c::save()` which calls `*::save()`
for its complex attributes like `problem_c::problems`, which in turn calls more
`save()` functions, etc.

## [puzzle_c](burr-tools/src/lib/puzzle.h#L47)
* [gridType_c](burr-tools/src/lib/gridtype.h#L46) *gt;
* vector\<[voxel_c](burr-tools/src/lib/voxel.h#L51)*\> shapes;
* vector\<[problem_c](burr-tools/src/lib/problem.h#L69)\> problems;

The puzzle is the main data structure saved in XML. It stores shapes and
problems as vectors (lists). In other data structures the shapes are referred
to by ID, which is just their index into their vector in the puzzle. This means
shapes aren't stored or passed around in duplicate, but it also makes modifying
the list of shapes a bit more complicated, since everywhere that stores an ID
must be updated.

## [problem_c](burr-tools/src/lib/problem.h#L69)
* vector\<[part_c](burr-tools/src/lib/problem.cpp#L48)\> parts;
* unsigned int result; (shape index of result shape)
* [assembler_c](burr-tools/src/lib/assembler.h#L63)* assm;
* vector\<[solution_c](burr-tools/src/lib/solution.h#L36)*\> solutions;

## [assembler_c](burr-tools/src/lib/assembler.h#L63)
Doesn't hold any data that makes it into XML except for an opaque value
representing the current assembler state, which is used to restore an
interrupted assembly.

## [solution_c](burr-tools/src/lib/solution.h#L36)
* [assembly_c](burr-tools/src/lib/assembly.h#L154)* assembly;
* [separation_c](burr-tools/src/lib/disassembly.h#L170)* tree;
* [separationInfo_c](burr-tools/src/lib/disassembly.h#L291)* treeInfo;

## [assembly_c](burr-tools/src/lib/assembly.h#L154)
* vector\<[placement_c](burr-tools/src/lib/assembly.h#L45)\> placements;
## [separation_c](burr-tools/src/lib/disassembly.h#L170)
TODO
## [separationInfo_c](burr-tools/src/lib/disassembly.h#L291)
TODO
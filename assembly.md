# Assembly

## Assemblers

Base assembler:
  * [assembler.cpp](burr-tools/src/lib/assembler.cpp)
  * [assembler.h](burr-tools/src/lib/assembler.h)

### Assembler 0
  * [assembler_0.cpp](burr-tools/src/lib/assembler_0.cpp)
  * [assembler_0.h](burr-tools/src/lib/assembler_0.h)

Cannot handle piece ranges or multiple instances of the same piece. Dancing
links Algorithm X (DLX) with modifications to allow for variable voxels and
hole counting.

[Description in assembler_0.h](burr-tools/src/lib/assembler_0.h#L36):

    It is more or less identical to Don Knuths idea. Some changes have been done
    though to provide for holes. This class can not handle ranges or multi-pieces.

    All involved pieces must be there exactly one time. But in that case it is a
    bit faster than assembler_1.

### Assembler 1
  * [assembler_1.cpp](burr-tools/src/lib/assembler_1.cpp)
  * [assembler_1.h](burr-tools/src/lib/assembler_1.h)

Handles multiple instances of a piece, and piece ranges.

[Description in assembler_1.h](burr-tools/src/lib/assembler_1.h#L37):

    This assembler is written with ideas from Wei-Hwa Huang. It can handle ranges for the piece
    numbers and thus also multiple instances of one piece.

    But for simple cases it is not really optimal.

## Piece Ranges

[Description in assembler_1_c::prepare](burr-tools/src/lib/assembler_1.cpp#L331):

    Multiple instances of the same piece is [handled] in a similar way [as
    optional voxels]. To prevent finding the same solution again and again with
    just the pieces swapping places we number the pieces and their possible
    placements and disallow that the position number of piece n is lower than
    the position number of piece n-1. This can be achieved by adding more
    constraint columns. There need to be one column for each

Although this comment is also in [assembler_0_c::prepare](burr-tools/src/lib/assembler_0.cpp#L402), I believe this is a mistake since only assembler_1 supports piece ranges.

## The Algorithm

The assembler creates an exact cover problem corresponding to the problem and
solves it using a dancing link data structure and Algorithm X. The following
explanations go into depth of the BurrTools code in particular and assume
prior knowledge on:

* The exact cover problem
* Reduction of a piece placement puzzle to exact cover
* Algorithm X for solving exact cover
* Dancing links data structure
* "optional" columns (these columns are left out of the header so they are never chosen)

This also focuses on Assembler 1. Assembler 0 is largely the same but simpler
(it seems like the code from Assembler 0 was copied to form the basis for
Assembler 1). Issues of symmetry are also not covered here (see
[Symmetry](symmetry.md)).

### Setting up the Matrix

`assembler_1_c::createMatrix()` is called to initialize the assembler, aka
initialize the cover problem matrix. Its call graph looks like this:

* [`createMatrix()`](burr-tools/src/lib/assembler_1.cpp#L657) - Error checking and calling `prepare()`.
  * [`prepare()`](burr-tools/src/lib/assembler_1.cpp#L341) - Actually constructing the initial matrix.
    * [`GenerateFirstRow()`](burr-tools/src/lib/assembler_1.cpp#L163) - Adds the dancing links header row.
    * [`AddVoxelNode()`](burr-tools/src/lib/assembler_1.cpp#L219) - Adds one node, corresponding to a 
    * [`AddRangeNode()`](burr-tools/src/lib/assembler_1.cpp#L245)
    * [`AddPieceNode()`](burr-tools/src/lib/assembler_1.cpp#L183)
    * [`checkForTransformedAssemblies()`](burr-tools/src/lib/assembler_1.cpp#L1009)
    * [`canPlace()`](burr-tools/src/lib/assembler_1.cpp#L296)
    * [`addToCache()`](burr-tools/src/lib/assembler_1.cpp#L283)

The matrix has the following columns (in order):
* **Part** (one for each)
* **Fixed Voxels** (one for each): These are the only columns required for a solution. TODO: How are the nodes from other columns removed from the header row?
* **Variable Voxels** (one for each)
* **Range Column** (a single column, only present if pieces have ranges): See [Range Column Optimization](#range-column-optimization)

Node fields: (See [assembler_1.h](burr-tools/src/lib/assembler_1.h#L61))
* `right` / `left` / `up` / `down` - Links to neighbor nodes in dancing links structure.
* `colCount` - Shared use: column number for normal nodes and count for column header nodes.
* `min` / `max` - Number of 1s allowed in rows picked for an exact cover solution.
* `weight` - TODO

Each property of nodes is a vector, with the same index of each vector
corresponding to properties of the same node. Nodes are indexed with 0 being
the master header, followed by nodes for each column's header, followed by
whatever nodes happen to be added next in the sparse matrix.

[`prepare()`](burr-tools/src/lib/assembler_1.cpp#L341)
* [L350](burr-tools/src/lib/assembler_1.cpp#L350): Generate header row by calling `GenerateFirstRow()`
* [L368](burr-tools/src/lib/assembler_1.cpp#L368): Set each column's min/max
  * Variable Voxels: set `min=0`
  * Range column: 
  * All other columns are left at the default `min=1, max=1`
* [L393-558](burr-tools/src/lib/assembler_1.cpp#L393) Symmetry reduction
* [L571](burr-tools/src/lib/assembler_1.cpp#L571): For part in parts:
  * [L577-578](burr-tools/src/lib/assembler_1.cpp#L577): Set min/max for that part's column
  * [L591](burr-tools/src/lib/assembler_1.cpp#L591): For each possible rotation of the piece:
    * [L602-605](burr-tools/src/lib/assembler_1.cpp#L602): For each place the rotated piece can be placed in solution shape:
      * [L607](burr-tools/src/lib/assembler_1.cpp#L607): Starts a new row in the matrix with a `1` in the piece's column. It does this by calling [AddPieceNode](burr-tools/src/lib/assembler_1.cpp#L183) to add a node. 
      * [L610-616](burr-tools/src/lib/assembler_1.cpp#L610): Set `1` for each voxel's column in the current row. It does this by  calling [AddVoxelNode](burr-tools/src/lib/assembler_1.cpp#L219) to add a node.
      * [L618-621](burr-tools/src/lib/assembler_1.cpp#L618): Set `1` for the range column if this is a ranged piece. It does this by calling [AddRangeNode](burr-tools/src/lib/assembler_1.cpp#L245) to add a node.

The last big for loop that adds the pieces is important, so let's summarize it
with pseudocode:
([assembler_1.cpp:570-638](burr-tools/src/lib/assembler_1.cpp#L570))

```cpp
for(each part) {
  min[part column] = problem.getPartMinimum(part)
  max[part column] = problem.getPartMaximum(part)
  for(rotatedPiece of all part rotations) {
    for(every (x, y, z) voxel in bounds) {
      if(canPlaceAt(rotatedPiece, x, y, z)) {
        AddPieceNode(...)
        for(voxel in placement) {
          AddVoxelNode(...)
        }
        if(part has a range) {
          AddRangeNode(...);
        }
      }
    }
  }
}
```

Aside from some topics like symmetry, that's all there is to the initial matrix! Note that the "optional" columns aren't actually removed from the dancing links header, their min is just set to `0`.

### Cover Solver

TODO

[`assemble()`](burr-tools/src/lib/assembler_1.cpp#L1807): 
[`iterative()`](burr-tools/src/lib/assembler_1.cpp#L1469): 


### Extracting Solutions

TODO

[L198](burr-tools/src/lib/assembler_1.cpp#L198): `piecePositions` tracks the
placements for each row so they can be retrieved later.

### Range Column Optimization

An extra column referred to as "rangeColumn" is added to the cover problem,
used as an optimization if any parts have a range. The idea is that there is a
minimum and maximum number of total ranged pieces that have to be used in a
solution, purely based on voxel counts of the shapes and solution of the
problem. When exploring solutions, this column prevents adding too many ranged
pieces if doing so wouldn't allow enough pieces with fixed ranges to be used.

* [assembler_1.cpp:715](burr-tools/src/lib/assembler_1.cpp#L715): Range is calculated
* [assembler_1.cpp:386](burr-tools/src/lib/assembler_1.cpp#L386): The column's range is set
* TODO: Where else is this column touched?

Explanatory comment from [assembler_1.cpp:705](burr-tools/src/lib/assembler_1.cpp#L705):

```cpp
  /* in some cases it is useful to count the number of blocks the range pieces contribute
   * this is calculated here
   *
   * the result will contain between res_filled-res_vari   and res_filled voxels
   * now we can subtract all voxels contributed by fixed placed pieces (with no range)
   * and get the range of voxels that must be occupied by the range pieces
   *
   * This additional check prevents adding too many pieces from the ranges and so making placement of
   * pieces that have to be placed impossible
   */
```

The code can get confusing at times around here. `res_filled` includes both filled and variable voxels, so it should really be named `res_total`. 
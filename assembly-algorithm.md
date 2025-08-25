# Assembly Algorithm

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


## Setting up the Matrix

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
* **Range Column** (a single column, only present if pieces have ranges): See [Range Column Optimization](#optimization-range-column)

Node fields: (See [assembler_1.h](burr-tools/src/lib/assembler_1.h#L61))
* `right` / `left` / `up` / `down` - Links to neighbor nodes in dancing links structure.
* `colCount` - Shared use:
  * Header nodes: sum of node `weight` for all nodes in this column
  * Non-Header nodes: column header node index
* `min` / `max` - Number of 1s allowed in rows picked for an exact cover solution.
* `weight`
  * Header nodes: sum of currently chosen row set's weights in this column.
  * Non-header nodes: How much this node contributes towards this column's min/max.

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


## Cover Solver

High-level functions
* [`assemble()`](burr-tools/src/lib/assembler_1.cpp#L1807): Top-level method for finding solutions. Mostly just calls `iterative()`.
* [`iterative()`](burr-tools/src/lib/assembler_1.cpp#L1469): Huge loop with the cover solving logic.
* [`solution()`](burr-tools/src/lib/assembler_1.cpp#L1054): Called when a solution is found.

Cover problem related methods:
* [`hiderows()`](burr-tools/src/lib/assembler_1.cpp#L1207)
* [`hiderow()`](burr-tools/src/lib/assembler_1.cpp#L1170)
* [`unhiderows()`](burr-tools/src/lib/assembler_1.cpp#L1229)
* [`unhiderow()`](burr-tools/src/lib/assembler_1.cpp#L1184)
* [`cover_column_only()`](burr-tools/src/lib/assembler_1.cpp#L1136)
* [`uncover_column_only()`](burr-tools/src/lib/assembler_1.cpp#L1141)
* [`cover_column_rows()`](burr-tools/src/lib/assembler_1.cpp#L1146)
* [`uncover_column_rows()`](burr-tools/src/lib/assembler_1.cpp#L1158)
* [`column_condition_fulfilled()`](burr-tools/src/lib/assembler_1.cpp#L1240)
* [`column_condition_fulfillable()`](burr-tools/src/lib/assembler_1.cpp#L1244)
* [`open_column_conditions_fulfillable()`](burr-tools/src/lib/assembler_1.cpp#L1084)
* [`find_best_unclosed_column()`](burr-tools/src/lib/assembler_1.cpp#L1111): Finds the best next column to pick. Uses `betterParams()` as a comparison function.
* [`betterParams()`](burr-tools/src/lib/assembler_1.cpp#L1096)

Most of the interesting logic is in `iterative()`. The whole method is one big
loop executing tasks in a task stack. Its general struture looks like this:

```cpp
// (initialization done on class init)
task_stack.push_back(0);
next_row_stack.push_back(0);

while (task_stack.size() > 0) {
  switch (task_stack.back()) {

    // Various cases remove tasks using:
    task_stack.pop_back();
    next_row_stack.pop_back();

    // Various cases add tasks using:
    task_stack.push_back(...);
    next_row_stack.push_back(...);

  }
}
```

Above `iterative()` is a recursive implementation, `rec()`, which is meant to
be easier to understand. It has the code corresponding to each case in
`iterative()` labeled. I'm not sure I can summarize `rec()` without rewriting
it, so here's a shot at a half-pseudocode cleanup. I've tried to remove
debug/optimization features, split parts into functions, and pair down on extra
comments/spaces so the code structure is more apparent.

```cpp
/**
 * If called with a header node, select a new column and recurse to process it.
 * Otherwise, this will process each row in the column, and recurse as needed.
 */
void assembler_1_c::rec(unsigned int node_index) {
  if (next_row_stack.back() < headerNodes) {  // Is given node in header?
    select_new_column()
    return;
  }

  // Get column of current node. (In non-header nodes, colCount references the column header node)
  unsigned int col = colCount[node_index];

  // Consider solutions with no rows from this column selected.
  if (column_condition_fulfilled(col)) {
    cover_column_rows(col);
    if (open_column_conditions_fulfillable())
      rec(0);
    uncover_column_rows(col);
  }

  // Add a marker for `unhiderows()` so it removes rows in the stack up only to this point.
  hidden_rows.push_back(0);

  for (row in col) {
    choose_row(col, row)
  }

  // reinsert all the rows that were removed over the course of the row by row inspection
  unhiderows();
}

/* Select a column to work with and recurse to process it. */
void select_new_column() {
  if (right[0] == 0) {  // Are there no more columns?
    solution();
    return;
  }

  int col = find_best_unclosed_column();
  if (col == -1) { return; }

  if (colCount[col] == 0) {
    if (column_condition_fulfilled(col)) {

      // Move to another column, since this one is fulfilled
      cover_column_only(col);
      rec(0);
      uncover_column_only(col);

    }
  } else {

    // Recurse to process rows in this column
    cover_column_only(col);
    rec(down[col]);
    uncover_column_only(col);

  }
}

void choose_row(unsigned int col, unsigned int row) {
  rows.push_back(row);

  // Subtract column weights for each `1` in this row
  weight[colCount[row]] += weight[row];
  for (unsigned int r = right[row]; r != row; r = right[r])
    weight[colCount[r]] += weight[r];

  // if there are unfulfillable columns we don't even need to check any further
  if (open_column_conditions_fulfillable()) {

    // remove useless rows (that are rows that have too much weight
    // in one of their nodes that would overflow the expected weight
    hiderows(row);

    if (open_column_conditions_fulfillable()) {
      continue_search_in_same_column(col, row)
    }

    unhiderows();
  }

  // Restore column weights
  for (unsigned int r = left[row]; r != row; r = left[r])
    weight[colCount[r]] -= weight[r];
  weight[colCount[row]] -= weight[row];

  rows.pop_back();

  // after we finished with this row, we will never use it again, so
  // remove it from the matrix
  hiderow(row);
  hidden_rows.push_back(row);
}

/**
 * Called after a row is chosen, but based on the column's
 * min/max/count we could choose more rows in the same column.
 */
void continue_search_in_same_column(unsigned int col, unsigned int row) {
  if (colCount[col] == 0) {

    // when there are no more rows in the current column
    // we can immediately start a new column
    // if the current column condition is really fulfilled
    if (column_condition_fulfilled(col))
      rec(0);

  } else {

    // we need to recurse, if there are rows left and the current
    // column condition is still fulfillable, we need to check
    // the current column again because this column is no longer open,
    // is was removed on selection
    if (column_condition_fulfillable(col)) {

      unsigned int newrow = row;

      // do gown until we hit a row that is still inside the matrix
      // this works because rows are hidden one by one and so the double link
      // to the row above or below is no longer intact, when the row is gone, the down
      // pointer still points to the row that is was below before the row was hidden, but
      // the pointer from the row below doesn't point up to us, so we do down until
      // the link down-up points back to us
      while ((down[newrow] >= headerNodes) && up[down[newrow]] != newrow) newrow = down[newrow];

      rec(newrow);
    }
  }
}
```

Got that? Good.

I gave up on understanding the iterative version without doing a substantial
cleanup on the code and running regression tests on my new version. You can
find my latest efforts here:

  https://github.com/mbrown1413/burr-tools/blob/master/src/lib/assembler_1.cpp

Notice `enum TaskType` near the top labels the tasks and gives a description of
what each does. This should make it somewhat clear how control flows through
each task. It's helpful to sometimes think of the tasks as "goto" statements,
with task 0 being a recursive call.

It's also worth a look at the recursive version, which is also cleaned up an
kept in sync with the iterative version.


## Extracting Solutions

TODO

[L198](burr-tools/src/lib/assembler_1.cpp#L198): `piecePositions` tracks the
placements for each row so they can be retrieved later.


## Optimization: Matrix Reduction

There is actually a step before `iterate()` is called which is purely an
optimization: `reduce()` is called to make some simplifications of the matrix
before we actually try to find a solution. The relavent methods not covered in
the explanation about the solver:

* [`reduce()`](burr-tools/src/lib/assembler_1.cpp#L859): 
* [`clumpify()`](burr-tools/src/lib/assembler_1.cpp#L766): 
* [`remove_column()`](burr-tools/src/lib/assembler_1.cpp#L750): 
* [`remove_row()`](burr-tools/src/lib/assembler_1.cpp#L842): 

TODO


## Optimization: Range Column

An extra column referred to as "rangeColumn" is added to the cover problem,
used as an optimization if any parts have a range. The idea is that there is a
minimum and maximum number of total voxels from ranged pieces that have to be
used in a solution, purely based on voxel counts of the shapes and solution of
the problem. When exploring solutions, this column prevents adding too many
ranged pieces if doing so wouldn't allow enough fixed pieces to be used.

* [assembler_1.cpp:715](burr-tools/src/lib/assembler_1.cpp#L715): Min/max is calculated
* [assembler_1.cpp:386](burr-tools/src/lib/assembler_1.cpp#L386): The range column's min/max is set
* [assembler_1.cpp:621](burr-tools/src/lib/assembler_1.cpp#L386): A node is added to range column when piece has a range. Its weight is the number of voxels in the piece.
  * See [AddRangeNode](burr-tools/src/lib/assembler_1.cpp#L386)

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

The code can get confusing at times around here. `res_filled` includes both
filled and variable voxels, so it should really be named `res_total`. 

The range calculation makes sense after you think about it for a bit. We take
the min/max contribution of voxels we need for the result, then subtract the
voxels from the fixed pieces:

    RangeMin = MAX(0, number of fixed voxels in result - number of voxels in fixed pieces)
    RangeMax =        number of total voxels in result - number of voxels in fixed pieces

Then the range column is added to the cover matrix with the following properties:
* Min / max are set to `RangeMin` / `RangeMax` as calculated above.
* A node is added for each row with a ranged piece.
* The weight of the node is the number of voxels in that piece.

Let me explain weights, as I think this is the only place it's used. An
extension to the cover problem is a column's min/max, which is how many nodes
are allowed in a column to be chosen for a valid solution. The weight is a
further extension to the cover problem, where instead of each node worth
just 1, it's worth a different weight.

Since each row corresponds to a placement of a piece, the range column
effectively counts the number of voxels used in ranged pieces, and limits it.
It won't actually change the result, but it could prevent a lot of exploration
of the search space that won't result in a solution.


## Optimization: Max Holes

Max holes is a user configurable value in a problem. To edit it, click "Detail"
under the "Puzzle" tab. If there are no ranged pieces, this value is determined
automatically, otherwise the user-specified value is used, with no limit if not
specified.

[BurrTools User Guide: Editing Problem Details](https://burrtools.sourceforge.net/gui-doc/EditingProblemDetails.html):

    Finally, this window also contains an entry field called Maximum Number of Holes (empty variable cubes). This value is used by the program, when piece ranges are used, in which case it is not possible for the program to determine how many holes there will be in the final solution. Because this missing information results in a huge slowdown as many more possibilities have to be tried, it is possible to use this field to specify the maximum number of holes allowed. If the number of holes should not be limited, the field should be left empty.

[BurrTools User Guide: Tips and Tricks](https://burrtools.sourceforge.net/gui-doc/TipsandTricks1.html):

    Keep this value as small as possible, because the more holes a puzzle contains the longer the solving will take. Normally the value is undefined, meaning the number of holes is not limited. So if you know the number of holes you want, or you want to limit them, or the solving takes too long, use this field. 

[assembler_1.h:70](burr-tools/src/lib/assembler_1.h#L70):
```cpp
  /* this vector contains all columns that are used for the hole
   * optimisation: up to "holes" instances of these columns might
   * be zero
   */
  std::vector<unsigned int> holeColumns;
  unsigned int holes;
```

* [assembler_1.cpp:375](burr-tools/src/lib/assembler_1.cpp#L375): Add variable voxel columns to a list of hole columns, `holeColumns`.
* [assembler_1.cpp:686](burr-tools/src/lib/assembler_1.cpp#L686): Set `holes` in the assembler (the maximum number of holes).
  * if min and max voxels of pieces used is the same (no ranged pieces):
    * holes = total result voxels - min voxels of pieces used
  * else if user has defined max holes manually:
    * Use user's max holes
  * else:
    * Disable max holes by setting `holes = 0xFFFFFF`
* [assembler_1.cpp:1505](burr-tools/src/lib/assembler_1.cpp#L1505): Check for max holes while solving.
  * Recursive version: [assembler_1.cpp:1266](burr-tools/src/lib/assembler_1.cpp#L1266)

The max holes check counts the number of columns in `holeColumns` which will
definitely have a `0` in it (that is, it's definitely a hole). If the count
exceeds `holes` (the max number of holes), it will backtrack. 

This is the condition used for considering a column to be a hole:

    colCount[col] == 0 && weight[col] == 0

* `colCount[col]` means there are no more unchosen rows with a `1` in that column.
* `weight[col]` means no rows with a `1` have already been chosen in that
  column. (Remember, a column's weight starts at `0` and is added to as we add
  rows to our solution row set.)

Unlike the Range Column optimization, the Max Holes optimization may actually
result in less solutions (if user-specified). It is usually culling solutions
that the user knows they don't want, or knows don't exist. At this time, I'm
not aware of any puzzles which take advantage of a user defined max holes (this
is saved in the "maxHoles" attribute of a problem's XML). I'm curious how much
the max holes optimization helps speed up the solve process when no ranged
pieces are used and the number of holes are exactly known.


## Progress

* [assembler_c::getFinished()](burr-tools/src/lib/assembler.h#L140)
* [assembler_1_c::finished_a()](burr-tools/src/lib/assembler_1.h#L94)


## Removing Duplicates

[Description in assembler_1_c::prepare](burr-tools/src/lib/assembler_1.cpp#L331):

    Multiple instances of the same piece is [handled] in a similar way [as
    optional voxels]. To prevent finding the same solution again and again with
    just the pieces swapping places we number the pieces and their possible
    placements and disallow that the position number of piece n is lower than
    the position number of piece n-1. This can be achieved by adding more
    constraint columns. There need to be one column for each

Although this comment is also in [assembler_0_c::prepare](burr-tools/src/lib/assembler_0.cpp#L402), I believe this is a mistake since only assembler_1 supports piece ranges.


## Open Questions
* The optimization in `iterative()` [here](burr-tools/src/lib/assembler_1.cpp#L1560) and `rec()` [here](burr-tools/src/lib/assembler_1.cpp#L1302) maybe doesn't do much. It's supposed to be a fast path if the column is empty, but `rec(0)` should be the same as `rec(down[col])` in that case. The only difference is that it skips if the column count is 0 and the condition is unfulfilled.
* In the if statement in `iterative()` [here](burr-tools/src/lib/assembler_1.cpp#L1621) and `rec()` [here](burr-tools/src/lib/assembler_1.cpp#L1359), it seems like if the condition is false there's no need to do the `cover_column_rows(col)` before and `uncover_column_rows(col)` after.
  * Not true! The act of doing `cover_column_rows(col)` may change the result of `open_column_conditions_fulfillable()`. I do get the feeling there's a slightly better way of doing this though.
* What is BurrTools doing when it says "optimize piece <#>"? What about other messages during solving?
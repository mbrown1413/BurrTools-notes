
# Symmetry

This will all be in the context of assembler_1 unless otherwise noted.

Symmetry flags:
* Keep Mirror Solutions
  * "Don't remove solutions that are mirrors of another solution"
  * [LFl_Check_Button* KeepMirrors](burr-tools/src/gui/mainwindow.cpp#L3425)
  * [Sets flag](burr-tools/src/gui/mainwindow.cpp#L997) `solveThread_c::PAR_KEEP_MIRROR` passed to [solveThread_c::solveThread_c()](burr-tools/src/lib/solvethread.cpp#L111)
* Keep Rotated Solutions
  * "Don't remove soltions that are rotations of other solutions"
  * [LFl_Check_Button* KeepRotations](burr-tools/src/gui/mainwindow.cpp#L3429)
  * [Sets flag](burr-tools/src/gui/mainwindow.cpp#L998) `solveThread_c::PAR_KEEP_ROTATIONS` passed to [solveThread_c::solveThread_c()](burr-tools/src/lib/solvethread.cpp#L111)
* Expnsv Rot Check
  * "Do expensive and thorough rotation check, eliminating translations and rotations not in symmetry of the result shape"
  * [LFl_Check_Button* CompleteRotations](burr-tools/src/gui/mainwindow.cpp#L3417)
  * [Sets flag](burr-tools/src/gui/mainwindow.cpp#L1002) `solveThread_c::PAR_COMPLETE_ROTATIONS` passed to [solveThread_c::solveThread_c()](burr-tools/src/lib/solvethread.cpp#L111)

Those flags are passed to the assembler:
  * [solveThread_c::run()](burr-tools/src/lib/solvethread.cpp#L47) passes flags to [assembler_1_c::createMatrix()](burr-tools/src/lib/assembler_1.cpp#L657) (or assembler_0 if it's in use)
    * Parameter names: `keepMirror`, `keepRotations`, `comp` (complete rotation check)

To summarize [createMatrix()](burr-tools/src/lib/assembler_1.cpp#L657)'s use of these flags:

```cpp
assembler_1_c::createMatrix(bool keepMirror, bool keepRotations, bool comp) {
  complete = comp;

  if (keepMirror) {
    if (avoidTransformedMirror)
      delete avoidTransformedMirror;
    avoidTransformedMirror = 0;
  }

  if (keepRotations)
    avoidTransformedAssemblies = false;
}
```

So we have the 3 instance variables of `assembler_1_c` affecting symmetry,
corresponding to the 3 flags from the UI:
* `complete` - Expnsv Rot Check
* `avoidTransformedMirror` - Keep Mirrored Solutions
* `avoidTransformedAssemblies` - Keep Rotated Solutions

`avoidTransformedMirror` is type `mirrorInfo_c*`, storing information about
which pieces are mirrors of one another, and is cleared if we want to keep
mirror solutions.

We'll explore each of these individually in more detail now.

## `avoidTransormedMirror` structure (Keep Mirror Solutions)
* [Definition](burr-tools/src/lib/assembler_1.h#L171): `mirrorInfo_c * avoidTransformedMirror;`
* [assembler_1_c::assembler_1_c()](burr-tools/src/lib/assembler_1.cpp#L268): Initialized to 0
* [assembler_1_c::createMatrix()](burr-tools/src/lib/assembler_1.cpp#L740): Cleared if `keepMirror`
* [assembler_1_c::checkForTransformedAssemblies()](burr-tools/src/lib/assembler_1.cpp#L1012): Set to parameter
  * Called from [assembler_1_c::prepare()](burr-tools/src/lib/assembler_1.cpp#L341) which we'll explore more later.
* [assembler_1_c::solution()](burr-tools/src/lib/assembler_1.cpp#L1060): Called when a solution is found. The solution is ignored if `avoidTransformedAssemblies && assembly->smallerRotationExists(problem, avoidTransformedPivot, avoidTransformedMirror, complete)`
  * We'll explore [assembly_c::smallerRotationExists()](burr-tools/src/lib/assembly.cpp#L630) more later.

## `avoidTransformedAssemblies` flag (Keep Rotated Solutions)
* [assembler_1_c::assembler_1_c()](burr-tools/src/lib/assembler_1.cpp#L268): Initialized to 0
* [assembler_1_c::createMatrix()](burr-tools/src/lib/assembler_1.cpp#L744): Set to `false` if `keepRotations`
* [assembler_1_c::checkForTransformedAssemblies()](burr-tools/src/lib/assembler_1.cpp#L1010): Set to `true`
  * Called from [assembler_1_c::prepare()](burr-tools/src/lib/assembler_1.cpp#L341) which we'll explore more later.
* [assembler_1_c::solution()](burr-tools/src/lib/assembler_1.cpp#L1060): Called when a solution is found. The solution is ignored if `avoidTransformedAssemblies && assembly->smallerRotationExists(problem, avoidTransformedPivot, avoidTransformedMirror, complete)`
  * We'll explore [assembly_c::smallerRotationExists()](burr-tools/src/lib/assembly.cpp#L630) more later.

## `complete` flag (Do expensive Rotation Check)
* [assembler_1_c::createMatrix()](burr-tools/src/lib/assembler_1.cpp#L661): Set to `comp` parameter (leads back to UI checkbox)
* [assembler_1_c::solution()](burr-tools/src/lib/assembler_1.cpp#L1060): Called when a solution is found. The solution is ignored if `avoidTransformedAssemblies && assembly->smallerRotationExists(problem, avoidTransformedPivot, avoidTransformedMirror, complete)`
  * We'll explore [assembly_c::smallerRotationExists()](burr-tools/src/lib/assembly.cpp#L630) more later.

## [assembler_1_c::prepare](burr-tools/src/lib/assembler_1.cpp#L341)
Called in [assembler_1_c::createMatrix()](burr-tools/src/lib/assembler_1.cpp#L728) before it clears `avoidTransformedMirror` if `keepMirror == true`.

`prepare()` does a number of important things:
* Initializes the Dancing Links matrix structure
* Finds symmetry breaking piece
* Initializes mirror symmetry 

At a high level, `prepare()` does the following:

```cpp
int assembler_1_c::prepare(bool hasRange, unsigned int rangeMin, unsigned int rangeMax) {
  if (result piece has some self-symmetry) {
    (1) Find symmetry breaker shape
    if (result shape has mirror self-symmetry) {
      (2) Initialize mirror structure if needed
    }
  }
  (3) Create cover problem from piece placements
}
```

1. Find symmetry breaker shape
This runs even if all checkboxes in UI indicate that you want to keep all
symmetries. It will also find the best piece to break symmetry, _then_ ignore
it that piece has a range. It'd make more sense to exclude pieces with a range
in the first place.
```cpp
symBreakerShape = i with the lowest maximum count and lowest symmetry in common with the result shape
// Lowest: problem.getPartMaximum(i) < problem.getPartMaximum(symBreakerShape)
// Lowest: sym->countSymmetryIntersection(resultSym, problem.getPartShape(i)->selfSymmetries());

if (
  we have remaining symmetries not reduced by symBreakerShape ||
  problem.getPartMaximum(symBreakerShape) > 1 ||
  any piece has a range
) {

  // we can not use the symmetry breaker shape, if there is more than one piece
  // of this shape in the problem
  if (
    any piece has a range ||
    problem.getPartMaximum(symBreakerShape) > 1
  ) {
    symBreakerShape = null
  }

  checkForTransformedAssemblies(symBreakerPiece, 0);
}
```

2. Initialize mirror structure if needed
This goes through every piece without mirror self-symmetry and checks if it has
mirror symmetry with another piece. The comments spell out the possible cases:
    * (1) all pieces contain mirror symmetries -> check mirrors, but no pairs
    * (2) as least one piece has no mirror symmetries
        * (2a) all pieces with no mirror symmetries have a mirror partner -> check mirrors, find pairs
        * (2b) at least one piece with no mirror symmetries has no partner -> no mirror check

If piece ranges are used, the mirror structure is initialized regardless of the previous cases, since we don't know which pieces will be used. However, we may have exited the loop where we find mirror pairs early. I think this is a bug since we're using the mirror structure when it's not complete, although I think the worst that can happen is we have some extra mirror solutions.

```cpp
bool mirrorCheck = true;
for (part i of parts) {
  if (we already found a mirror) { continue }

  if (part has no mirror self-symmetry) {
    /* this shape is not self mirroring, so we need to look out
      * for a shape that is the mirror of this shape
      */
    bool found = false;

    for (part j of parts with i>j) {
      if (we already found a mirror for part j) { continue }
      if (there is a mirror transform from i to j) {
        // found a mirror shape
        set mirror of i to j
        found = true;
        break;
      }
    }

    // when we could not find a mirror transformation for the non mirrorable piece
    // we can stop and we don't need to make mirror checks
    if (!found) {
      mirrorCheck = false;
      break;
    }
  }
}

if (mirrorCheck || pieceRanges) {
  /* all the shapes are either self mirroring or have a mirror pair
    * so we create the mirror structure and we do the mirror check
    * we also need to that when ranges are used because the final solution
    * might use only mirrorable pieces and then we need this information
    */
  mirrorInfo_c * mir = new mirrorInfo_c();
  initialize mir with mirror info discovered above
  checkForTransformedAssemblies(symBreakerPiece, mir);
}
```

3. Create cover problem from piece placements
This does the expected enumeration of all pieces and their placements. For each
placement, it checks a cache first and skips it if that placement has already
been processed. When it comes across the symmetry breaker piece, it adds to the
cache any rotations it has in common with the result piece. This prevents those
rotations from being processed in further loop iterations.
```cpp
for (each piece) {
  for (rotatedPiece of all piece rotations) {

      for(every (x, y, z) voxel in bounds) {
        if (canPlace(rotatedPiece, x, y, z)) {
          AddPieceNode(...)
          for(voxel in placement) {
            AddVoxelNode(...)
          }
          if (range) {
            AddRangeNode(...);
          }
        }
      }

      if (this is the symmetry breaker shape) {
        Mark all transforms that symmetry breaker shape has in common with
        result as complete, so we don't add them to the matrix next iteration
      }

  }
}
```

### [assembly_c::smallerRotationExists()](burr-tools/src/lib/assembly.cpp#L630)

This function performs rotation and mirror checks to determine if we should drop the given 

```cpp
bool assembly_c::smallerRotationExists(
  const problem_c & puz,
  unsigned int pivot,
  const mirrorInfo_c * mir,
  bool complete
) const {

  /* we only need to check for mirrored transformations, if mirrorInfo is given
   * if not we assume that the piece set contains at least one piece that has no
   * mirror symmetries and no mirror pair
   */
  unsigned int endTrans = mir ? sym->getNumTransformationsMirror() : sym->getNumTransformations();

  if (complete)
  {
    (1) Expensive rotation check
  }
  else
  {
    (2) Normal rotation check
  }

  return false;
}
```

1. Expensive rotation check
```cpp
for (unsigned char t = 0; t < endTrans; t++)
{
  assembly_c tmp(this);

  // if we can not create the transformation we can continue to
  // the next orientation
  if (!tmp.transform(t, puz, mir))
  {
    continue;
  }

  // check, if the found transformation is valid, if not
  // we can continue to the next
  if ((t >= sym->getNumTransformations()) &&
      (tmp.containsMirroredPieces() || !tmp.validSolution(puz)))
  {
    continue;
  }

  // ok, we have found the rotated assembly and it exists, we now need to
  // find out if the whole arrangement can be shifted to the lower positions
  // if that is the case we don't keep the stuff

  // now we create a voxel space of the given assembly shape/ and shift that one around
  voxel_c * assm = tmp.createSpace(puz);
  const voxel_c * res = getResultShape(puz);

  for (int x = (int)res->boundX1()-(int)assm->boundX1(); (int)assm->boundX2()+x <= (int)res->boundX2(); x++)
    for (int y = (int)res->boundY1()-(int)assm->boundY1(); (int)assm->boundY2()+y <= (int)res->boundY2(); y++)
      for (int z = (int)res->boundZ1()-(int)assm->boundZ1(); (int)assm->boundZ2()+z <= (int)res->boundZ2(); z++)
      {
        if (assm->onGrid(x, y, z))
        {
          bool fits = true;

          for (int pz = (int)assm->boundZ1(); pz <= (int)assm->boundZ2(); pz++)
            for (int py = (int)assm->boundY1(); py <= (int)assm->boundY2(); py++)
              for (int px = (int)assm->boundX1(); px <= (int)assm->boundX2(); px++)
              {
                if (
                    // the piece can not be place if the result is empty and the piece is filled at a given voxel
                    ((assm->getState(px, py, pz) == voxel_c::VX_FILLED) &&
                      (res->getState2(x+px, y+py, z+pz) == voxel_c::VX_EMPTY)) ||

                    // the placement is also invalid, when not all "must be filled" voxels are filled
                    ((assm->getState(px, py, pz) == voxel_c::VX_EMPTY) &&
                      (res->getState2(x+px, y+py, z+pz) == voxel_c::VX_FILLED)) ||


                    // the piece can also not be placed when the colour constraints don't fit
                    !puz.placementAllowed(assm->getColor(px, py, pz), res->getColor2(x+px, y+py, z+pz))

                    )
                  fits = false;
              }

          if (fits)
          {
            // well the assembly fits at the current position, so let us see...
            for (unsigned int i = 0; i < tmp.placements.size(); i++)
            {
              tmp.placements[i].xpos += x;
              tmp.placements[i].ypos += y;
              tmp.placements[i].zpos += z;
            }

            if (tmp.compare(*this, pivot)) {
              delete assm;
              return true;
            }

            for (unsigned int i = 0; i < tmp.placements.size(); i++)
            {
              tmp.placements[i].xpos -= x;
              tmp.placements[i].ypos -= y;
              tmp.placements[i].zpos -= z;
            }
          }
        }
      }

  delete assm;
}
```

2. Normal rotation check
```cpp
for (unsigned char t = 0; t < endTrans; t++)
{
  symmetries_t s = getResultShape(puz)->selfSymmetries();

  if (sym->symmetrieContainsTransformation(s, t))
  {
    assembly_c tmp(this);
    bt_assert2(tmp.transform(t, puz, mir));

    // if the assembly orientation requires mirrored pieces
    // it is invalid, that should be the case for most assemblies
    // when checking for mirrored
    //
    // FIXME: we should check, if we can exchange 2 shapes that are
    // mirrors of one another to see, if we can remove the mirror
    // problem
    //
    // we also need to make sure that the new found assembly uses the right amount
    // of pieces from each shape. Because it is possible that the mirror
    // shape is allowed with a different interval it is possible that
    // after mirroring the number of instances for some shapes is wrong
    if ((t >= sym->getNumTransformations()) &&
        (tmp.containsMirroredPieces() || !tmp.validSolution(puz)))
    {
      continue;
    }

    if (tmp.compare(*this, pivot)) {
      return true;
    }
  }
}
```
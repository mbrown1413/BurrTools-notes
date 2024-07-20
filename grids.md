# Grids

<table>
  <thead>
    <tr>
      <th>#</th>
      <th>gridType enum</th>
      <th>Name in UI</th>
      <th>Description</th>
    </tr>
  </thead>
  <tr>
    <td>0</td>
    <td>GT_BRICKS</td>
    <td>Brick</td>
    <td>cubes</td>
  </tr>
  <tr>
    <td>1</td>
    <td>GT_TRIANGULAR_PRISM</td>
    <td>Triangular Prism</td>
    <td>triangles stacked in Z-direction</td>
  </tr>
  <tr>
    <td>2</td>
    <td>GT_SPHERES</td>
    <td>Spheres</td>
    <td>tightly packed spheres</td>
  </tr>
  <tr>
    <td>3</td>
    <td>GT_RHOMBIC</td>
    <td>Rhombic Tetrahedra</td>
    <td>complicated cut cube to build rhombic dodecahedra</td>
  </tr>
  <tr>
    <td>4</td>
    <td>GT_TETRA_OCTA</td>
    <td>Tetrahedra-Octahera</td>
    <td>spacegrid for with tetrahedron and octrahera, also a cut cube</td>
  </tr>
</table>

See:
  * [gridtype.h](burr-tools/src/lib/gridtype.h)
    * [gridType_c](burr-tools/src/lib/gridtype.h#L46)
  * [guigridtype.cpp](burr-tools/src/gui/guigridtype.cpp)

The `gridType_c` class defines the current grid. It is mostly a pass-through
which multiplexes between different classes actually handling the logic for
voxels, symmetries, etc.

<table>
  <thead>
    <tr>
      <th>#</th>
      <th>gridType enum</th>
      <th>Voxel Class</th>
      <th>Symmetry Class</th>
      <th>Movement Cache</th>
    </tr>
  </thead>
  <tr>
    <td>0</td>
    <td>GT_BRICKS</td>
    <td>
      0<br>
      <a href="burr-tools/src/lib/voxel_0.h">voxel_0.h</a><br>
      <a href="burr-tools/src/lib/voxel_0.cpp">voxel_0.cpp</a>
    </td>
    <td>
      0<br>
      <a href="burr-tools/src/lib/symmetries_0.h">symmetries_0.h</a><br>
      <a href="burr-tools/src/lib/symmetries_0.cpp">symmetries_0.cpp</a>
    </td>
    <td>
      0<br>
      <a href="burr-tools/src/lib/movementcache_0.h">movementcache_0.h</a><br>
      <a href="burr-tools/src/lib/movementcache_0.cpp">movementcache_0.cpp</a>
    </td>
  </tr>
  <tr>
    <td>1</td>
    <td>GT_TRIANGULAR_PRISM</td>
    <td>
      1<br>
      <a href="burr-tools/src/lib/voxel_1.h">voxel_1.h</a><br>
      <a href="burr-tools/src/lib/voxel_1.cpp">voxel_1.cpp</a>
    </td>
    <td>
      1<br>
      <a href="burr-tools/src/lib/symmetries_1.h">symmetries_1.h</a><br>
      <a href="burr-tools/src/lib/symmetries_1.cpp">symmetries_1.cpp</a>
    </td>
    <td>
      1<br>
      <a href="burr-tools/src/lib/movementcache_1.h">movementcache_1.h</a><br>
      <a href="burr-tools/src/lib/movementcache_1.cpp">movementcache_1.cpp</a>
    </td>
  </tr>
  <tr>
    <td>2</td>
    <td>GT_SPHERES</td>
    <td>
      2<br>
      <a href="burr-tools/src/lib/voxel_2.h">voxel_2.h</a><br>
      <a href="burr-tools/src/lib/voxel_2.cpp">voxel_2.cpp</a>
    </td>
    <td>
      2<br>
      <a href="burr-tools/src/lib/symmetries_2.h">symmetries_2.h</a><br>
      <a href="burr-tools/src/lib/symmetries_2.cpp">symmetries_2.cpp</a>
    </td>
    <td>(none)</td>
  </tr>
  </tr>
  <tr>
    <td>3</td>
    <td>GT_RHOMBIC</td>
    <td>
      3<br>
      <a href="burr-tools/src/lib/voxel_3.h">voxel_3.h</a><br>
      <a href="burr-tools/src/lib/voxel_3.cpp">voxel_3.cpp</a>
    </td>
    <td>
      0<br>
      <a href="burr-tools/src/lib/symmetries_0.h">symmetries_0.h</a><br>
      <a href="burr-tools/src/lib/symmetries_0.cpp">symmetries_0.cpp</a>
    </td>
    <td>(none)</td>
  </tr>
  <tr>
    <td>4</td>
    <td>GT_TETRA_OCTA</td>
    <td>
      4<br>
      <a href="burr-tools/src/lib/voxel_4.h">voxel_4.h</a><br>
      <a href="burr-tools/src/lib/voxel_4.cpp">voxel_4.cpp</a>
    </td>
    <td>
      0<br>
      <a href="burr-tools/src/lib/symmetries_0.h">symmetries_0.h</a><br>
      <a href="burr-tools/src/lib/symmetries_0.cpp">symmetries_0.cpp</a>
    </td>
    <td>(none)</td>
  </tr>
</table>


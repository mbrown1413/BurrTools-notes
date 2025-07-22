# Assemblers

Base assembler:
  * [assembler.cpp](burr-tools/src/lib/assembler.cpp)
  * [assembler.h](burr-tools/src/lib/assembler.h)

## Assembler 0
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

## Assembler 1
  * [assembler_1.cpp](burr-tools/src/lib/assembler_1.cpp)
  * [assembler_1.h](burr-tools/src/lib/assembler_1.h)

Handles multiple instances of a piece, and piece ranges.

[Description in assembler_1.h](burr-tools/src/lib/assembler_1.h#L37):

    This assembler is written with ideas from Wei-Hwa Huang. It can handle ranges for the piece
    numbers and thus also multiple instances of one piece.

    But for simple cases it is not really optimal.
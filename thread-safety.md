# Thread Safety

There are two threads we're concerned about when we're talking about thread
safety:

* Main thread: Handles the UI and most tasks
* Solve thread: Runs the solver

The most obvious example of an issue is the main thread removing a shape that
the assembler was using, which will crash the solve thread. Editing a piece
could also cause the solver to give malformed solutions, or yes, even crash.
There are also data structure thread safety issues, as basic C++ data
structures like vectors are not thread safe.

To prevent these issues, many UI actions are disabled while the assembly solver
is running. Here's a summary of the callbacks in `mainWindow_c` and how they
behave when the solve thread is running (I've bolded known or potential issues):

| Callback                       | Use while solving | Justification |
| ------------------------------ | ----------------- | ------------- |
| cb_AddColor()                  | Enabled | New colors won't affect the solver, as they are only added at the end
| cb_RemoveColor()               | Disabled in `updateInterface()`
| cb_ChangeColor()               | Enabled | This just changes metadata, the actual color, not anything that the assembler actually uses.
| cb_NewShape()                  | Enabled | New shapes won't affect the solver, as they are only added at the end.
| cb_DeleteShape()               | Disabled in `updateInterface()`
| cb_CopyShape()                 | Enabled | Copied shapes won't affect the solver, as they are only added at the end.
| cb_NameShape()                 | Enabled | Names are just metadata and they aren't used by the solver.
| cb_WeightChange()              | Enabled | **Does this cause issues in disassembler?** I assume the worst this could cause is inconsistent weighting of pieces, but I'm not sure.
| cb_TaskSelectionTab()          | Enabled | UI only
| cb_TransformPiece()            | Disabled in `updateInterface()` if piece is being used
| cb_EditSym()                   | Enabled | UI only
| cb_EditChoice()                | Enabled | UI only
| cb_EditMode()                  | Enabled | UI only
| cb_PcSel()                     | Enabled | UI only
| cb_SolProbSel()                | Enabled | UI only
| cb_ColSel()                    | Enabled | UI only
| cb_ProbSel()                   | Enabled | UI only
| cb_pieceEdit()                 | Disabled in `updateInterface()`
| cb_NewProblem()                | Enabled | New problems won't affect existing problems
| cb_DeleteProblem()             | Disabled in `updateInterface()`
| cb_CopyProblem()               | Enabled | New problems won't affect existing problems
| cb_RenameProblem()             | Enabled | Problem names are just metadata, not used by the solver
| cb_ProblemExchange()           | Disabled in `updateInterface()`
| cb_ShapeExchange()             | Disabled in `updateInterface()`
| cb_ProbShapeExchange()         | Disabled in `updateInterface()`
| cb_ColorAssSel()               | Enabled | UI only
| cb_ColorConstrSel()            | Disabled in `updateInterface()` if problem is being solved
| cb_ShapeToResult()             | Disabled in `updateInterface()` if problem is being solved
| cb_SelectProblemShape()        | Enabled | UI only
| cb_PiecesClicked()             | Enabled | UI only
| cb_AddShapeToProblem()         | Disabled in `updateInterface()` if problem is being solved
| cb_AddAllShapesToProblem()     | Disabled in `updateInterface()` if problem is being solved
| cb_RemoveShapeFromProblem()    | Disabled in `updateInterface()` if problem is being solved
| cb_SetShapeMinimumToZero()     | Disabled in `updateInterface()` if problem is being solved
| cb_RemoveAllShapesFromProblem()| Disabled in `updateInterface()` if problem is being solved
| cb_ShapeGroup()                | Disabled in `updateInterface()` if problem is being solved
| cb_BtnPlacementBrowser()       | Disabled in `updateInterface()`
| cb_BtnMovementBrowser()        | Disabled in `updateInterface()`
| cb_BtnAssemblerStep()          | Disabled in `updateInterface()`
| cb_AllowColor()                | Disabled in `updateInterface()` if problem is being solved
| cb_DisallowColor()             | Disabled in `updateInterface()` if problem is being solved
| cb_CCSort()                    | Enabled | UI only
| cb_BtnPrepare()                | Disabled in `updateInterface()`
| cb_BtnStart()                  | Disabled in `updateInterface()`
| cb_BtnCont()                   | Disabled in `updateInterface()`
| cb_BtnStop()                   | Disabled in `updateInterface()` if problem is not being solved
| cb_SolutionSel()               | Enabled | UI only
| cb_SolutionAnim()              | Enabled | UI only
| cb_SortSolutions()             | Disabled in `updateInterface()` if problem is being solved
| cb_DeleteSolutions()           | Disabled in `updateInterface()` if problem is being solved
| cb_DeleteDisasm()              | Disabled in `updateInterface()` if problem is being solved
| cb_DeleteAllDisasm()           | Disabled in `updateInterface()` if problem is being solved
| cb_AddDisasm()                 | Disabled in `updateInterface()` if problem is being solved
| cb_AddAllDisasm()              | Disabled in `updateInterface()` if problem is being solved
| cb_PcVis()                     | Enabled | UI only
| cb_Status()                    | Enabled | UI only
| cb_3dClick()                   | Enabled | **Causes malformed solutions and sometimes segfault**
| cb_New()                       | Disabled via dialog
| cb_Load()                      | Disabled via dialog
| cb_Load_Ps3d()                 | Disabled via dialog
| cb_Save()                      | Disabled via dialog
| cb_Convert()                   | Not disabled | **Causes a segfault**
| cb_AssembliesToShapes()        | Enabled | Only adds shapes. **This reads from solutions, so should be disabled?**
| cb_SaveAs()                    | Disabled via dialog
| hide()                         | Enabled | **This quits the main thread successfully but leaves the solve thread running until it finishes.**
| cb_Config()                    | Enabled | UI only
| cb_Coment()                    | Enabled | Doesn't modify anything in puzzle solver uses
| cb_ImageExportVector()         | Enabled | Only reads shapes
| cb_ImageExport()               | Enabled | **This reads from solutions, so should be disabled?**
| cb_STLExport()                 | Enabled | Only reads shapes
| cb_StatusWindow()              | Enabled | **Causes segfault when shapes deleted**
| cb_Toggle3D()                  | Enabled | UI only
| cb_Help()                      | Enabled | UI only
| cb_About()                     | Enabled | UI only

## Unanswered questions
* I think when the user clicks "Stop" on the solver, the UI enables editing again without waiting for the thread to actually stop?
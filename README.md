# Project Proposal

## Project Title:
Sudoku Game in MASM

## Project Members:
- Raghib Rizwan Rabani (23K-0012)
- Syed Fahad Faheem Shah (23K-0062)
- Talha Rusman (23K-0065)

## Course Instructor:
Professor Ubaidullah

---

## Introduction:
This project aims to develop a fully functional Sudoku game using MASM (Microsoft Macro Assembler). The game will provide an interactive interface for users to solve a 9x9 Sudoku puzzle, with essential features such as number input, validation, and a simple graphical representation using ASCII characters. The project will demonstrate our understanding of assembly language programming, memory management, and low-level input/output operations.

## Objectives:
- **Develop a Sudoku Grid** – Display a formatted 9x9 Sudoku board using ASCII characters.
- **User Input Handling** – Allow users to navigate the grid and enter numbers (1-9) into empty cells.
- **Validation Mechanism** – Ensure the entered number adheres to Sudoku rules (unique in row, column, and 3x3 sub-grid).
- **Predefined Puzzle Generation** – Load a partially completed Sudoku board at the start of the game.
- **Error Handling** – Notify users of invalid inputs and prevent rule-breaking entries.
- **Win Condition** – Verify when the puzzle is solved correctly and display a success message.
- **Basic UI Elements** – Enhance the user experience with a structured interface using ASCII characters.

## Project Plan & Implementation Steps:

### 1. Grid Representation:
- Implement a 2D array to store the Sudoku board.
- Display the grid using ASCII characters for a structured view.
- Use different symbols or colors (if supported) to distinguish pre-filled and user-filled numbers.

### 2. User Interaction:
- Use keyboard inputs to allow navigation across the grid.
- Enable users to select a cell and enter a number.
- Implement a mechanism to erase/edit user-entered numbers.

### 3. Validation Mechanism:
- Implement row, column, and 3x3 sub-grid validation using loops.
- Prevent users from altering predefined numbers in the grid.
- Provide real-time feedback when an invalid number is entered.

### 4. Game Logic:
- Implement a function to check if the board is completely filled.
- Verify whether the solution satisfies Sudoku rules.
- Include a hint system that provides possible numbers for a selected cell (optional).

### 5. Winning Condition:
- Check the completion status of the grid.
- Compare the final state with the correct solution.
- Display a congratulatory message upon successful completion.

### 6. Additional Features (Optional):
- **Timer Mechanism:** Track the time taken to solve the puzzle.
- **Difficulty Levels:** Implement different levels (Easy, Medium, Hard) with varying numbers of pre-filled cells.
- **Save/Load Feature:** Allow users to save progress and resume later.

## Tools & Technologies Used:
- MASM (Microsoft Macro Assembler) for assembly language coding.
- Visual Studio or any emulator to run MASM programs.

## Conclusion:
By implementing this project, we aim to create a functional and interactive Sudoku game in MASM that showcases our proficiency in assembly language programming. The project will cover essential low-level programming concepts such as memory management, input handling, and logic validation while offering an engaging user experience.

```asm
include Irvine32.inc

.data
    ; Sudoku board (0 = empty) using DWORD
    board DWORD 81 DUP(0)  ; 9x9 Sudoku board
    row DWORD ?
    col DWORD ?
    num DWORD ?

    ; Strings for display
    titleStr BYTE "MASM Sudoku", 0
    promptStr BYTE "Enter row, column, and value (1-9) or 0 to exit: ", 0
    promptRowStr BYTE "Enter row (1-9): ", 0
    promptColStr BYTE "Enter column (1-9): ", 0
    promptValStr BYTE "Enter value (1-9 or 0 to clear): ", 0
    invalidStr BYTE "Invalid input! Try again.", 0
    winStr BYTE "Congratulations! You solved the Sudoku!", 0
    errorStr BYTE "Invalid move! Try again.", 0
    boardTop BYTE "  +-------+-------+-------+", 0
    divider BYTE "  +-------+-------+-------+", 0
    spaceStr BYTE " ", 0
    barStr BYTE "|", 0
    newline BYTE 0Dh, 0Ah, 0
    editable DWORD 81 DUP(1)  ; 1 means editable, 0 means not editable
    
    ; Input variables
    inputBuffer DWORD 16 DUP(?)
    rownum SDWORD ?
    colnum SDWORD ?
    valnum SDWORD ?

.code
main PROC
    call Randomize       ; Initialize random seed

    ; Initialize board to zero
    mov esi, 0
    mov ecx, 81
initBoard:
    mov board[esi*4], 0
    mov editable[esi*4], 1  ; Mark all cells as editable initially
    inc esi
    loop initBoard

    ; Set up the puzzle
    call InitializeBoard  ; Now directly load the predefined board
    call removeNumbers    ; Remove some numbers to create puzzle
    
    ; Mark cells with initial values as non-editable
    mov esi, 0
    mov ecx, 81
markEditable:
    cmp board[esi*4], 0
    je cellEmpty
    mov editable[esi*4], 0  ; Mark non-empty cells as not editable
cellEmpty:
    inc esi
    loop markEditable
    
    call DisplayBoard     ; Display the board

GameLoop:
    ; Prompt for row
    mov edx, OFFSET promptRowStr
    call WriteString
    call ReadInt
    mov rownum, eax
    
    ; Check for exit condition
    cmp rownum, 0
    je ExitGame
    
    call ParseRowInput
    cmp eax, 0
    je MoveError  ; Handle invalid row input

    ; Prompt for column
    mov edx, OFFSET promptColStr
    call WriteString
    call ReadInt
    mov colnum, eax
    call ParseColInput
    cmp eax, 0
    je MoveError  ; Handle invalid column input

    ; Prompt for value
    mov edx, OFFSET promptValStr
    call WriteString
    call ReadInt
    mov valnum, eax
    call ParseValInput
    cmp eax, 0
    je MoveError  ; Handle invalid value input

    ; Validate move
    call ValidateMove
    cmp eax, 0
    je MoveError

    ; Update board
    call UpdateBoard

    ; Display updated board
    call DisplayBoard

    ; Check if ALL cells are filled
    call CheckSolved
    cmp eax, 1
    jne GameLoop  ; If not all filled, keep playing

    ; Only when all cells are filled, verify the solution
    call VerifySolution
    cmp eax, 1
    jne GameLoop  ; If invalid, keep playing

    ; Show win message
    mov edx, OFFSET winStr
    call WriteString
    call Crlf
    jmp ExitGame

MoveError:
    mov edx, OFFSET errorStr
    call WriteString
    call Crlf
    jmp GameLoop  ; Return to the start of the game loop

ExitGame:
    exit
main ENDP

InitializeBard proc
    call fillDiagonal
    call fillRemaining
    ret

InitializeBard endp


; Fill diagonal 3x3 boxes (not used in current implementation)
fillDiagonal PROC
    mov ecx, 0
boxLoop:
    call fillBox
    add ecx, 3
    cmp ecx, 9
    jl boxLoop
    ret
fillDiagonal ENDP

; Fill a 3x3 box (not used in current implementation)
fillBox PROC
    push ecx
    mov row, ecx
    mov col, ecx
    mov edi, ecx  ; Save original column start
    mov ecx, 0

fillLoop:
    call getRandomNum
    mov eax, num
    mov esi, row
    imul esi, 9
    add esi, col

    ; Ensure valid index before writing
    cmp esi, 81
    jge fillBoxEnd

    mov board[esi*4], eax  ; Write number
    mov editable[esi*4], 0 ; Mark as not editable

    inc col
    inc ecx
    cmp ecx, 3
    jl fillLoop

    ; Reset column and move to next row
    mov col, edi
    inc row
    add edi, 3
    cmp row, edi
    jl fillLoop

fillBoxEnd:
    pop ecx
    ret
fillBox ENDP

; Get random number 1-9
getRandomNum PROC
    mov eax, 9
    call RandomRange
    inc eax
    mov num, eax
    ret
getRandomNum ENDP

; Fill remaining cells
fillRemaining PROC
    call InitializeBoard
    ret
    mov row, 0
rowLoop:
    mov col, 0
colLoop:
    mov esi, row
    imul esi, 9
    add esi, col

    ; Check if index is valid
    cmp esi, 81
    jge skip

    cmp board[esi*4], 0
    jne skip
    mov num, 1

numLoop:
    push row
    push col
    push num
    call isSafe
    cmp eax, 1
    jne nextNum

    mov esi, row
    imul esi, 9
    add esi, col
    mov eax, num
    mov board[esi*4], eax

    call fillRemaining
    cmp eax, 1
    je skip

    mov esi, row
    imul esi, 9
    add esi, col
    mov board[esi*4], 0

nextNum:
    inc num
    cmp num, 10
    jle numLoop
    mov eax, 0
    ret

skip:
    inc col
    cmp col, 9
    jl colLoop
    inc row
    cmp row, 9
    jl rowLoop
    mov eax, 1
    ret
fillRemaining ENDP

; Check if number is safe
isSafe PROC
    push ebp
    mov ebp, esp
    push ebx
    push esi
    push edi

    mov ebx, [ebp+8]   ; num
    mov edx, [ebp+12]  ; col
    mov eax, [ebp+16]  ; row

    mov ecx, 0
rowCheck:
    mov esi, eax
    imul esi, 9
    add esi, ecx
    cmp board[esi*4], ebx
    je notSafe
    inc ecx
    cmp ecx, 9
    jl rowCheck

    mov ecx, 0
colCheck:
    mov esi, ecx
    imul esi, 9
    add esi, edx
    cmp board[esi*4], ebx
    je notSafe
    inc ecx
    cmp ecx, 9
    jl colCheck

    mov esi, eax
    mov edi, edx
    and esi, -3
    and edi, -3
    mov ecx, 0
boxCheckRow:
    cmp ecx, 3
    je boxSafe
    mov edx, 0
boxCheckCol:
    cmp edx, 3
    je nextBoxRow
    mov eax, esi
    add eax, ecx
    imul eax, 9
    add eax, edi
    add eax, edx
    cmp board[eax*4], ebx
    je notSafe
    inc edx
    jmp boxCheckCol

nextBoxRow:
    inc ecx
    jmp boxCheckRow

boxSafe:
    mov eax, 1
    jmp done
notSafe:
    mov eax, 0
done:
    pop edi
    pop esi
    pop ebx
    pop ebp
    ret 12
isSafe ENDP

; Initialize the board with a valid solution
InitializeBoard proc
    ; Row 1
    mov board[0*4], 5
    mov board[1*4], 3
    mov board[2*4], 4
    mov board[3*4], 6
    mov board[4*4], 7
    mov board[5*4], 8
    mov board[6*4], 9
    mov board[7*4], 1
    mov board[8*4], 2
    
    ; Row 2
    mov board[9*4], 6
    mov board[10*4], 7
    mov board[11*4], 2
    mov board[12*4], 1
    mov board[13*4], 9
    mov board[14*4], 5
    mov board[15*4], 3
    mov board[16*4], 4
    mov board[17*4], 8
    
    ; Row 3
    mov board[18*4], 1
    mov board[19*4], 9
    mov board[20*4], 8
    mov board[21*4], 3
    mov board[22*4], 4
    mov board[23*4], 2
    mov board[24*4], 5
    mov board[25*4], 6
    mov board[26*4], 7
    
    ; Row 4
    mov board[27*4], 8
    mov board[28*4], 5
    mov board[29*4], 9
    mov board[30*4], 7
    mov board[31*4], 6
    mov board[32*4], 1
    mov board[33*4], 4
    mov board[34*4], 2
    mov board[35*4], 3
    
    ; Row 5
    mov board[36*4], 4
    mov board[37*4], 2
    mov board[38*4], 6
    mov board[39*4], 8
    mov board[40*4], 5
    mov board[41*4], 3
    mov board[42*4], 7
    mov board[43*4], 9
    mov board[44*4], 1
    
    ; Row 6
    mov board[45*4], 7
    mov board[46*4], 1
    mov board[47*4], 3
    mov board[48*4], 9
    mov board[49*4], 2
    mov board[50*4], 4
    mov board[51*4], 8
    mov board[52*4], 5
    mov board[53*4], 6
    
    ; Row 7
    mov board[54*4], 9
    mov board[55*4], 6
    mov board[56*4], 1
    mov board[57*4], 5
    mov board[58*4], 3
    mov board[59*4], 7
    mov board[60*4], 2
    mov board[61*4], 8
    mov board[62*4], 4
    
    ; Row 8
    mov board[63*4], 2
    mov board[64*4], 8
    mov board[65*4], 7
    mov board[66*4], 4
    mov board[67*4], 1
    mov board[68*4], 9
    mov board[69*4], 6
    mov board[70*4], 3
    mov board[71*4], 5
    
    ; Row 9
    mov board[72*4], 3
    mov board[73*4], 4
    mov board[74*4], 5
    mov board[75*4], 2
    mov board[76*4], 8
    mov board[77*4], 6
    mov board[78*4], 1
    mov board[79*4], 7
    mov board[80*4], 9
    
    ret
InitializeBoard endp

; Remove numbers to create puzzle
removeNumbers PROC
    push ecx
    push eax
    push esi
    push ebx
    
    ; Remove a number of values to create the puzzle
    mov ecx, 40  ; Number of cells to empty - adjust for difficulty (40-60 is typical)
removeLoop:
    ; Generate random index (0-80)
    mov eax, 81
    call RandomRange
    mov esi, eax
    
    ; Check if cell already empty
    mov ebx, board[esi*4]
    cmp ebx, 0
    je removeLoop  ; If already empty, try another cell
    
    ; Empty the cell
    mov board[esi*4], 0
    loop removeLoop
    
    pop ebx
    pop esi
    pop eax
    pop ecx
    ret
removeNumbers ENDP

; Print the board
DisplayBoard PROC
    pushad
    call Clrscr  ; Clear screen

    ; Print title
    mov edx, OFFSET titleStr
    call WriteString
    call Crlf
    call Crlf

    ; Print top border
    mov edx, OFFSET boardTop
    call WriteString
    call Crlf

    mov ecx, 0       ; Row counter
    mov esi, OFFSET board  ; Board start

DisplayRow:
    ; Print left border
    mov edx, OFFSET barStr
    call WriteString
    mov edx, OFFSET spaceStr
    call WriteString

    mov ebx, 0       ; Column counter
    
DisplayCell:
    mov eax, [esi]    ; Load DWORD board value
    cmp eax, 0
    je DisplaySpace

    add eax, '0'      ; Convert to ASCII
    call WriteChar
    jmp AfterDisplay

DisplaySpace:
    mov al, ' '
    call WriteChar

AfterDisplay:
    mov edx, OFFSET spaceStr
    call WriteString

    inc ebx
    add esi, 4       ; Move to next DWORD

    cmp ebx, 9
    je EndOfRow  ; If all 9 columns printed, go to next row

    cmp ebx, 3
    je PrintColumnDivider
    cmp ebx, 6
    je PrintColumnDivider

    jmp DisplayCell

PrintColumnDivider:
    mov edx, OFFSET barStr
    call WriteString
    mov edx, OFFSET spaceStr
    call WriteString
    jmp DisplayCell

EndOfRow:
    ; Print right border
    mov edx, OFFSET barStr
    call WriteString
    call Crlf

    inc ecx  ; Move to next row

    cmp ecx, 9
    je DisplayBottom

    cmp ecx, 3
    je PrintRowDivider
    cmp ecx, 6
    je PrintRowDivider

    jmp DisplayRow

PrintRowDivider:
    mov edx, OFFSET divider
    call WriteString
    call Crlf
    jmp DisplayRow

DisplayBottom:
    mov edx, OFFSET divider
    call WriteString
    call Crlf

    popad
    ret
DisplayBoard ENDP

; Parse Row Input
ParseRowInput PROC
    push ebx
    push ecx
    push edx
    push esi

    ; Read the input
    mov eax, rownum

    ; Check if in range (1-9)
    cmp eax, 1
    jl InvalidParse
    cmp eax, 9
    jg InvalidParse

    mov rownum, eax  ; Store the valid row number
    mov eax, 1       ; Success
    jmp ParseDone

InvalidParse:
    mov eax, 0       ; Failure

ParseDone:
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret
ParseRowInput ENDP

; Parse Column Input
ParseColInput PROC
    push ebx
    push ecx
    push edx
    push esi

    ; Read the input
    mov eax, colnum

    ; Check if in range (1-9)
    cmp eax, 1
    jl InvalidParse
    cmp eax, 9
    jg InvalidParse

    mov colnum, eax  ; Store the valid column number
    mov eax, 1       ; Success
    jmp ParseDone

InvalidParse:
    mov eax, 0       ; Failure

ParseDone:
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret
ParseColInput ENDP

ParseValInput PROC
    push ebx
    push ecx
    push edx
    push esi

    ; Read the input
    mov eax, valnum

    ; Check if in range (0-9)
    cmp eax, 0
    jl InvalidParse
    cmp eax, 9
    jg InvalidParse

    mov valnum, eax  ; Store the valid value
    mov eax, 1       ; Success
    jmp ParseDone

InvalidParse:
    mov eax, 0       ; Failure

ParseDone:
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret
ParseValInput ENDP

ValidateMove PROC
    push ebx
    push ecx
    push edx
    push esi
    push edi

    ; Convert to 0-based indices
    mov eax, rownum
    dec eax                 ; 0-based row
    mov ebx, colnum
    dec ebx                 ; 0-based column

    ; Check bounds (0-8)
    cmp eax, 0
    jl InvalidValidation
    cmp eax, 8
    jg InvalidValidation
    cmp ebx, 0
    jl InvalidValidation
    cmp ebx, 8
    jg InvalidValidation

    ; Calculate index = row * 9 + col
    imul eax, 9
    add eax, ebx
    mov edi, eax            ; Save cell index

    ; Check if the cell is editable
    cmp editable[edi*4], 0
    je InvalidValidation    ; If not editable, invalid move

    ; If we're clearing a cell (value=0), it's a valid move
    mov ecx, valnum
    cmp ecx, 0
    je ValidMove

    ; Check if position is empty
    cmp board[edi*4], 0
    jne InvalidValidation   ; Cell must be empty for a new value

    ; Now check if the value violates Sudoku rules
    
    ; Save registers
    push eax
    push ebx
    push ecx
    
    ; Check row
    mov edx, rownum
    dec edx         ; Convert to 0-based
    mov esi, 0      ; Column index
CheckRowLoop:
    ; Calculate index = row * 9 + col
    mov eax, edx
    imul eax, 9
    add eax, esi
    
    ; Skip current cell
    cmp eax, edi
    je SkipRowCheck
    
    ; Check if the value exists in this row
    cmp board[eax*4], ecx
    je FailValidation
    
SkipRowCheck:
    inc esi
    cmp esi, 9
    jl CheckRowLoop
    
    ; Check column
    mov edx, colnum
    dec edx         ; Convert to 0-based
    mov esi, 0      ; Row index
CheckColLoop:
    ; Calculate index = row * 9 + col
    mov eax, esi
    imul eax, 9
    add eax, edx
    
    ; Skip current cell
    cmp eax, edi
    je SkipColCheck
    
    ; Check if the value exists in this column
    cmp board[eax*4], ecx
    je FailValidation
    
SkipColCheck:
    inc esi
    cmp esi, 9
    jl CheckColLoop
    
    ; Check 3x3 box
    ; Calculate box starting indices
    mov eax, rownum
    dec eax          ; Convert to 0-based
    xor edx, edx
    mov ebx, 3
    div ebx          ; eax = row / 3
    imul eax, 3      ; eax = (row / 3) * 3 = start row of box
    mov esi, eax     ; esi = start row of box
    
    mov eax, colnum
    dec eax          ; Convert to 0-based
    xor edx, edx
    mov ebx, 3
    div ebx          ; eax = col / 3
    imul eax, 3      ; eax = (col / 3) * 3 = start col of box
    mov ebx, eax     ; ebx = start col of box
    
    ; Check each cell in the 3x3 box
    mov edx, 0       ; Offset for row
CheckBoxRowLoop:
    cmp edx, 3
    je BoxCheckDone
    
    mov eax, 0       ; Offset for column
CheckBoxColLoop:
    cmp eax, 3
    je NextBoxRow
    
    ; Calculate index = (start_row + offset_row) * 9 + (start_col + offset_col)
    push eax
    push edx
    
    add edx, esi     ; row = start_row + offset_row
    imul edx, 9
    add eax, ebx     ; col = start_col + offset_col
    add edx, eax     ; index = row * 9 + col
    
    ; Skip current cell
    cmp edx, edi
    je SkipBoxCheck
    
    ; Check if the value exists in this box cell
    cmp board[edx*4], ecx
    je FailBoxCheck
    
SkipBoxCheck:
    pop edx
    pop eax
    inc eax
    jmp CheckBoxColLoop
    
NextBoxRow:
    inc edx
    jmp CheckBoxRowLoop
    
BoxCheckDone:
    ; All checks passed
    pop ecx
    pop ebx
    pop eax
    jmp ValidMove
    
FailBoxCheck:
    pop edx
    pop eax
    
FailValidation:
    pop ecx
    pop ebx
    pop eax
    jmp InvalidValidation

ValidMove:
    mov eax, 1       ; Valid move
    jmp ValidateDone

InvalidValidation:
    mov eax, 0       ; Invalid move

ValidateDone:
    pop edi
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret
ValidateMove ENDP

UpdateBoard PROC
    push eax
    push ebx
    push ecx
    push edx

    ; Convert 1-based row and column to 0-based
    mov eax, rownum
    dec eax
    mov ebx, colnum
    dec ebx
    
    ; Calculate index = row * 9 + col
    imul eax, 9
    add eax, ebx
    
    ; Check if the cell is editable
    cmp editable[eax*4], 0
    je UpdateDone       ; If not editable, don't update
    
    ; Update board
    mov ecx, valnum
    mov board[eax*4], ecx  ; Store the new value

UpdateDone:
    pop edx
    pop ecx
    pop ebx
    pop eax
    ret
UpdateBoard ENDP

CheckSolved PROC
    push ecx
    push esi
    
    mov ecx, 0      ; Counter
    
CheckCell:
    cmp ecx, 81
    je Solved
    
    cmp board[ecx*4], 0
    je NotSolved
    
    inc ecx
    jmp CheckCell
    
Solved:
    mov eax, 1      ; All cells filled
    jmp CheckDone
    
NotSolved:
    mov eax, 0      ; Some cells still empty
    
CheckDone:
    pop esi
    pop ecx
    ret
CheckSolved ENDP

VerifySolution PROC
    push ebx
    push ecx
    push edx
    push esi
    push edi

    ; Check all rows
    mov ebx, 0          ; Row counter
RowCheck:
    cmp ebx, 9
    je ColumnsCheck
    
    ; Use a bit mask to track numbers 1-9
    mov edx, 0          ; Bitmask for numbers seen
    mov ecx, 0          ; Column counter
    
RowLoop:
    cmp ecx, 9
    je NextRow
    
    ; Calculate index = row * 9 + col
    mov eax, ebx
    imul eax, 9
    add eax, ecx
    
    ; Get value
    mov esi, board[eax*4]
    
    ; Check if empty (shouldn't happen at this point)
    cmp esi, 0
    je InvalidSolution
    
    ; Check if number already seen in this row
    ; Adjust to 0-based for bit testing
    dec esi
    bt edx, esi
    jc InvalidSolution   ; Duplicate found
    
    ; Mark number as seen
    bts edx, esi
    
    inc ecx
    jmp RowLoop

NextRow:
    inc ebx
    jmp RowCheck

ColumnsCheck:
    ; Check all columns
    mov ebx, 0          ; Column counter
    
ColumnCheck:
    cmp ebx, 9
    je BoxesCheck
    
    ; Use a bit mask to track numbers 1-9
    mov edx, 0          ; Bitmask
    mov ecx, 0          ; Row counter
    
ColumnLoop:
    cmp ecx, 9
    je NextColumn
    
    ; Calculate index = row * 9 + col
    mov eax, ecx
    imul eax, 9
    add eax, ebx
    
    ; Get value
    mov esi, board[eax*4]
    
    ; Adjust to 0-based for bit testing
    dec esi
    
    ; Check if number already seen in this column
    bt edx, esi
    jc InvalidSolution   ; Duplicate found
    
    ; Mark number as seen
    bts edx, esi
    
    inc ecx
    jmp ColumnLoop

NextColumn:
    inc ebx
    jmp ColumnCheck

BoxesCheck:
    ; Check all 3x3 boxes
    mov edi, 0          ; Box counter (0-8)
    
BoxLoop:
    cmp edi, 9
    je ValidSolution
    
    ; Calculate box starting position
    mov eax, edi
    xor edx, edx
    mov ebx, 3
    div ebx          ; eax = box / 3, edx = box % 3
    
    ; eax = row start, edx = col start
    imul eax, 3
    imul edx, 3
    
    ; Save starting coordinates
    push eax         ; row start
    push edx         ; col start
    
    ; Use a bit mask to track numbers 1-9
    mov esi, 0       ; Bitmask
    
    ; Check each cell in the box
    mov ebx, 0       ; Row offset
BoxRowLoop:
    cmp ebx, 3
    je BoxDone
    
    mov ecx, 0       ; Col offset
BoxColLoop:
    cmp ecx, 3
    je NextBoxRow
    
    ; Calculate index
    mov eax, [esp+4]  ; row start
    add eax, ebx
    imul eax, 9
    mov edx, [esp]    ; col start
    add edx, ecx
    add eax, edx
    
    ; Get value
    mov edx, board[eax*4]
    
    ; Adjust to 0-based for bit testing
    dec edx
    
    ; Check if number already seen in this box
    bt esi, edx
    jc InvalidBox     ; Duplicate found
    
    ; Mark number as seen
    bts esi, edx
    
    inc ecx
    jmp BoxColLoop
    
NextBoxRow:
    inc ebx
    jmp BoxRowLoop
    
BoxDone:
    ; Box OK, move to next box
    pop edx
    pop eax
    inc edi
    jmp BoxLoop
    
InvalidBox:
    ; Clean up stack and return invalid
    pop edx
    pop eax
    jmp InvalidSolution

ValidSolution:
    mov eax, 1
    jmp VerifyDone

InvalidSolution:
    mov eax, 0

VerifyDone:
    pop edi
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret
VerifySolution ENDP

END main
```

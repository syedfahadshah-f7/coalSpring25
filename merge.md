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
    inc esi
    loop initBoard

    call fillDiagonal   ; Fill diagonal boxes first
    call fillRemaining  ; Fill remaining cells
    call removeNumbers  ; Remove some numbers to create puzzle
    call DisplayBoard     ; Display the board

GameLoop:
    ; Prompt for row
    mov edx, OFFSET promptRowStr
    call WriteString
    mov eax,rownum
    call ReadInt
    mov rownum,eax
    call ParseRowInput
    cmp eax, 0
    je MoveError  ; Handle invalid row input

    mov eax,rownum
    call WriteInt
    call Crlf

    ; Prompt for column
    mov edx, OFFSET promptColStr
    call WriteString
    mov eax,colnum
    call ReadInt
    mov colnum,eax
    call ParseColInput
    cmp eax, 0
    je MoveError  ; Handle invalid column input

    mov eax,colnum
    call WriteInt
    call Crlf

    ; Prompt for value
    mov edx, OFFSET promptValStr
    call WriteString
    mov eax,valnum
    call ReadInt
    mov valnum,eax
    call ParseValInput
    cmp eax, 0
    je MoveError  ; Handle invalid value input

    mov eax,valnum
    call WriteInt
    call Crlf

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
    mov edx, OFFSET invalidStr
    call WriteString
    call Crlf
    jmp GameLoop  ; Return to the start of the game loop

ExitGame:
    exit
main ENDP

fillDiagonal PROC
    mov ecx, 0
boxLoop:
    call fillBox
    add ecx, 3
    cmp ecx, 9
    jl boxLoop
    ret
fillDiagonal ENDP

; Fill a 3x3 box
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

; Remove numbers to create puzzle
removeNumbers PROC
    mov ecx, 58
removeLoop:
    mov eax, 81
    call RandomRange
    mov esi, eax

    cmp board[esi*4], 0
    je removeLoop

    mov board[esi*4], 0
    loop removeLoop
    ret
removeNumbers ENDP

; Print the board
printBoard PROC
    mov esi, 0
    mov ecx, 0
printLoop:
    mov eax, board[esi*4]
    call WriteDec
    mov al, ' '
    call WriteChar
    inc esi
    inc ecx
    cmp ecx, 9
    jl sameLine
    call Crlf
    mov ecx, 0
sameLine:
    cmp esi, 81
    jl printLoop
    ret
printBoard ENDP

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

    ; Convert character to number
    cmp eax, 1                   ; Check if in range (1-9)
    jl InvalidParse
    cmp eax, 9
    jg InvalidParse

    mov rownum, eax             ; Store the valid row number
    mov eax, 1                  ; Success
    jmp ParseDone

InvalidParse:
    mov eax, 0                  ; Failure

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
    mov eax,colnum

    ; Convert character to number
    ;sub al, '0'                 ; Convert ASCII to integer
    cmp eax, 1                   ; Check if in range (1-9)
    jl InvalidParse
    cmp eax, 9
    jg InvalidParse

    mov colnum, eax             ; Store the valid column number
    mov eax, 1                  ; Success
    jmp ParseDone

InvalidParse:
    mov eax, 0                  ; Failure

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
    mov eax,valnum

    ; Convert character to number
    ;sub al, '0'                 ; Convert ASCII to integer
    cmp eax, 0                   ; Check if in range (0-9)
    jl InvalidParse
    cmp eax, 9
    jg InvalidParse

    mov valnum, eax             ; Store the valid value
    mov eax, 1                  ; Success
    jmp ParseDone

InvalidParse:
    mov eax, 0                  ; Failure

ParseDone:
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret
ParseValInput ENDP

ParseInput PROC
    push ebx
    push ecx
    push edx
    push esi
    push edi

    ; Initialize
    mov esi, OFFSET inputBuffer
    mov edi, 0                  ; Number counter (0=row, 1=col, 2=value)

ParseLoop:
    ; Skip whitespace
    mov eax, [esi]
    cmp eax, '0'                   ; End of string?
    je CheckComplete
    jmp CheckDigit

SkipChar:
    inc esi
    jmp ParseLoop

CheckDigit:
    ; Check if digit
    cmp eax, '0'
    jl InvalidParse
    cmp eax, '9'
    jg InvalidParse

    ; Parse the number
    xor eax, eax
    xor ebx, ebx

ParseNumber:
    mov ebx, [esi]
    cmp ebx, '0'
    jl NumberDone
    cmp ebx, '9'
    jg NumberDone
    
    ; Accumulate number
    imul eax, 10
    sub ebx, '0'
    add eax, ebx
    inc esi
    mov ebx, [esi]
    jmp ParseNumber

NumberDone:
    ; Store based on which number we're parsing
    cmp edi, 0
    jne NotRow
    ; Row (1-9)
    cmp eax, 1
    jl InvalidParse
    cmp eax, 9
    jg InvalidParse
    mov row, eax
    inc edi
    jmp ParseLoop

NotRow:
    cmp edi, 1
    jne NotCol
    ; Column (1-9)
    cmp eax, 1
    jl InvalidParse
    cmp eax, 9
    jg InvalidParse
    mov col, eax
    inc edi
    jmp ParseLoop

NotCol:
    ; Value (0-9)
    cmp eax, 0
    jl InvalidParse
    cmp eax, 9
    jg InvalidParse
    mov num, eax
    inc edi

    ; Check for any extra characters
    mov eax, [esi]
    cmp eax, 0
    jne InvalidParse
    jmp CheckComplete

CheckComplete:
    ; Must have all 3 numbers
    cmp edi, 3
    jne InvalidParse

    mov eax, 1        ; Success
    jmp ParseDone

InvalidParse:
    mov eax, 0        ; Failure

ParseDone:
    pop edi
    pop esi
    pop edx
    pop ecx
    pop ebx
    ret
ParseInput ENDP

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

    ; Calculate index = (row * 9 + col) * 4
    imul eax,  9
    add eax, ebx
    shl eax, 2       ; Multiply by 4 for DWORD offset
    mov edi, eax            ; Save cell index

    ; Check if the cell is editable
    cmp editable[eax], 0
    je InvalidValidation   ; If not editable, invalid move

    ; Check if cell is empty or we're clearing it (value=0)
    mov ecx, valnum
    cmp ecx, 0
    je ValidMove            ; Allow clearing any cell
    cmp DWORD PTR [board + edi], 0
    jne InvalidValidation   ; Cell must be empty to place number

    ; Check value is 1-9
    cmp ecx, 1
    jl InvalidValidation
    cmp ecx, 9
    jg InvalidValidation

    ; Check row for duplicate
    mov esi, edi            ; Save current index
    mov eax, edi
    xor edx, edx
    mov ecx, 9              ; Check all 9 columns
RowCheck:
    cmp eax, esi            ; Skip current cell
    je SkipRowCheck
    cmp DWORD PTR [board + eax], ecx
    je DuplicateFound
SkipRowCheck:
    add eax, 4              ; Move to next DWORD
    loop RowCheck

    ; Check column for duplicate
    mov eax, edi            ; Original index
    xor edx, edx
    mov ecx, 9              ; Check all 9 rows
ColCheck:
    ; Calculate index = (row * 9) + column offset
    mov eax, ebx
    imul eax, 9
    add eax, edx            ; eax = (row * 9) + column offset
    cmp eax, edi            ; Skip current cell
    je SkipColCheck
    cmp DWORD PTR [board + eax], ecx
    je DuplicateFound
SkipColCheck:
    inc edx
    cmp edx, 9
    jl ColCheck

    ; Check 3x3 box for duplicate
    mov eax, edi            ; Original index
    xor edx, edx
    mov ecx, 3              ; 3 rows in box
    div ecx                 ; eax = row (0-8), edx = byte offset

    ; Calculate box starting row = (row / 3) * 3
    push eax
    xor edx, edx
    mov ecx, 3
    div ecx                 ; eax = row / 3
    imul eax, 3             ; eax = box starting row
    mov ebx, eax            ; ebx = box starting row
    pop eax

    ; Calculate box starting column = (column / 3) * 3
    mov eax, ebx            ; column byte offset (0-32)
    shr eax, 2              ; Convert to column index (0-8)
    xor edx, edx
    mov ecx, 3
    div ecx                 ; eax = column / 3
    imul eax, 3             ; eax = box starting column
    shl eax, 2              ; Convert back to byte offset

    ; Calculate starting index = (box starting row * 9 * 4) + (box starting column * 4)
    imul ebx, 9
    add ebx, eax            ; ebx = starting index of box

    ; Check all 9 cells in the box
    mov ecx, 3              ; 3 rows in box
BoxRowCheck:
    push ecx
    mov ecx, 3              ; 3 columns in box
    mov eax, ebx            ; start of row in box
    mov edx, valnum         ; number to validate
BoxColCheck:
    cmp eax, edi            ; skip the original cell
    je SkipBoxCheck
    cmp DWORD PTR [board + eax], edx
    je DuplicateFound
SkipBoxCheck:
    add eax, 4              ; move to next column (DWORD)
    loop BoxColCheck
    add ebx, 36             ; move to next row (9 cells * 4 bytes = 36)
    pop ecx
    loop BoxRowCheck

ValidMove:
    mov eax, 1              ; valid move
    jmp ValidateDone

DuplicateFound:
InvalidValidation:
    mov eax, 0              ; invalid move

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
    
    ; Calculate index = (row * 9 + col) * 4
    imul eax, 9
    add eax, ebx
    shl eax, 2       ; Multiply by 4 for DWORD offset

    ; Check if the cell is editable
    cmp editable[eax], 0
    je NotEditable     ; If not editable, jump to error handling

    ; If the value is 0, do not allow clearing of non-editable cells
    cmp valnum, 0
    je NotEditable     ; If trying to clear a non-editable cell, jump to error handling

    ; Update board
    mov ecx, valnum
    mov [board + eax], ecx  ; Store DWORD value

    pop edx
    pop ecx
    pop ebx
    pop eax
    ret

NotEditable:
    ; Handle the case where the player tries to edit a non-editable cell
    mov eax, 0          ; Indicate invalid move
    jmp UpdateDone

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
    
    mov ecx, 0
    mov esi, OFFSET board
    
CheckCell:
    cmp ecx, 81
    je Solved
    cmp DWORD PTR [esi], 0
    je NotSolved
    inc ecx
    add esi, 4       ; Move to next DWORD
    jmp CheckCell
    
Solved:
    mov eax, 1
    jmp CheckDone
    
NotSolved:
    mov eax, 0
    
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
    mov edx, 0          ; Bitmask for numbers seen
    mov ecx, 0          ; Column counter
RowLoop:
    cmp ecx, 9
    je NextRow
    ; Calculate index = (row * 9 + col) * 4
    mov eax, ebx
    imul eax, 9
    add eax, ecx
    shl eax, 2          ; Multiply by 4 for DWORD offset
    mov esi, [board + eax]
    cmp esi, 0          ; Shouldn't happen since CheckSolved passed
    je InvalidSolution
    dec esi             ; Convert to 0-based
    bt edx, esi
    jc InvalidSolution
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
    mov edx, 0          ; Bitmask
    mov ecx, 0          ; Row counter
ColumnLoop:
    cmp ecx, 9
    je NextColumn
    ; Calculate index = (row * 9 + col) * 4
    mov eax, ecx
    imul eax, 9
    add eax, ebx
    shl eax, 2          ; ```assembly
    ; Multiply by 4 for DWORD offset
    mov esi, [board + eax]
    dec esi
    bt edx, esi
    jc InvalidSolution
    bts edx, esi
    inc ecx
    jmp ColumnLoop

NextColumn:
    inc ebx
    jmp ColumnCheck

BoxesCheck:
    ; Check all 3x3 boxes
    mov ebx, 0          ; Box row counter (0-2)
BoxRowLoop:
    cmp ebx, 3
    je ValidSolution
    mov ecx, 0          ; Box column counter (0-2)
BoxColLoop:
    cmp ecx, 3
    je NextBoxRow
    mov edx, 0          ; Bitmask
    
    ; Calculate box boundaries
    mov eax, ebx
    imul eax, 3         ; Starting row
    mov edi, eax
    add edi, 3          ; Ending row
    
    mov eax, ecx
    imul eax, 3         ; Starting column
    mov esi, eax
    add esi, 3          ; Ending column
    
    ; Check each cell in box
    mov eax, edi
    sub eax, 3          ; Reset to starting row
BoxCellRow:
    cmp eax, edi
    je NextBoxCol
    mov ebp, esi
    sub ebp, 3          ; Reset to starting column
BoxCellCol:
    cmp ebp, esi
    je NextBoxCellRow
    ; Calculate index = (row * 9 + col) * 4
    push eax
    imul eax, 9
    add eax, ebp
    shl eax, 2          ; Multiply by 4 for DWORD offset
    mov eax, [board + eax]
    dec eax
    bt edx, eax
    jc InvalidSolution
    bts edx, eax
    pop eax
    inc ebp
    jmp BoxCellCol

NextBoxCellRow:
    inc eax
    jmp BoxCellRow

NextBoxCol:
    inc ecx
    jmp BoxColLoop

NextBoxRow:
    inc ebx
    jmp BoxRowLoop

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

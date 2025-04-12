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
    invalidStr BYTE "Invalid input! Try again.", 0
    winStr BYTE "Congratulations! You solved the Sudoku!", 0
    errorStr BYTE "Invalid move! Try again.", 0
    boardTop BYTE "  +-------+-------+-------+", 0
    divider BYTE "  +-------+-------+-------+", 0
    spaceStr BYTE " ", 0
    barStr BYTE "|", 0
    newline BYTE 0Dh, 0Ah, 0
    
    ; Input variables
    inputBuffer BYTE 16 DUP(?)
    ;row BYTE ?
    ;col BYTE ?
    ;value BYTE ?

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
    ;call DisplayBoard

GameLoop:
    ; Prompt for input
    mov edx, OFFSET promptStr
    call WriteString
    
    ; Read input
    mov edx, OFFSET inputBuffer
    mov ecx, SIZEOF inputBuffer
    call ReadString
    
    ; Check for exit
    cmp BYTE PTR [inputBuffer], '0'
    je ExitGame
    
    ; Parse input
    call ParseInput
    cmp eax, 0
    je InputError
    
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
    
    ; Only show win message if both conditions are met
    mov edx, OFFSET winStr
    call WriteString
    call Crlf
    jmp ExitGame

InputError:
    mov edx, OFFSET invalidStr
    call WriteString
    call Crlf
    jmp GameLoop

MoveError:
    mov edx, OFFSET errorStr
    call WriteString
    call Crlf
    jmp GameLoop

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

    inc col
    inc ecx
    cmp ecx, 3
    jl fillLoop

    ; Reset column and move to next row
    mov col, edi
    inc row
    add edi,3
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
    mov ecx, 30
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
    mov al, [esi]
    cmp al, 0                   ; End of string?
    je CheckComplete
    cmp al, ' '                 ; Space
    je SkipChar
    cmp al, 9                   ; Tab
    je SkipChar
    cmp al, ','                 ; Comma
    je SkipChar
    jmp CheckDigit

SkipChar:
    inc esi
    jmp ParseLoop

CheckDigit:
    ; Check if digit
    cmp al, '0'
    jl InvalidParse
    cmp al, '9'
    jg InvalidParse

    ; Parse the number
    xor eax, eax
    xor ebx, ebx

ParseNumber:
    mov bl, [esi]
    cmp bl, '0'
    jl NumberDone
    cmp bl, '9'
    jg NumberDone
    
    ; Accumulate number
    imul eax, 10
    sub bl, '0'
    add eax, ebx
    inc esi
    mov bl, [esi]
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
    mov al, [esi]
    cmp al, 0
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

ValidateMove PROC
    push ebx
    push ecx
    push edx
    push esi
    push edi

    ; Convert to 0-based indices
    mov eax, row
    dec eax                 ; 0-based row
    mov ebx, col
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
    imul eax, 9
    add eax, ebx
    shl eax, 2       ; Multiply by 4 for DWORD offset
    mov edi, eax            ; Save cell index

    ; Check if cell is empty or we're clearing it (value=0)
    mov ecx, num
    cmp ecx, 0
    je ValidMove            ; Allow clearing any cell
    cmp DWORD PTR [board + eax], 0
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
    mov ecx, 36             ; 9 columns * 4 bytes
    div ecx                 ; eax = row number (0-8), edx = byte offset in row
    imul eax, 36            ; eax = start of row (row * 36 bytes)
    
    mov ecx, 9              ; Check all 9 columns
    mov edx, num
RowCheck:
    cmp eax, esi            ; Skip current cell
    je SkipRowCheck
    cmp DWORD PTR [board + eax], edx
    je DuplicateFound
SkipRowCheck:
    add eax, 4              ; Move to next DWORD
    loop RowCheck

    ; Check column for duplicate
    mov eax, edi            ; Original index
    xor edx, edx
    mov ecx, 36             ; 9 rows * 4 bytes
    div ecx                 ; eax = row, edx = byte offset
    mov ebx, edx            ; ebx = column byte offset (0,4,8,...,32)
    
    mov ecx, 9              ; Check all 9 rows
    mov edx, num
ColCheck:
    ; Calculate index = (row * 36) + column offset
    mov eax, ecx
    dec eax                 ; row index (0-8)
    imul eax, 36
    add eax, ebx            ; eax = (row * 36) + column offset
    
    cmp eax, edi            ; Skip current cell
    je SkipColCheck
    cmp DWORD PTR [board + eax], edx
    je DuplicateFound
SkipColCheck:
    loop ColCheck

    ; Check 3x3 box for duplicate
    mov eax, edi            ; Original index
    xor edx, edx
    mov ecx, 36             ; 9 rows * 4 bytes
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
    mov eax, edx            ; column byte offset (0-32)
    shr eax, 2              ; Convert to column index (0-8)
    xor edx, edx
    mov ecx, 3
    div ecx                 ; eax = column / 3
    imul eax, 3             ; eax = box starting column
    shl eax, 2              ; Convert back to byte offset
    
    ; Calculate starting index = (box starting row * 36) + (box starting column * 4)
    imul ebx, 36
    add ebx, eax            ; ebx = starting index of box
    
    ; Check all 9 cells in the box
    mov ecx, 3              ; 3 rows in box
BoxRowCheck:
    push ecx
    mov ecx, 3              ; 3 columns in box
    mov eax, ebx            ; start of row in box
    mov edx, num        ; number to validate
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
    shl eax, 2          ; Multiply by 4 for DWORD offset
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

UpdateBoard PROC
    push eax
    push ebx
    push ecx
    push edx

    ; Convert 1-based row and column to 0-based
    mov eax, row
    dec eax
    mov ebx, col
    dec ebx
    
    ; Calculate index = (row * 9 + col) * 4
    imul eax, 9
    add eax, ebx
    shl eax, 2       ; Multiply by 4 for DWORD offset

    ; Update board
    mov ecx, num
    mov [board + eax], ecx  ; Store DWORD value

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

END main
```

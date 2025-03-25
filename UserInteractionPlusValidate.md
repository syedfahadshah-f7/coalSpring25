```asm
include Irvine32.inc

.data
    ; Sudoku board (0 = empty) using DWORD
    board DWORD 5,3,0,0,7,0,0,0,0
          DWORD 6,0,0,1,9,5,0,0,0
          DWORD 0,9,8,0,0,0,0,6,0
          DWORD 8,0,0,0,6,0,0,0,3
          DWORD 4,0,0,8,0,3,0,0,1
          DWORD 7,0,0,0,2,0,0,0,6
          DWORD 0,6,0,0,0,0,2,8,0
          DWORD 0,0,0,4,1,9,0,0,5
          DWORD 0,0,0,0,8,0,0,7,9

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
    row BYTE ?
    col BYTE ?
    value BYTE ?

.code
main PROC
    call DisplayBoard

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
    mov row, al
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
    mov col, al
    inc edi
    jmp ParseLoop

NotCol:
    ; Value (0-9)
    cmp eax, 0
    jl InvalidParse
    cmp eax, 9
    jg InvalidParse
    mov value, al
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
    movzx eax, row
    dec eax                 ; 0-based row
    movzx ebx, col
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
    movzx ecx, value
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
    movzx edx, value
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
    movzx edx, value
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
    mov ecx, 3              ; 3 rows
BoxRowCheck:
    push ecx
    mov ecx, 3              ; 3 columns
    mov eax, ebx            ; start of row in box
    movzx edx, value
BoxColCheck:
    cmp eax, edi            ; compare with original index
    je SkipBoxCheck
    cmp DWORD PTR [board + eax], edx
    je DuplicateFound
SkipBoxCheck:
    add eax, 4              ; move to next column
    loop BoxColCheck
    add ebx, 36             ; move to next row in box (36 bytes per row)
    pop ecx
    loop BoxRowCheck

ValidMove:
    mov eax, 1
    jmp ValidationDone

DuplicateFound:
InvalidValidation:
    mov eax, 0

ValidationDone:
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
    movzx eax, row
    dec eax
    movzx ebx, col
    dec ebx
    
    ; Calculate index = (row * 9 + col) * 4
    imul eax, 9
    add eax, ebx
    shl eax, 2       ; Multiply by 4 for DWORD offset

    ; Update board
    movzx ecx, value
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

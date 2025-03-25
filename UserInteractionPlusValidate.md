```asm
include Irvine32.inc

.data
    ; Sudoku board (0 = empty)
    board BYTE 5,3,0,0,7,0,0,0,0
          BYTE 6,0,0,1,9,5,0,0,0
          BYTE 0,9,8,0,0,0,0,6,0
          BYTE 8,0,0,0,6,0,0,0,3
          BYTE 4,0,0,8,0,3,0,0,1
          BYTE 7,0,0,0,2,0,0,0,6
          BYTE 0,6,0,0,0,0,2,8,0
          BYTE 0,0,0,4,1,9,0,0,5
          BYTE 0,0,0,0,8,0,0,7,9

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
    mov al, [esi]    ; Load board value

    cmp al, 0
    je DisplaySpace

    add al, '0'      ; Convert to ASCII
    call WriteChar
    jmp AfterDisplay

DisplaySpace:
    mov al, ' '
    call WriteChar

AfterDisplay:
    mov edx, OFFSET spaceStr
    call WriteString

    inc ebx
    inc esi

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

ParseNumber PROC
    xor eax, eax        ; Clear result
    
ParseDigit:
    mov bl, [esi]
    cmp bl, '0'
    jl ParseDone        ; Not a digit
    cmp bl, '9'
    jg ParseDone        ; Not a digit
    
    ; Convert digit and accumulate
    imul eax, 10
    sub bl, '0'
    add eax, ebx
    inc esi
    jmp ParseDigit
    
ParseDone:
    ret
ParseNumber ENDP

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

    ; Calculate index = row * 9 + column
    imul eax, 9
    add eax, ebx
    mov edi, eax            ; Save cell index

    ; Check if cell is empty or we're clearing it (value=0)
    movzx ecx, value
    cmp ecx, 0
    je ValidMove            ; Allow clearing any cell
    cmp board[eax], 0
    jne InvalidValidation   ; Cell must be empty to place number

    ; Check value is 1-9
    cmp ecx, 1
    jl InvalidValidation
    cmp ecx, 9
    jg InvalidValidation

    ; ========== FIXED ROW VALIDATION ==========
    ; Check row for duplicate
    mov esi, eax            ; Save current index
    mov eax, esi
    xor edx, edx
    mov ecx, 9
    div ecx                 ; eax = row number (0-8), edx = column (0-8)
    imul eax, 9             ; eax = start of row (row * 9)
    
    mov ecx, 9              ; Check all 9 columns
    movzx edx, value
RowCheck:
    cmp eax, esi            ; Skip current cell
    je SkipRowCheck
    cmp board[eax], dl
    je DuplicateFound
SkipRowCheck:
    inc eax
    loop RowCheck

    ; ========== COLUMN VALIDATION ==========
    ; Check column for duplicate
    mov eax, edi            ; Original index
    xor edx, edx
    mov ecx, 9
    div ecx                 ; eax = row, edx = column
    mov ebx, edx            ; ebx = column number (0-8)
    
    mov ecx, 9              ; Check all 9 rows
    movzx edx, value
ColCheck:
    ; Calculate index = row * 9 + column
    mov eax, ecx
    dec eax                 ; row index (0-8)
    imul eax, 9
    add eax, ebx            ; eax = row * 9 + column
    
    cmp eax, edi            ; Skip current cell
    je SkipColCheck
    cmp board[eax], dl
    je DuplicateFound
SkipColCheck:
    loop ColCheck

    ; ========== BOX VALIDATION ==========
    ; Check 3x3 box for duplicate
    mov eax, edi            ; Original index
    xor edx, edx
    mov ecx, 9
    div ecx                 ; eax = row (0-8), edx = column (0-8)
    
    ; Calculate box starting row = (row / 3) * 3
    push eax
    xor edx, edx
    mov ecx, 3
    div ecx                 ; eax = row / 3
    imul eax, 3             ; eax = box starting row
    mov ebx, eax            ; ebx = box starting row
    pop eax
    
    ; Calculate box starting column = (column / 3) * 3
    mov eax, edx            ; column (0-8)
    xor edx, edx
    mov ecx, 3
    div ecx                 ; eax = column / 3
    imul eax, 3             ; eax = box starting column
    
    ; Calculate starting index = (box starting row * 9) + box starting column
    imul ebx, 9
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
    cmp board[eax], dl
    je DuplicateFound
SkipBoxCheck:
    inc eax
    loop BoxColCheck
    add ebx, 9              ; move to next row in box
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
    mov eax, ebx
    imul eax, 9
    add eax, ecx
    movzx esi, board[eax]
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
    mov eax, ecx
    imul eax, 9
    add eax, ebx
    movzx esi, board[eax]
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
    push eax
    imul eax, 9
    add eax, ebp
    movzx eax, board[eax]
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
    imul eax, 9      ; eax = row * 9
    add eax, ebx     ; eax = row * 9 + col

    ; Update board
    movzx ecx, value
    mov board[eax], cl  ; Store the value

    pop edx
    pop ecx
    pop ebx
    pop eax
    ret
UpdateBoard ENDP

CheckSolved PROC
    pushad
    mov ecx, 81           ; Number of cells in board
    mov esi, OFFSET board
    mov eax, 0            ; Counter for empty cells

CheckLoop:
    cmp BYTE PTR [esi], 0
    jne SkipCount
    inc eax               ; Count empty cells

SkipCount:
    inc esi
    loop CheckLoop

    ; Debug: Print empty cell count
    mov edx, OFFSET newline
    call WriteString
    mov edx, OFFSET spaceStr
    call WriteString
    call WriteDec   ; Print number of empty cells
    call Crlf

    ; If no empty cells, return 1
    cmp eax, 0
    je Solved
    mov eax, 0
    jmp Done

Solved:
    mov eax, 1

Done:
    popad
    ret
CheckSolved ENDP

END main
```

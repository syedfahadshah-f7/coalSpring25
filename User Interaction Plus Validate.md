```asm
.686
.model flat, stdcall
option casemap :none

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
    je InvalidInput
    
    ; Validate move
    call ValidateMove
    cmp eax, 0
    je InvalidMove
    
    ; Update board
    call UpdateBoard
    
    ; Display updated board
    call DisplayBoard
    
    ; Check if solved
    call CheckSolved
    cmp eax, 1
    je GameWon
    
    jmp GameLoop

InvalidInput:
    mov edx, OFFSET invalidStr
    call WriteString
    call Crlf
    jmp GameLoop

InvalidMove:
    mov edx, OFFSET errorStr
    call WriteString
    call Crlf
    jmp GameLoop

GameWon:
    mov edx, OFFSET winStr
    call WriteString
    call Crlf

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
    dec eax
    movzx ebx, col
    dec ebx

    ; Ensure row and col are within bounds (0-8)
    cmp eax, 0
    jl InvalidValidation
    cmp eax, 8
    jg InvalidValidation
    cmp ebx, 0
    jl InvalidValidation
    cmp ebx, 8
    jg InvalidValidation

    ; Compute board index
    imul eax, 9
    add eax, ebx

    ; Check if cell is empty (0) or we're clearing it (value=0)
    movzx ecx, value
    cmp ecx, 0
    je ValidMove  ; Allow clearing any cell
    cmp board[eax], 0
    jne InvalidValidation

    ; Check if value is valid (1-9)
    cmp ecx, 1
    jl InvalidValidation
    cmp ecx, 9
    jg InvalidValidation

    ; Check row for duplicate
    push eax
    push ebx
    mov edi, eax    ; Save cell index
    and eax, 0FFFFFFF8h  ; Get start of row (row * 9)
    mov ecx, 9      ; Check all 9 columns
    movzx edx, value
RowCheck:
    cmp eax, edi    ; Skip current cell
    je SkipRowCheck
    cmp board[eax], dl
    je DuplicateFound
SkipRowCheck:
    inc eax
    loop RowCheck
    pop ebx
    pop eax

    ; Check column for duplicate
    push eax
    push ebx
    mov edi, ebx    ; column number
    mov eax, ebx    ; start at top of column
    mov ecx, 9      ; check all 9 rows
    movzx edx, value
ColCheck:
    cmp eax, edi    ; Skip current cell (row*9 + col)
    je SkipColCheck
    cmp board[eax], dl
    je DuplicateFound
SkipColCheck:
    add eax, 9      ; move down one row
    loop ColCheck
    pop ebx
    pop eax

    ; Check 3x3 box for duplicate
    push eax
    push ebx
    ; Calculate top-left corner of 3x3 box
    mov eax, ebx    ; column
    mov edx, 0
    mov ecx, 3
    div ecx         ; eax = column / 3
    imul eax, 3     ; eax = (column / 3) * 3 (start column of box)
    mov ebx, eax    ; save start column

    mov eax, edi    ; original index
    mov edx, 0
    mov ecx, 27     ; row / 3 * 27
    div ecx
    imul eax, 27    ; eax = (row / 3) * 27 (start row of box)
    add eax, ebx    ; eax = start index of box
    mov edi, eax    ; save start index

    mov ecx, 3      ; 3 rows
BoxRowCheck:
    push ecx
    mov ecx, 3      ; 3 columns
    mov eax, edi    ; start of row in box
    movzx edx, value
BoxColCheck:
    cmp eax, [esp + 12] ; compare with original index (pushed eax)
    je SkipBoxCheck
    cmp board[eax], dl
    je DuplicateFound
SkipBoxCheck:
    inc eax
    loop BoxColCheck
    add edi, 9      ; move to next row in box
    pop ecx
    loop BoxRowCheck
    pop ebx
    pop eax

ValidMove:
    mov eax, 1
    jmp ValidationDone

DuplicateFound:
    pop ebx
    pop eax
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
    push ecx
    
    mov ecx, 0
CheckCell:
    cmp ecx, 81
    je Solved
    cmp board[ecx], 0
    je NotSolved
    inc ecx
    jmp CheckCell
    
Solved:
    mov eax, 1
    jmp CheckDone
    
NotSolved:
    mov eax, 0
    
CheckDone:
    pop ecx
    ret
CheckSolved ENDP

END main
```

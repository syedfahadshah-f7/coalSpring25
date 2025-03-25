```asm
INCLUDE Irvine32.inc

.DATA
    board DWORD 81 DUP(0)  ; 9x9 Sudoku board
    row DWORD ?
    col DWORD ?
    num DWORD ?

.CODE
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
    call printBoard     ; Display the board

    exit
main ENDP

; Fill diagonal 3x3 boxes
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

END main

```

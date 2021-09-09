```asm
000000000009a6a0         push       rbp                                         ; CODE XREF=Precompiled____main_main_1435+17
000000000009a6a1         mov        rbp, rsp
000000000009a6a4         sub        rsp, 0x18
000000000009a6a8         mov        rdx, qword [r15+0x1e07]
000000000009a6af         cmp        rsp, qword [r14+0x40]
000000000009a6b3         jbe        loc_9a7a6

                     loc_9a6b9:
000000000009a6b9         mov        rsi, qword [rdx+7]                          ; CODE XREF=Precompiled____main_1434+269
000000000009a6bd         mov        qword [rbp+var_10], rsi
000000000009a6c1         xor        edi, edi

                     loc_9a6c3:
000000000009a6c3         mov        qword [rbp+var_8], rdi                      ; CODE XREF=Precompiled____main_1434+257
000000000009a6c7         cmp        rsp, qword [r14+0x40]
000000000009a6cb         jbe        loc_9a7b2

                     loc_9a6d1:
000000000009a6d1         cmp        rdi, 0x2                                    ; CODE XREF=Precompiled____main_1434+281
000000000009a6d5         jl         loc_9a6ec

000000000009a6db         call       Precompiled____exit_1023                    ; Precompiled____exit_1023
000000000009a6e0         mov        rax, qword [r14+0xc8]
000000000009a6e7         mov        rsp, rbp
000000000009a6ea         pop        rbp
000000000009a6eb         ret
                        ; endp

                     loc_9a6ec:
000000000009a6ec         mov        rax, rdi                                    ; CODE XREF=Precompiled____main_1434+53
000000000009a6ef         add        rax, rax
000000000009a6f2         jno        loc_9a701

000000000009a6f8         call       Precompiled_Stub__iso_stub_AllocateMintSharedWithoutFPURegsStub ; Precompiled_Stub__iso_stub_AllocateMintSharedWithoutFPURegsStub
000000000009a6fd         mov        qword [rax+7], rdi

                     loc_9a701:
000000000009a701         movzx      rcx, word [rdx+1]                           ; CODE XREF=Precompiled____main_1434+82
000000000009a706         mov        r11, qword [r15+0x1e07]
000000000009a70d         push       r11
000000000009a70f         push       rax
000000000009a710         mov        rax, qword [r14+0x60]
000000000009a714         call       qword [rax+rcx*8]
000000000009a717         pop        r11
000000000009a719         pop        r11
000000000009a71b         mov        rbx, rax
000000000009a71e         mov        rsi, qword [rbp+var_8]
000000000009a722         mov        qword [rbp+var_18], rbx
000000000009a726         add        rsi, 0x1
000000000009a72a         mov        qword [rbp+var_8], rsi
000000000009a72e         cmp        rbx, qword [r14+0xc8]
000000000009a735         jne        loc_9a76b

000000000009a73b         mov        rax, rbx
000000000009a73e         mov        rdx, qword [rbp+var_10]
000000000009a742         mov        rcx, qword [r14+0xc8]
000000000009a749         cmp        rdx, qword [r14+0xc8]
000000000009a750         je         loc_9a76b

000000000009a756         mov        rsi, qword [rdx+0x27]
000000000009a75a         mov        rbx, qword [r15+0xb7]
000000000009a761         mov        r9, qword [r15+0x1e0f]
000000000009a768         call       qword [rsi+7]

                     loc_9a76b:
000000000009a76b         mov        rax, qword [rbp+var_18]                     ; CODE XREF=Precompiled____main_1434+149, Precompiled____main_1434+176
000000000009a76f         test       al, 0x1
000000000009a771         mov        ecx, 0x35
000000000009a776         je         loc_9a77d

000000000009a778         movzx      rcx, word [rax+1]

                     loc_9a77d:
000000000009a77d         push       rax                                         ; CODE XREF=Precompiled____main_1434+214
000000000009a77e         mov        rax, qword [r14+0x60]
000000000009a782         call       qword [rax+rcx*8+0x58d8]
000000000009a789         pop        r11
000000000009a78b         push       rax
000000000009a78c         call       Precompiled____printToConsole_149           ; Precompiled____printToConsole_149
000000000009a791         pop        rcx
000000000009a792         mov        rdi, qword [rbp+var_8]
000000000009a796         mov        rsi, qword [rbp+var_10]
000000000009a79a         mov        rdx, qword [r15+0x1e07]
000000000009a7a1         jmp        loc_9a6c3

                     loc_9a7a6:
000000000009a7a6         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1434+19
000000000009a7ad         jmp        loc_9a6b9

                     loc_9a7b2:
000000000009a7b2         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1434+43
000000000009a7b9         jmp        loc_9a6d1
```
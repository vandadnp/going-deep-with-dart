# `for` loop in Dart

Let's go deep into how `for` loops work in Dart.

## What is a `for` loop?

There are two types of `for` loops in Dart:

1. `for ([final/var] x = N; x [<|<=|>|>=] y; x [+= y|++|--|-=])`
2. `for (final x in y)` where y is an `Iterable`

we will start off by looking at the first for loop and see what we learn!

## `for` loop with index

given the following Dart code:

```dart
import 'dart:io' show exit;

void main(List<String> args) {
  for (var x = 0xDEADBEEF; x < 0xFEEDFEED; x++) {
    print(x);
  }
  exit(0);
}
```

we will get the following AOT:

```asm
000000000009a6a0         push       rbp                                         ; CODE XREF=Precompiled____main_main_1435+17
000000000009a6a1         mov        rbp, rsp
000000000009a6a4         sub        rsp, 0x8
000000000009a6a8         cmp        rsp, qword [r14+0x40]
000000000009a6ac         jbe        loc_9a72a

                     loc_9a6b2:
000000000009a6b2         mov        edx, 0xdeadbeef                             ; CODE XREF=Precompiled____main_1434+145

                     loc_9a6b7:
000000000009a6b7         mov        qword [rbp+var_8], rdx                      ; CODE XREF=Precompiled____main_1434+119
000000000009a6bb         cmp        rsp, qword [r14+0x40]
000000000009a6bf         jbe        loc_9a736

                     loc_9a6c5:
000000000009a6c5         mov        r11d, 0xfeedfeed                            ; CODE XREF=Precompiled____main_1434+157
000000000009a6cb         cmp        rdx, r11
000000000009a6ce         jge        loc_9a719

000000000009a6d4         mov        rax, rdx
000000000009a6d7         add        rax, rax
000000000009a6da         jno        loc_9a6e9

000000000009a6e0         call       Precompiled_Stub__iso_stub_AllocateMintSharedWithoutFPURegsStub ; Precompiled_Stub__iso_stub_AllocateMintSharedWithoutFPURegsStub
000000000009a6e5         mov        qword [rax+7], rdx

                     loc_9a6e9:
000000000009a6e9         test       al, 0x1                                     ; CODE XREF=Precompiled____main_1434+58
000000000009a6eb         mov        ecx, 0x35
000000000009a6f0         je         loc_9a6f7

000000000009a6f2         movzx      rcx, word [rax+1]

                     loc_9a6f7:
000000000009a6f7         push       rax                                         ; CODE XREF=Precompiled____main_1434+80
000000000009a6f8         mov        rax, qword [r14+0x60]
000000000009a6fc         call       qword [rax+rcx*8+0x58d8]
000000000009a703         pop        r11
000000000009a705         push       rax
000000000009a706         call       Precompiled____printToConsole_149           ; Precompiled____printToConsole_149
000000000009a70b         pop        rcx
000000000009a70c         mov        rax, qword [rbp+var_8]
000000000009a710         add        rax, 0x1
000000000009a714         mov        rdx, rax
000000000009a717         jmp        loc_9a6b7

                     loc_9a719:
000000000009a719         call       Precompiled____exit_1023                    ; Precompiled____exit_1023, CODE XREF=Precompiled____main_1434+46
000000000009a71e         mov        rax, qword [r14+0xc8]
000000000009a725         mov        rsp, rbp
000000000009a728         pop        rbp
000000000009a729         ret
                        ; endp

                     loc_9a72a:
000000000009a72a         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1434+12
000000000009a731         jmp        loc_9a6b2

                     loc_9a736:
000000000009a736         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1434+31
000000000009a73d         jmp        loc_9a6c5
```
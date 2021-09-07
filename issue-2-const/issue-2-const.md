# `const` in Dart

Let's go deep with `const` in Dart and see how it works under the hood.

## What is `const`?

`const` denotes a variable whose value is known at compile-time. The value of the variable cannot be overwritten during runtime and nor can the value change internally.

## `const` basic data types

given the following Dart code:

```dart
import 'dart:io' show exit;

const intConst = 0xDEADBEEF;
const strConst = 'Hello, World!';
const doubleConst = 1.2;
void main(List<String> args) {
  print(intConst);
  print(strConst);
  print(doubleConst);
  exit(0);
}
```

we will get the following AOT compilation:

```asm
                     Precompiled____main_1558:
000000000005fad8         push       rbp                                         ; CODE XREF=Precompiled____main_main_1559+17
000000000005fad9         mov        rbp, rsp
000000000005fadc         cmp        rsp, qword [r14+0x40]
000000000005fae0         jbe        loc_5fb24

                     loc_5fae6:
000000000005fae6         mov        r11, qword [r15+0x207f]                     ; CODE XREF=Precompiled____main_1558+83
000000000005faed         push       r11
000000000005faef         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005faf4         pop        rcx
000000000005faf5         mov        r11, qword [r15+0x2087]
000000000005fafc         push       r11
000000000005fafe         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb03         pop        rcx
000000000005fb04         mov        r11, qword [r15+0x208f]
000000000005fb0b         push       r11
000000000005fb0d         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb12         pop        rcx
000000000005fb13         call       Precompiled____exit_1070                    ; Precompiled____exit_1070
000000000005fb18         mov        rax, qword [r14+0xc8]
000000000005fb1f         mov        rsp, rbp
000000000005fb22         pop        rbp
000000000005fb23         ret
                        ; endp

                     loc_5fb24:
000000000005fb24         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1558+8
000000000005fb2b         jmp        loc_5fae6
```

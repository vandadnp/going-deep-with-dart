# `const` in Dart

Let's go deep with `const` in Dart and see how it works under the hood.

## What is `const`?

`const` denotes a variable whose value is known at compile-time. The value of the variable cannot be overwritten during runtime and nor can the value change internally.

## `const int`

given the following Dart code:

```dart
import 'dart:io' show exit;

const intConst = 0xDEADBEEF;
void main(List<String> args) {
  print(intConst);
  exit(0);
}
```

we will get the following AOT compilation:

```asm
                     Precompiled____main_1558:
000000000005faec         push       rbp                                         ; CODE XREF=Precompiled____main_main_1559+17
000000000005faed         mov        rbp, rsp
000000000005faf0         cmp        rsp, qword [r14+0x40]
000000000005faf4         jbe        loc_5fb17

                     loc_5fafa:
000000000005fafa         mov        eax, 0xdeadbeef                             ; CODE XREF=Precompiled____main_1558+50
000000000005faff         push       rax
000000000005fb00         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb05         pop        rcx
000000000005fb06         call       Precompiled____exit_1070                    ; Precompiled____exit_1070
000000000005fb0b         mov        rax, qword [r14+0xc8]
000000000005fb12         mov        rsp, rbp
000000000005fb15         pop        rbp
000000000005fb16         ret
                        ; endp

                     loc_5fb17:
000000000005fb17         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1558+8
000000000005fb1e         jmp        loc_5fafa
```

there are no surprises there, the const value is placed right inside the `eax` 32-bit register (not sure why we are using eax here and not `rax`, could it be because it's faster?), and then pushed into the stack using the `push` instruction, and then we are calling the `Precompiled____print_813` procedure to print it to the console. The thing to note is how the value of that integer was placed in the stack (as opposed to the heap) and then printed immediately.

# `const double`

given the following Dart code:

```dart
import 'dart:io' show exit;

const doubleConst = 1.2;
void main(List<String> args) {
  print(doubleConst);
  exit(0);
}
```

we'll get the following AOT compiled code:

```asm
                     Precompiled____main_1558:
000000000005fac8         push       rbp                                         ; CODE XREF=Precompiled____main_main_1559+17
000000000005fac9         mov        rbp, rsp
000000000005facc         cmp        rsp, qword [r14+0x40]
000000000005fad0         jbe        loc_5faec

                     loc_5fad6:
000000000005fad6         call       Precompiled____print_813                    ; Precompiled____print_813, CODE XREF=Precompiled____main_1558+43
000000000005fadb         call       Precompiled____exit_1070                    ; Precompiled____exit_1070
000000000005fae0         mov        rax, qword [r14+0xc8]
000000000005fae7         mov        rsp, rbp
000000000005faea         pop        rbp
000000000005faeb         ret
                        ; endp

                     loc_5faec:
000000000005faec         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1558+8
000000000005faf3         jmp        loc_5fad6
```

this code is very different from the `const int` variant, since you see nowhere in this code where the value of `1.2` is moved to any register of any sort. Instead, Dart has created a new function called `Precompiled____print_813` and is calling that function instead. Let's go deep into that function to see what's going on:

```asm
                     Precompiled____print_813:
0000000000037458         push       rbp                                         ; CODE XREF=Precompiled____main_1558+14
0000000000037459         mov        rbp, rsp
000000000003745c         cmp        rsp, qword [r14+0x40]
0000000000037460         jbe        loc_37488

                     loc_37466:
0000000000037466         mov        r11, qword [r15+0x20af]                     ; CODE XREF=Precompiled____print_813+55
000000000003746d         push       r11
000000000003746f         call       Precompiled__Double_0150898_toString_1175   ; Precompiled__Double_0150898_toString_1175
0000000000037474         pop        rcx
0000000000037475         push       rax
0000000000037476         call       Precompiled____printToConsole_146           ; Precompiled____printToConsole_146
000000000003747b         pop        rcx
000000000003747c         mov        rax, qword [r14+0xc8]
0000000000037483         mov        rsp, rbp
0000000000037486         pop        rbp
0000000000037487         ret
                        ; endp

                     loc_37488:
0000000000037488         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____print_813+8
000000000003748f         jmp        loc_37466
```

the first part of the code before the linebreak is the setting up of the local stack so I won't talk about that. the interesting part is the `loc_37466` label where the first instruction is the `mov` instruction. This is _most definitely_ moving the pointer to the `doubleConst` constat into the `r11` 64-bit register. I could be wrong about this but I think that's it. So Dart is not hardcoding the `1.2` value as a 64-bit floating point into a register, instead, it's loading it from an effective address into the `r11` register. I _think_ this _could_ be less efficient than putting the constant value of the `double` right into the stack but I could be wrong about this. If you know, let me know too!

then Dart is calling the `Precompiled__Double_0150898_toString_1175` function, supposedly to convert the double to a string instance. I won't go into the internals of that function but we get what it's doing. once that function is done, it puts its result (the pointer to the string) into the `rax` 64-bit register nad we are then pushing `rax` into the stack just before calling the `Precompiled____printToConsole_146` function which will read that value from its stack using the `rbp` and `rsp`. so to summarize, `const double` values are not really loaded into the stack in Dart the same way as `const int`. I don't know why! They _could_ be though, if they were just treated as a 64-bit value and placed right into a 64-bit register like `rax`! If you're a compiler engineer at Google or know why this is not the case, please do chime in.

Conclusions

- constant `int` are placed inside a register (not even in the stack) directly and then worked with
- constant `double` values

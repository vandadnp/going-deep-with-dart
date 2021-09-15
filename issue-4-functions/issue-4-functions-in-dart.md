# Functions in Dart

in this issue I want to explore how functions are compiled and are passed their arguments internally in Dart. I'm going to compile the code into x86_64 AOT with `dart compile` and then see the instructions generated for the Dart code.

## Global functions without 0 arguments

given the following Dart code with a loose function with 0 arguments:

```dart
import 'dart:io' show exit;

void foo() {
  print(0xDEADBEEF);
}

void main(List<String> args) async {
  foo();
  exit(0);
}
```

we will get the following AOT:

```asm
                     Precompiled____foo_1430:
000000000009a62c         push       rbp                                         ; CODE XREF=Precompiled____main__async_op_1429+72
000000000009a62d         mov        rbp, rsp
000000000009a630         mov        eax, 0xdeadbeef
000000000009a635         cmp        rsp, qword [r14+0x40]
000000000009a639         jbe        loc_9a652

                     loc_9a63f:
000000000009a63f         push       rax                                         ; CODE XREF=Precompiled____foo_1430+45
000000000009a640         call       Precompiled____print_911                    ; Precompiled____print_911
000000000009a645         pop        rcx
000000000009a646         mov        rax, qword [r14+0xc8]
000000000009a64d         mov        rsp, rbp
000000000009a650         pop        rbp
000000000009a651         ret
                        ; endp

                     loc_9a652:
000000000009a652         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____foo_1430+13
000000000009a659         jmp        loc_9a63f
```

let's dig into this beauty and see what happened! the first part of the code is a so-called StackOverflowCheck. I had no idea about this. just to be clear, this is the code I'm talking about:

```asm
                     Precompiled____foo_1430:
000000000009a62c         push       rbp                                         ; CODE XREF=Precompiled____main__async_op_1429+72
000000000009a62d         mov        rbp, rsp
000000000009a630         mov        eax, 0xdeadbeef
000000000009a635         cmp        rsp, qword [r14+0x40]
000000000009a639         jbe        loc_9a652
```

minus the `mov eax 0xdeadbeef`, that's just the value to printed later, but the rest is a stackoverflow check. I asked on Twitter about this and got the following response from Vyacheslav Egorov:

> it's called StackOverflowCheck - it serves two main purposes: 1) checking for stack overflow 2) checking for internal interrupts scheduled by VM (e.g. might be used by GC to pause the execution on a safepoint).


# Recursive Functions in Dart

I know a lot of programmers, including myself, sometimes get confused by what a recursive function is and how it works internally and I want to shed some lights on this matter and see how Dart compiles recursive functions into AOT. I have done some OS programming many years ago (around year 2000) with NASM and created a 32-bit x86 OS and got it working with basic kernel functionalities and wrote some cmd apps for it and writing code in asm then really helped in understanding how recursion works at a low level so I hope this article helps with shedding light on recursion.

## What is a recursive function?

the best way to demonstrate what a recursive function is, is through an example. given the following Dart code:

```dart
import 'dart:io' show exit;

int incrementUntilValueIs100OrMore(int value) {
  if (value >= 100) {
    return value;
  } else {
    return incrementUntilValueIs100OrMore(value + 1);
  }
}

void main(List<String> args) {
  final value = incrementUntilValueIs100OrMore(0);
  print(value);
  exit(0);
}
```

we'll get the following AOT for the `incrementUntilValueIs100OrMore` function:

```asm
                     Precompiled____incrementUntilValueIs100OrMore_1436:
000000000009a738         push       rbp                                         ; CODE XREF=Precompiled____main_1435+17, Precompiled____incrementUntilValueIs100OrMore_1436+38
000000000009a739         mov        rbp, rsp
000000000009a73c         cmp        rsp, qword [r14+0x40]
000000000009a740         jbe        loc_9a769

                     loc_9a746:
000000000009a746         mov        rax, qword [rbp+arg_0]                      ; CODE XREF=Precompiled____incrementUntilValueIs100OrMore_1436+56
000000000009a74a         cmp        rax, 0x64
000000000009a74e         jl         loc_9a759

000000000009a754         mov        rsp, rbp
000000000009a757         pop        rbp
000000000009a758         ret
                        ; endp

                     loc_9a759:
000000000009a759         add        rax, 0x1                                    ; CODE XREF=Precompiled____incrementUntilValueIs100OrMore_1436+22
000000000009a75d         push       rax
000000000009a75e         call       Precompiled____incrementUntilValueIs100OrMore_1436 ; Precompiled____incrementUntilValueIs100OrMore_1436
000000000009a763         pop        rcx
000000000009a764         mov        rsp, rbp
000000000009a767         pop        rbp
000000000009a768         ret
                        ; endp

                     loc_9a769:
000000000009a769         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____incrementUntilValueIs100OrMore_1436+8
000000000009a770         jmp        loc_9a746
```

let's break it down

these two bits of code are related to each other and as pointed by Vyacheslav Egorov, they are there as overflow checks:

```asm
                     Precompiled____incrementUntilValueIs100OrMore_1436:
000000000009a738         push       rbp                                         ; CODE XREF=Precompiled____main_1435+17, Precompiled____incrementUntilValueIs100OrMore_1436+38
000000000009a739         mov        rbp, rsp
000000000009a73c         cmp        rsp, qword [r14+0x40]
000000000009a740         jbe        loc_9a769

...

                     loc_9a769:
000000000009a769         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____incrementUntilValueIs100OrMore_1436+8
000000000009a770         jmp        loc_9a746
```

so I won't go through them too much now since we've looked at them in issue-4 just briefly. the important part is the `jmp` that is a short jump to the `loc_9a746` tag where the actual procedure code is laid out. then we get to `loc_9a746` which starts like this:

```asm
                     loc_9a746:
000000000009a746         mov        rax, qword [rbp+arg_0]                      ; CODE XREF=Precompiled____incrementUntilValueIs100OrMore_1436+56
000000000009a74a         cmp        rax, 0x64
000000000009a74e         jl         loc_9a759
```

the `mov` instruction there is setting the 64-bit value of `rax` to the value of the `value` argument, that we passed to this function to start with. then you can see `cmp        rax, 0x64` where `0x64` is the base-16 value of 100, basically our if statement. the `cmp` is there to do the `if` basically and compare `rax` with 100. this is then followed by `jl` which is "Jump near if less (SF≠ OF).". this will jump to `loc_9a759` if `rax` is less than 100. in our code we said `if value >= 100` but Dart is translating this to `if value < 100` and then it jumps to `loc_9a759`. if that's not the case, in other words, if `rax` is greater than or equal to 100, then this happens:

```asm
000000000009a754         mov        rsp, rbp
000000000009a757         pop        rbp
000000000009a758         ret
                        ; endp
```

I asked Vyacheslav about the calling convention of Dart and he said "it's a custom one" and I can see a little indication that the callee stores the return value in `rax` in this case we are returning an `int` in our function so it seems like Dart is reserving `rax` for that return purpose. So keep that in mind!

however if `rax` is less than 100, then we jump short to `loc_9a759` which is this:

```asm
                     loc_9a759:
000000000009a759         add        rax, 0x1                                    ; CODE XREF=Precompiled____incrementUntilValueIs100OrMore_1436+22
000000000009a75d         push       rax
000000000009a75e         call       Precompiled____incrementUntilValueIs100OrMore_1436 ; Precompiled____incrementUntilValueIs100OrMore_1436
000000000009a763         pop        rcx
000000000009a764         mov        rsp, rbp
000000000009a767         pop        rbp
000000000009a768         ret
                        ; endp
```

as you can see, `rax` is getting incremented by 1 and then pushed into the stack, and Dart is calling the `Precompiled____incrementUntilValueIs100OrMore_1436` procedure again, while we are already in that procedure. in this case `Precompiled____incrementUntilValueIs100OrMore_1436` becomes both the caller and the callee.

back in the `main` function where we called our procedure from, you can see this:

```asm
                     Precompiled____main_1435:
000000000009a700         push       rbp                                         ; CODE XREF=Precompiled____main_main_1437+17
000000000009a701         mov        rbp, rsp
000000000009a704         xor        eax, eax
000000000009a706         cmp        rsp, qword [r14+0x40]
000000000009a70a         jbe        loc_9a72f

                     loc_9a710:
000000000009a710         push       rax                                         ; CODE XREF=Precompiled____main_1435+54
000000000009a711         call       Precompiled____incrementUntilValueIs100OrMore_1436 ; Precompiled____incrementUntilValueIs100OrMore_1436
000000000009a716         pop        rcx
000000000009a717         push       rax
000000000009a718         call       Precompiled____print_911                    ; Precompiled____print_911
000000000009a71d         pop        rcx
000000000009a71e         call       Precompiled____exit_1024                    ; Precompiled____exit_1024
000000000009a723         mov        rax, qword [r14+0xc8]
000000000009a72a         mov        rsp, rbp
000000000009a72d         pop        rbp
000000000009a72e         ret
                        ; endp

                     loc_9a72f:
000000000009a72f         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1435+10
000000000009a736         jmp        loc_9a710
```

with this part being the most interesting part to me:

```asm
000000000009a710         push       rax                                         ; CODE XREF=Precompiled____main_1435+54
000000000009a711         call       Precompiled____incrementUntilValueIs100OrMore_1436 ; Precompiled____incrementUntilValueIs100OrMore_1436
000000000009a716         pop        rcx
000000000009a717         push       rax
000000000009a718         call       Precompiled____print_911                    ; Precompiled____print_911
000000000009a71d         pop        rcx
```

seems like the `Precompiled____incrementUntilValueIs100OrMore_1436` procedure is keeping its return value in `rax` after it's returning. so seems like the callee returns its value in `rax` in Dart. and the `push` and `pop` is just to balance the stack since `value` went into the stack with `push rax` so the caller is responsible for balancing the calls on the stack and that's what the `pop rcx` seems to be doing there. so we can conclude that the `Precompiled____incrementUntilValueIs100OrMore_1436` procedure here is keeping its return value in `rax` all the while it is calling itself on and on until it gets to the `JL` instruction where it pops out of the whole routine!

## Conclusions

- conclusion 1

## References

- Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 1: Basic Architecture
- Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 2 (2A, 2B, 2C & 2D): Instruction Set Reference, A-Z

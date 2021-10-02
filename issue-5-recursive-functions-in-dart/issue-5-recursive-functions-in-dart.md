# Recursive Functions in Dart

Let's have a look at how recursion works at low-level by having a look at some Dart AOT code!

- [Recursive Functions in Dart](#recursive-functions-in-dart)
  - [What is a recursive function?](#what-is-a-recursive-function)
  - [Low-level anatomy of recursive functions](#low-level-anatomy-of-recursive-functions)
  - [Traditional factorial recursive function in Dart](#traditional-factorial-recursive-function-in-dart)
  - [Conclusions](#conclusions)
  - [References](#references)

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

to better understand how recursive functions actually work in Dart, you'd need to know how `call`, `ret`, stack pointers, base pointer, etc work at assembly level so let's dig into those things now.

## Low-level anatomy of recursive functions

let's first talk about stacks. to know about stack, you'd need to know about segments. a segment is usually a defined piece of a software that has a limit, usually maximum of 4 gigs of memory on modern hardware. then you'd have a pointer, let's say the stack pointer, that starts at the **top** of the segment. so let's say we have a stack that is 1 megabytes, 1024 bytes in other words. in normal conditions, the program would set the stack up for you, in this case Dart, and then you'd have a stack pointer, or SP (stack pointer), that is stored in `esp` under x86_32 and `rsp` in x86_64. the stack pointer's value in this case would be 1024, so it would point to the top of the stack.

when you pass a function to a procedure, the compiler would set up the stack using a calling convention. a calling convention is an agreed-upon *way of* calling other procedures. for instance, a calling convention might be that the first parameter to a function is passed into `rax`, the second into `rcx` and the rest are pushed into the stack as 64-bit pointers. something like that. i'm making this up but the idea stays corrected. Dart, as I've understood from Vyacheslav Egorov, has a custom calling convention meaning that it's not really documented in a central place and this is nothing strange. the calling convention has to make sense to those who write a specific compiler, otherwise you and I who use the compiler won't even notice how it is working under the hood but a good understanding of how the stack works will help you in understanding recursive functions.

to continue you'd also need to know about the base pointer, or `ebp` in 32-bit mode or `rbp` in 64-bit protected mode. usually what happens inside the creation of a stack frame for a procedure, depending on who or what has compiled the code, would look like this:

```asm
push    rbp
mov     rbp, rsp
sub     rsp, NNN
... now we have NNN bytes of space on the stack
leave
ret
```

when we enter this procedure, usually what happens is that the stack pointer points to the top of the stack for the current procedure, so any values above that pointer are usually the values that have been pushed to the stack. again depending on the calling convention, maybe arguments will get passed to gprs (general purpose registers) and from what Vyacheslav said, Dart has a custom calling convention so I cannot really document it here but we will assume that in our fictitious calling convention in this article, all arguments are passed in the stack! in that case when we enter our procedure shown before, the stack pointer would point to the last argument pushed to the stack. then you would use the base pointer to point to the memory the procedure needs to allocate for its local variables as well as the variables that the *caller* pushed to the stack and passed to us. if you see `rbp + NNN` it may mean that the procedure is reading arguments passed to it while if you see `rsp + NNN` after a `sub rsp, NNN` it *may* mean that the procedure is working with its local variables.

so to summarize, calling conventions define how a caller and callee communicate with each other while the base pointer and the stack pointer are used to coordinate the access to both arguments passed to a procedure and the locally stored stack space for the procedure.

knowing that, and understanding that upon a Dart function getting called, it will set up its stack using the stack pointer and the base pointer, and knowing that every value pushed to the stack decreases the stack pointer by the number of bytes needed for that variable to be stored in the stack, you'll understand that the stack can actually run out of space if you recursively call a function with no proper exit.

the thing to take away from all of this rant is that every time we enter a new procedure through `CALL` where that procedure is setting up its stack, the stack pointer is decremented since we start at the top of the stack segment, so if you continue calling procedures like in our case, recursively, without actually leaving the nested procedure calls, the stack pointer will get so low that the runtime will and should eventually throw a stack overflow, which I'm sure Dart does.

here is also a good little bit of information about the stack pointer from Intel:

> Items are placed on the stack using the PUSH instruction and removed from the stack using the POP instruction. When an item is pushed onto the stack, the processor decrements the ESP register, then writes the item at the new top of stack. When an item is popped off the stack, the processor reads the item from the top of stack, then incre- ments the ESP register. In this manner, the stack grows down in memory (towards lesser addresses) when items are pushed on the stack and shrinks up (towards greater addresses) when the items are popped from the stack.

## Traditional factorial recursive function in Dart

i can't talk about recursive functions without paying tribute to the classical factorial function implementation that uses recursion so let's have a look at that. factorial of N is the *product* of all numbers from and including 1 up to and including N, so the factorial of 6 is `1*2*3*4*5*6 = 720`. a non-recursive way of calculating factorial of N would be like this:

```dart
import 'dart:io' show exit;

int factorial(int value) {
  var result = 1;
  for (var count = 1; count <= value; count++) {
    result *= count;
  }
  return result;
}

void main(List<String> args) {
  print(factorial(6));
  exit(0);
}
```

and this is quite straightforward but not what I want to demonstrate in this issue. let's have a look at how this would look like if we used recursion:

```dart
import 'dart:io' show exit;

int factorial(int value) => value == 1 
  ? value 
  : value * factorial(value - 1);

void main(List<String> args) {
  print(factorial(4));
  exit(0);
}
```

let's check this function's compiled AOT and see how that looks like:

```asm
        ; ================ B E G I N N I N G   O F   P R O C E D U R E ================

        ; Variables:
        ;    arg_0: int, 16


                     Precompiled____factorial_1436:
000000000009a73c         push       rbp                                         ; CODE XREF=Precompiled____main_1435+20, Precompiled____factorial_1436+65
000000000009a73d         mov        rbp, rsp
000000000009a740         cmp        rsp, qword [r14+0x40]
000000000009a744         jbe        loc_9a793

                     loc_9a74a:
000000000009a74a         mov        rcx, qword [rbp+arg_0]                      ; CODE XREF=Precompiled____factorial_1436+94
000000000009a74e         mov        rax, rcx
000000000009a751         add        rax, rax
000000000009a754         jno        loc_9a763

000000000009a75a         call       Precompiled_Stub__iso_stub_AllocateMintSharedWithoutFPURegsStub ; Precompiled_Stub__iso_stub_AllocateMintSharedWithoutFPURegsStub
000000000009a75f         mov        qword [rax+7], rcx

                     loc_9a763:
000000000009a763         cmp        rax, 0x2                                    ; CODE XREF=Precompiled____factorial_1436+24
000000000009a767         jne        loc_9a775

000000000009a76d         mov        rax, rcx
000000000009a770         jmp        loc_9a78e

                     loc_9a775:
000000000009a775         mov        rax, rcx                                    ; CODE XREF=Precompiled____factorial_1436+43
000000000009a778         sub        rax, 0x1
000000000009a77c         push       rax
000000000009a77d         call       Precompiled____factorial_1436               ; Precompiled____factorial_1436
000000000009a782         pop        rcx
000000000009a783         mov        rcx, qword [rbp+arg_0]
000000000009a787         imul       rcx, rax
000000000009a78b         mov        rax, rcx

                     loc_9a78e:
000000000009a78e         mov        rsp, rbp                                    ; CODE XREF=Precompiled____factorial_1436+52
000000000009a791         pop        rbp
000000000009a792         ret
                        ; endp

                     loc_9a793:
000000000009a793         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____factorial_1436+8
```

the first part of the procedure is just setting up the stack:

```asm
                     Precompiled____factorial_1436:
000000000009a73c         push       rbp                                         ; CODE XREF=Precompiled____main_1435+20, Precompiled____factorial_1436+65
000000000009a73d         mov        rbp, rsp
000000000009a740         cmp        rsp, qword [r14+0x40]
000000000009a744         jbe        loc_9a793
```

so i won't go through it again! after that we are going to `loc_9a74a` which looks like this:

```asm
                     loc_9a74a:
000000000009a74a         mov        rcx, qword [rbp+arg_0]                      ; CODE XREF=Precompiled____factorial_1436+94
000000000009a74e         mov        rax, rcx
000000000009a751         add        rax, rax
000000000009a754         jno        loc_9a763

000000000009a75a         call       Precompiled_Stub__iso_stub_AllocateMintSharedWithoutFPURegsStub ; Precompiled_Stub__iso_stub_AllocateMintSharedWithoutFPURegsStub
000000000009a75f         mov        qword [rax+7], rcx

                     loc_9a763:
000000000009a763         cmp        rax, 0x2                                    ; CODE XREF=Precompiled____factorial_1436+24
000000000009a767         jne        loc_9a775

000000000009a76d         mov        rax, rcx
000000000009a770         jmp        loc_9a78e
```

i truly got puzzled by all of this so I asked Vyacheslav on Twitter what all of this means and he answerd with this:

> it boxes an unboxed int64 value (either into a smi or a mint box if it does not fit into a smi) and then compares result with with a smi representation of 1 it's a bug - boxing should not be needed, will be fixed by https://t.co/PIKVq6kYFD

Since it seems to be a known bug that the compiler is doing additional work here than it should, and the work should essentially be replaced by doing calculations in a gpr, I will skip explaining this. but if you're interested in what mint representations and smi are in Dart, I suggest that you have a look at [this resource](https://dart.dev/articles/archive/numeric-computation).

we then get to the juciest part of the procedure, `loc_9a775`, which in my opinion is this:

```asm
                     loc_9a775:
000000000009a775         mov        rax, rcx                                    ; CODE XREF=Precompiled____factorial_1436+43
000000000009a778         sub        rax, 0x1
000000000009a77c         push       rax
000000000009a77d         call       Precompiled____factorial_1436               ; Precompiled____factorial_1436
000000000009a782         pop        rcx
000000000009a783         mov        rcx, qword [rbp+arg_0]
000000000009a787         imul       rcx, rax
000000000009a78b         mov        rax, rcx
```

so it looks like the current accumulated value is being placed inside `rbp+arg_0` and the current `value` is being processed by the first three instructions in `loc_9a775`. if you look at the original Dart code you may get confused by the ternary statement comparing `value` with 1 and then passing the `value - 1` (which by the way is the `sub rax, 0x01` instruction above) into itself, and you'd wonder how `value` can both be `1` and also can be the result of the calculation of the `factorial` function but one thing we need to keep in mind is that the calculation done inside the `factorial` function is being saved in the stack, and the result is being stored in the stack too, and when `value` then gets subtracted, it gets pushed as a new value and passed to the stack and then read from the stack by the new `factorial` procedure call. so we essentially have two representations of the `value`! one is being essentially used as a counter, and the other as an accumulator, perfect for `rcx` and `rax` (or vice versa) if you ask me!

## Conclusions

- recursive functions do indeed call themselves, and this comes with the overhead of settign up a new stack and unwinding the stack in every pass through the function. you may not incur the same execution cost if you write your functions using iterators!
- if the exit scenario for a recursive function is not set properly, you will get stack overflow since you will run out of stack space after N execution depending on what Dart is allocated the stack size for. Usually the stack can be up to 4GB long, but I'm unsure as how large the stack is allowed to grow in Dart. The only part-official information about this is provided in GitHub on the Dart SDK repo from Ivan Posva who wrote "Stack space for the main thread is not set by the VM, but by the OS on launch."

## References

- [Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 1: Basic Architecture](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-1-manual.pdf)
- [Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 2 (2A, 2B, 2C & 2D): Instruction Set Reference, A-Z](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-instruction-set-reference-manual-325383.pdf)
- [Dart: Numeric computation](https://dart.dev/articles/archive/numeric-computation)

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

let's break this down and see what's going on:

```asm
000000000009a6a0         push       rbp                                         ; CODE XREF=Precompiled____main_main_1435+17
000000000009a6a1         mov        rbp, rsp
000000000009a6a4         sub        rsp, 0x8
000000000009a6a8         cmp        rsp, qword [r14+0x40]
000000000009a6ac         jbe        loc_9a72a
```

with the following pseudo-code:

```asm
 if (rsp <= *(r14 + 0x40)) {
         (*(r14 + 0x240))();
 }
```
this all is setting up the stack for us so there is nothing fancy that we should dive into. the next part of the code is this cute little guy:

```asm
                     loc_9a6b2:
000000000009a6b2         mov        edx, 0xdeadbeef                             ; CODE XREF=Precompiled____main_1434+145
```

and this is moving the starting value of the `x` variable into the 32-bit register of `edx` since Dart realizes that the initial value of the `x` variable is indeed a constant so it places it inside a register immediately. I don't yet know why Dart doesn't use the 64-bit `rdx` register when it does these moves, but maybe the CPU does that internally!

the next part is this:

```asm
                     loc_9a6b7:
000000000009a6b7         mov        qword [rbp+var_8], rdx                      ; CODE XREF=Precompiled____main_1434+119
000000000009a6bb         cmp        rsp, qword [r14+0x40]
000000000009a6bf         jbe        loc_9a736
```

if you look at the original assembly code and look for the `loc_9a6b7` label you'll see that there is a `loc_9a6f7` label at the end of which there is a `jmp` instruction that jumps to `loc_9a6b7`. do you know what this means? This is a typical `do { ... } while (true);` statement. So you can say that `for` loops with indices in Dart are created internally using a `while` loop!

back to the assembly code, this was another way of writing the following pseudo-code:

```asm
var_8 = rdx;
if (rsp <= *(r14 + 0x40)) {
   (*(r14 + 0x240))();
}
```

remember the value of `0xdeadbeef` being stored in `edx`? well, `edx` are the lower 32-bits of the `rdx` register so by doing this `mov qword [rbp+var_8], rdx`, it seems like Dart is simply storing the initial value of our variable into the stack, since it is using `rbp` (64-bit base pointer). the `cmp` and the `jpm` instruction I cannot be sure right now as to their purpose but let's carry on and ignore those for now!

then we have the next chunk of code like this:

```asm
                     loc_9a6c5:
000000000009a6c5         mov        r11d, 0xfeedfeed                            ; CODE XREF=Precompiled____main_1434+157
000000000009a6cb         cmp        rdx, r11
000000000009a6ce         jge        loc_9a719
```

remember how Dart placed `0xdeadbeef` inside the `edx` register? well since that's the lower 32-bits of the `rdx` register Dart is now comparing that value to the `r11` register which is a quadword register (fancy way of saying 64-bits register) and its lower 32-bits value are inside `r11d` dword register. don't worry if this all sounds weird for now but know that Dart is storing both the initial and the upper-bound values of our loop variable in two CPU registers, `rdx` and `r11`!

the next interesting part of the code for us is this

```asm
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
```

especially the `mov        rax, qword [r14+0x60]` and the `push       rax` calls where the actual value of the `x` variable is being loaded to the 64-bit `rax` register and then pushed into the stack to later be used by the `Precompiled____printToConsole_149` function. So this all is quite normal but the exciting part is this:

```asm
000000000009a710         add        rax, 0x1
000000000009a714         mov        rdx, rax
000000000009a717         jmp        loc_9a6b7
```

recall how the `edx` is supposed to hold onto our `x` variable's current value. Well, there you have it. Dart is doing `add rax, 0x01` for the `x++` part and then it's moving the value to `rdx` which is the 64-bit value with `edx` being the lower 32-bit half, essentially just adding 1 to `x` and then continuing the code until it hits `loc_9a6c5` where the value of `rdx` will be compared to the upper-bounds of our `x` variable, stored in `r11` and if the value is greater than or equal to `0xfeedfeed` (`jge        loc_9a719`) then it jumps to `loc_9a719` which is the `exit(0)` function:

```asm
                     loc_9a719:
000000000009a719         call       Precompiled____exit_1023                    ; Precompiled____exit_1023, CODE XREF=Precompiled____main_1434+46
000000000009a71e         mov        rax, qword [r14+0xc8]
000000000009a725         mov        rsp, rbp
000000000009a728         pop        rbp
000000000009a729         ret
                        ; endp
```
here you can see the entire pseudo-code for this asm code:

```asm
int Precompiled____main_1434(int arg0, int arg1, int arg2, int arg3, int arg4, int arg5) {
   r9 = arg5;
   r8 = arg4;
   rcx = arg3;
   rsi = arg1;
   rdi = arg0;
   if (rsp <= *(r14 + 0x40)) {
         (*(r14 + 0x240))();
   }
   rdx = 0xffffffffdeadbeef;
   do {
      var_8 = rdx;
      if (rsp <= *(r14 + 0x40)) {
         (*(r14 + 0x240))();
      }
      if (rdx >= 0xfffffffffeedfeed) {
         break;
      }
      rax = rdx + rdx;
      if (OVERFLOW(rax)) {
         rax = Precompiled_Stub__iso_stub_AllocateMintSharedWithoutFPURegsStub(rdi, rsi, rdx, rcx, r8, r9, var_8, stack[-8], stack[0], stack[8], stack[16], stack[24], stack[32], stack[40], stack[48], stack[56], stack[64], stack[72], stack[80]);
         *(rax + 0x7) = rdx;
      }
      rcx = 0x35;
      if ((rax & 0x1) != 0x0) {
         rcx = *(int16_t *)(rax + 0x1) & 0xffff;
      }
      stack[-24] = (*(*(r14 + 0x60) + rcx * 0x8 + 0x58d8))();
      Precompiled____printToConsole_149(rdi, rsi, rdx, rcx, r8, r9, stack[-24]);
      rcx = stack[-24];
      rsp = ((rsp - 0x8) + 0x8 - 0x8) + 0x8;
      rdx = var_8 + 0x1;
   } while (true);
   Precompiled____exit_1023();
   rax = *(r14 + 0xc8);
   return rax;
}
```

## The curious case of the unoptimized empty `for` loop

for the given Dart code:

```dart
import 'dart:io' show exit;

void main(List<String> args) {
  for (var x = 0xDEADBEEF; x < 0xFEEDFEED; x++) {}
  exit(0);
}
```

we get the following AOT code 🤦🏻‍♂️:

```asm
                     Precompiled____main_1433:
000000000009a644         push       rbp                                         ; CODE XREF=Precompiled____main_main_1434+17
000000000009a645         mov        rbp, rsp
000000000009a648         cmp        rsp, qword [r14+0x40]
000000000009a64c         jbe        loc_9a687

                     loc_9a652:
000000000009a652         mov        eax, 0xdeadbeef                             ; CODE XREF=Precompiled____main_1433+74

                     loc_9a657:
000000000009a657         cmp        rsp, qword [r14+0x40]                       ; CODE XREF=Precompiled____main_1433+48
000000000009a65b         jbe        loc_9a690

                     loc_9a661:
000000000009a661         mov        r11d, 0xfeedfeed                            ; CODE XREF=Precompiled____main_1433+83
000000000009a667         cmp        rax, r11
000000000009a66a         jge        loc_9a676

000000000009a670         add        rax, 0x1
000000000009a674         jmp        loc_9a657

                     loc_9a676:
000000000009a676         call       Precompiled____exit_1015                    ; Precompiled____exit_1015, CODE XREF=Precompiled____main_1433+38
000000000009a67b         mov        rax, qword [r14+0xc8]
000000000009a682         mov        rsp, rbp
000000000009a685         pop        rbp
000000000009a686         ret
                        ; endp

                     loc_9a687:
000000000009a687         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1433+8
000000000009a68e         jmp        loc_9a652

                     loc_9a690:
000000000009a690         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1433+23
000000000009a697         jmp        loc_9a661
```

this is very similar, if not identical to the previous code we looked at, and the problem I have with this code is that the Dart compiler didn't understand that this was an empty loop, and really created the loop code for it! Is this a bug? It could be. If you're a person working on the Dart compiler maybe you could sort this out!

For us Dart developers though this means that if you have a `for` loop somewhere with a variable, make sure it does something 😂

## Non-entry `for` loops with `const` start/end values

for the given Dart code:

```dart
import 'dart:io' show exit;

void main(List<String> args) {
  for (var x = 0xDEADBEEF; x < 0xDEADBEEF; x++) {
    print(x);
  }
  exit(0);
}
```

we get the following AOT:

```asm
                     Precompiled____main_1433:
000000000009a644         push       rbp                                         ; CODE XREF=Precompiled____main_main_1434+17
000000000009a645         mov        rbp, rsp
000000000009a648         cmp        rsp, qword [r14+0x40]
000000000009a64c         jbe        loc_9a66d

                     loc_9a652:
000000000009a652         cmp        rsp, qword [r14+0x40]                       ; CODE XREF=Precompiled____main_1433+48
000000000009a656         jbe        loc_9a676

                     loc_9a65c:
000000000009a65c         call       Precompiled____exit_1022                    ; Precompiled____exit_1022, CODE XREF=Precompiled____main_1433+57
000000000009a661         mov        rax, qword [r14+0xc8]
000000000009a668         mov        rsp, rbp
000000000009a66b         pop        rbp
000000000009a66c         ret
                        ; endp

                     loc_9a66d:
000000000009a66d         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1433+8
000000000009a674         jmp        loc_9a652

                     loc_9a676:
000000000009a676         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1433+18
000000000009a67d         jmp        loc_9a65c
```

as you can see nowhere in this code you can find a reference to our magic numbers nor can you find a reference to the `print()` function so this is great to know that as long as the start and the end values of your `for` loops are known at compile-time (contants), and if the end value and your incremenets/decrements make it so that the loop can never produce any iterations, then Dart is able to optimize out the whole loop! Good to know!

## `for` loops over variable iterables

now let's imagine the following scenario that you want to iterate over a list of strings as shown here:

```dart
import 'dart:io' show exit;

void main(List<String> args) {
  for (final value in args) {
    print(value);
  }
  exit(0);
}
```

for this code, we will get a rather chunky AOT asm so I'm not going to dump the whole thing here but the jist of it is this part:

```asm
000000000009a6e6         mov        rax, qword [rbp+var_8]
000000000009a6ea         movzx      rcx, word [rax+1]
000000000009a6ef         push       rax
000000000009a6f0         mov        rax, qword [r14+0x60]
000000000009a6f4         call       qword [rax+rcx*8+0x60]
000000000009a6f8         pop        r11
000000000009a6fa         push       rax
000000000009a6fb         call       Precompiled____printToConsole_141           ; Precompiled____printToConsole_141
000000000009a700         pop        rcx
000000000009a701         mov        rax, qword [rbp+var_8]
000000000009a705         jmp        loc_9a6c5
```

the part to pay close attention to is here, which might actually help a lot of programmers to understand how indexing into arrays work:

```asm
000000000009a6f4         call       qword [rax+rcx*8+0x60]
```

check this out, this is just beautiful, it seems like `rax` holds the base address to a function that can access this particular memory address assigned to `args`! then `rcx` is holding the index (well done Dart compiler, that's exactly what `*cx` registers are for!) to the current item we are iterating over. so `rcx` starts at 0 for the first item in `args` so the result will be `base address + 0*8 + 0x60`, since every item in `args` has an 8 bytes (64 bits) pointer to it under x86_64, so once `rcx` is moved to the second item (index 1), we calculate `base address + 8 + 0x60` where `0x60` is most definitely the offset for where `args` is stored in the heap.

## `for` loop over `const` iterables

we can look at a simpler example now where the iterable is a compile-time constant:

```dart
import 'dart:io' show exit;

const values = [0xDEADBEEF, 0xFEEDFEED];

void main(List<String> args) {
  for (final value in values) {
    print(value);
  }
  exit(0);
}
```

we will get [the following AOT code](snippet-1.md). I've decided not to paste that whole thing here since it is too long. the first thing you will notice is how long the code is and that Dart seems to be using some threading functions such as `AllocateMintSharedWithoutFPURegs` which you can find references to in the [Dart's actual source code](https://github.com/dart-lang/sdk/blob/e995cb5f7cd67d39c1ee4bdbe95c8241db36725f/runtime/vm/thread.h). This is way above my head since I haven't had the time to dive deep into the Dart's compiler code itself but I can certainly see the generated AOT and relalize that it is **not** smart. it would have been much smarter for Dart to realize that there are only two values in this iterable and simply made a normal `for` loop into that.

let's now convert this `for` with iterable into a traditional `for` loop and see what the AOT is:

```dart
import 'dart:io' show exit;

const values = [0xDEADBEEF, 0xFEEDFEED];

void main(List<String> args) {
  for (var i = 0; i < values.length; i++) {
    print(values[i]);
  }
  exit(0);
}
```

this generates the following AOT:

```asm
                     Precompiled____main_1434:
000000000009a6a0         push       rbp                                         ; CODE XREF=Precompiled____main_main_1435+17
000000000009a6a1         mov        rbp, rsp
000000000009a6a4         sub        rsp, 0x8
000000000009a6a8         cmp        rsp, qword [r14+0x40]
000000000009a6ac         jbe        loc_9a71d

                     loc_9a6b2:
000000000009a6b2         xor        edx, edx                                    ; CODE XREF=Precompiled____main_1434+132

                     loc_9a6b4:
000000000009a6b4         mov        rax, qword [r15+0x1e07]                     ; CODE XREF=Precompiled____main_1434+106
000000000009a6bb         mov        qword [rbp+var_8], rdx
000000000009a6bf         cmp        rsp, qword [r14+0x40]
000000000009a6c3         jbe        loc_9a726

                     loc_9a6c9:
000000000009a6c9         cmp        rdx, 0x2                                    ; CODE XREF=Precompiled____main_1434+141
000000000009a6cd         jge        loc_9a70c

000000000009a6d3         mov        rcx, qword [rax+rdx*8+0x17]
000000000009a6d8         test       cl, 0x1
000000000009a6db         mov        ebx, 0x35
000000000009a6e0         je         loc_9a6e7

000000000009a6e2         movzx      rbx, word [rcx+1]

                     loc_9a6e7:
000000000009a6e7         push       rcx                                         ; CODE XREF=Precompiled____main_1434+64
000000000009a6e8         mov        rcx, rbx
000000000009a6eb         mov        rax, qword [r14+0x60]
000000000009a6ef         call       qword [rax+rcx*8+0x58d8]
000000000009a6f6         pop        r11
000000000009a6f8         push       rax
000000000009a6f9         call       Precompiled____printToConsole_149           ; Precompiled____printToConsole_149
000000000009a6fe         pop        rcx
000000000009a6ff         mov        rax, qword [rbp+var_8]
000000000009a703         add        rax, 0x1
000000000009a707         mov        rdx, rax
000000000009a70a         jmp        loc_9a6b4

                     loc_9a70c:
000000000009a70c         call       Precompiled____exit_1023                    ; Precompiled____exit_1023, CODE XREF=Precompiled____main_1434+45
000000000009a711         mov        rax, qword [r14+0xc8]
000000000009a718         mov        rsp, rbp
000000000009a71b         pop        rbp
000000000009a71c         ret
                        ; endp

                     loc_9a71d:
000000000009a71d         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1434+12
000000000009a724         jmp        loc_9a6b2

                     loc_9a726:
000000000009a726         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1434+35
```

it's sad to say that the traditional `for` variant in this case is actually more performant and without a doubt easier for the compiler to solve since there are no iterables to iterate through. note that the iteration is being done using the `for` with an index, so no iterables are being created unlike the previous example.

in this example we have a simple loop that is being incremented 1 value at a time (`0x01`) like this:

```asm
000000000009a6ff         mov        rax, qword [rbp+var_8]
000000000009a703         **add        rax, 0x1**
000000000009a707         mov        rdx, rax
000000000009a70a         jmp        loc_9a6b4
```

## Conclusions

- Dart's `for` loop with an index is internally a `do { ... } while (true);` statement under the hood!
- Dart keeps, if possible, both the initial and the upper/lower bound of a `for` loop inside CPU registers, speeding up calculations. 
- Dart doesn't seem to be able to optimize out empty `for` loops with indices! But hopefully you're not writing loops that don't do anything!
- Dart optimizes out non-entry loops as long as conditions are compile-time constants!
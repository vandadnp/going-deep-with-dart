# Prefer `const` over `final`

Let's discuss the optimizations that the Dart compiler applies to using constants over finals.

## What's the difference between `const` and `final`?

A `const` in Dart is a compile-time constant, meaning that all values that comprise the final value should be constants. For instance, the value `123` is a constant, but the value `123` read from the console into a variable of type `int` is **not** a constant since it's value is **not** known at compile-time.

A `final` value on the other hand cannot be assigned a new value after it has received its initial value. In Swift and Rust, this is similar to the `let` statement. A `final` variable's internals can change, but the variable cannot be overwritten by a new one.

## Diving into `const`

With the following Dart code:

```dart
import 'dart:io' show exit;

const value1 = 0xDEADBEEF;
const value2 = 0xFEEDFEED;

void main(List<String> args) {
  print(value1);
  print(value2);
  print(value1 + value2);
  exit(0);
}
```

this code compiles to the following x86_64 AOT:

```asm
        ; ================ B E G I N N I N G   O F   P R O C E D U R E ================


                     Precompiled____main_1558:
000000000005faec         push       rbp                                         ; CODE XREF=Precompiled____main_main_1559+17
000000000005faed         mov        rbp, rsp
000000000005faf0         cmp        rsp, qword [r14+0x40]
000000000005faf4         jbe        loc_5fb34

                     loc_5fafa:
000000000005fafa         mov        eax, 0xdeadbeef                             ; CODE XREF=Precompiled____main_1558+79
000000000005faff         push       rax
000000000005fb00         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb05         pop        rcx
000000000005fb06         mov        eax, 0xfeedfeed
000000000005fb0b         push       rax
000000000005fb0c         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb11         pop        rcx
000000000005fb12         movabs     rax, 0x1dd9bbddc
000000000005fb1c         push       rax
000000000005fb1d         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb22         pop        rcx
000000000005fb23         call       Precompiled____exit_1070                    ; Precompiled____exit_1070
000000000005fb28         mov        rax, qword [r14+0xc8]
000000000005fb2f         mov        rsp, rbp
000000000005fb32         pop        rbp
000000000005fb33         ret
                        ; endp
```

I won't focus on the `cmp` and the `jbe` parts where that's the compiler setting up the stack for the *main* function. We are interested in `loc_5fafa` in this case which is the body of our main function.

the following Dart code:

```dart
print(value1);
```

was then compiled into these x86_64 instructions:

```asm
000000000005fafa         mov        eax, 0xdeadbeef                             ; CODE XREF=Precompiled____main_1558+79
000000000005faff         push       rax
000000000005fb00         call       Precompiled____print_813                    ; Precompiled____print_813
```

first the compiler is moving the value of `0xdeadbeef` into the 64 bit `eax` register (this fills the upper-bits all with zero while the lower-bits get set to the aforementioned value) and then pushes that value into the stack. The call then happens to the `Precompiled____print_813` function where the function will set up its own stack and then pop the value of `eax` from the stack to use for printing so we won't jump into those details. I'm not sure about the `pop ecx` bit of the code but usually that means the result of the print statement is placed into the stack after it's done and is being retrieved by the `pop` instruction into the `ecx` register, it being 32 bits, instead of 64 otherwise it would be `pop rcx`!

The assembly code for this:

```dart
print(value2);
```

is the following, identical to the previous print statement where a `const` was involved:

```asm
000000000005fb06         mov        eax, 0xfeedfeed
000000000005fb0b         push       rax
000000000005fb0c         call       Precompiled____print_813                    ; Precompiled____print_813
```

so I won't explain this more since we've already seen the previous explanation!

then comes the plus operator:

```dart
print(value1 + value2);
```

which gets compiled into the following assembly code:

```asm
000000000005fb12         movabs     rax, 0x1dd9bbddc
000000000005fb1c         push       rax
000000000005fb1d         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb22         pop        rcx
```

the compiler simply added `0xdeadbeef` and `0xfeedfeed` and the result was `0x1dd9bbddc` which then is moved to the 64 bit `rax` register using `movabs` which I just learned is a GAS specific `mov` instruction so opcode-wise is the same as `mov`.

the take-away from this was the simplicity of the code and how compile-time constants get added at compile-time as well, so there is no `add` instruction to add the two values since a constant `mov` is faster in most modern cpus compared with an `add` instruction even if the two operands of the `add` are cpu registers!

## How about the `final` code?

So let's just make one small adjustment and turn `value2` into a `final` variable instead of a `const`:

```dart
import 'dart:io' show exit;

const value1 = 0xDEADBEEF;
final value2 = 0xFEEDFEED;

void main(List<String> args) {
  print(value1);
  print(value2);
  print(value1 + value2);
  exit(0);
}
```

the compiled code for this is almost painfully longer and more complicated. let's have a look:

```asm
000000000005faf4         push       rbp                                         ; CODE XREF=Precompiled____main_main_1560+17
000000000005faf5         mov        rbp, rsp
000000000005faf8         cmp        rsp, qword [r14+0x40]
000000000005fafc         jbe        loc_5fb79

                     loc_5fb02:
000000000005fb02         mov        eax, 0xdeadbeef                             ; CODE XREF=Precompiled____main_1559+140
000000000005fb07         push       rax
000000000005fb08         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb0d         pop        rcx
000000000005fb0e         mov        rax, qword [r14+0x88]
000000000005fb15         mov        rax, qword [rax+0x900]
000000000005fb1c         sar        rax, 0x1
000000000005fb1f         jae        loc_5fb29

000000000005fb21         mov        rax, qword [0x8+rax*2]

                     loc_5fb29:
000000000005fb29         push       rax                                         ; CODE XREF=Precompiled____main_1559+43
000000000005fb2a         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb2f         pop        rcx
000000000005fb30         mov        rax, qword [r14+0x88]
000000000005fb37         mov        rax, qword [rax+0x900]
000000000005fb3e         cmp        rax, qword [r14+0xc8]
000000000005fb45         je         loc_5fb82

000000000005fb4b         sar        rax, 0x1
000000000005fb4e         jae        loc_5fb58

000000000005fb50         mov        rax, qword [0x8+rax*2]

                     loc_5fb58:
000000000005fb58         mov        r11d, 0xdeadbeef                            ; CODE XREF=Precompiled____main_1559+90
000000000005fb5e         add        rax, r11
000000000005fb61         push       rax
000000000005fb62         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb67         pop        rcx
000000000005fb68         call       Precompiled____exit_1070                    ; Precompiled____exit_1070
000000000005fb6d         mov        rax, qword [r14+0xc8]
000000000005fb74         mov        rsp, rbp
000000000005fb77         pop        rbp
000000000005fb78         ret
                        ; endp

                     loc_5fb79:
000000000005fb79         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1559+8
000000000005fb80         jmp        loc_5fb02

                     loc_5fb82:
000000000005fb82         call       Precompiled_Stub__iso_stub_NullErrorSharedWithoutFPURegsStub ; Precompiled_Stub__iso_stub_NullErrorSharedWithoutFPURegsStub, CODE XREF=Precompiled____main_1559+81
000000000005fb87         int3
                        ; endp
```

jesus christ! that was a lot of code. I'm not going to go through it all since we've covered some of the basics and I try not to explain what all the instructios do since Intel has documented that already!

the code for printing `value1` is the exact same as it was before, since it still is a `const`:

```asm
000000000005fb02         mov        eax, 0xdeadbeef                             ; CODE XREF=Precompiled____main_1559+140
000000000005fb07         push       rax
000000000005fb08         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb0d         pop        rcx
```

how about the compiled code for this though?

```dart
print(value2);
```

well, that's where things go south! even though the value of `value2` is a final value and won't be re-assigned to, but Dart doesn't know that! everything in Dart is a class and so Dart treats them as so. In this case, what we are telling Dart is that we have an instance of the `int` class inside a `final` variable, whose value cannot be overwritten, but the `int` instance internally can change, so Dart has to accommodate this into its calculations, hence the code becomes much longer:

```asm
000000000005fb0e         mov        rax, qword [r14+0x88]
000000000005fb15         mov        rax, qword [rax+0x900]
000000000005fb1c         sar        rax, 0x1
000000000005fb1f         jae        loc_5fb29

000000000005fb21         mov        rax, qword [0x8+rax*2]

                     loc_5fb29:
000000000005fb29         push       rax                                         ; CODE XREF=Precompiled____main_1559+43
000000000005fb2a         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb2f         pop        rcx
```

the first two `mov` instructions are *most defintely* setting up the pointer to the `value2` pointer, I could be wrong about this, but I am assuming this since I don't know any better! If you know please let me know. then we have a `jae` which is pretty much the same as `jnc` which tests the Carry Flag (CF) in EFLAGS (refer to Intel's instructions for this!) since line before that is `sar` that stands for shift-arithmetic-right and the `jae` jumps to the print statement if the carry flag is 0. It's possible all of this is done to ensure `value2` is copied over to the stack before it is handed over to the `loc_5fb29` sub-procedure but I could be completely wrong about this. one thing that is clear though is that the code is definitely using `value2` as a constant, although it is not re-written at all!

the part that annoys me the most is the compiled code for this Dart code:

```dart
print(value1 + value2);
```

it translates to this code:

```asm
000000000005fb30         mov        rax, qword [r14+0x88]
000000000005fb37         mov        rax, qword [rax+0x900]
000000000005fb3e         cmp        rax, qword [r14+0xc8]
000000000005fb45         je         loc_5fb82

000000000005fb4b         sar        rax, 0x1
000000000005fb4e         jae        loc_5fb58

000000000005fb50         mov        rax, qword [0x8+rax*2]

                     loc_5fb58:
000000000005fb58         mov        r11d, 0xdeadbeef                            ; CODE XREF=Precompiled____main_1559+90
000000000005fb5e         add        rax, r11
000000000005fb61         push       rax
000000000005fb62         call       Precompiled____print_813                    ; Precompiled____print_813
```

you can see the same two `mov` instructions happening here again, and you can pretty much see Dart is repeating itself, and not keeping the value of `value2` in a register, since as I said before, Dart doesn't know better. It just knows that `int` is a class and between the previous `print()` function and now it's internals may have changed, so it has to redo the whole thing again, and bring `value2` into context!

this is really expensive if you have massive number of these hidden `final` values that could essentially be `const` but they just are not for some reason (most probably human error).

the part with the `value1` in this addition is the most straightforward as you can see

```asm
000000000005fb58         mov        r11d, 0xdeadbeef                            ; CODE XREF=Precompiled____main_1559+90
```

## Conclusion

Try to keep your constants as constants, and don't make the mistake of defining them as `final` values just because it's your team's convention or a similar reason. The Dart compiler treats your `final` values as potentially mutable class instances, which they are! So use `const` where you can and only use `final` if you cannot use `const`.

## References

* Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 1: Basic Architecture
* Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 2 (2A, 2B, 2C & 2D): Instruction Set Reference, A-Z
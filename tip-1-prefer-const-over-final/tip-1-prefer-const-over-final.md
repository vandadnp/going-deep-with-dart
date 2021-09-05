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
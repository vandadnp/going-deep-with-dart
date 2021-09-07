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

## `const String`

given the following Dart code:

```dart
import 'dart:io' show exit;

const stringConst = 'Hello, World!';
void main(List<String> args) {
  print(stringConst);
  exit(0);
}
```

we get the following AOT:

```asm
                     Precompiled____main_1558:
000000000005fab8         push       rbp                                         ; CODE XREF=Precompiled____main_main_1559+17
000000000005fab9         mov        rbp, rsp
000000000005fabc         cmp        rsp, qword [r14+0x40]
000000000005fac0         jbe        loc_5fadc

                     loc_5fac6:
000000000005fac6         call       Precompiled____print_813                    ; Precompiled____print_813, CODE XREF=Precompiled____main_1558+43
000000000005facb         call       Precompiled____exit_1070                    ; Precompiled____exit_1070
000000000005fad0         mov        rax, qword [r14+0xc8]
000000000005fad7         mov        rsp, rbp
000000000005fada         pop        rbp
000000000005fadb         ret
                        ; endp

                     loc_5fadc:
000000000005fadc         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1558+8
000000000005fae3         jmp        loc_5fac6
```

this is very similar to the `const double` AOT if you look closely, so let's go deep into the `Precompiled____print_813` function and see what's happening there:

```asm
000000000003745c         push       rbp                                         ; CODE XREF=Precompiled____main_1558+14
000000000003745d         mov        rbp, rsp
0000000000037460         cmp        rsp, qword [r14+0x40]
0000000000037464         jbe        loc_3747b

                     loc_3746a:
000000000003746a         call       Precompiled____printToConsole_146           ; Precompiled____printToConsole_146, CODE XREF=Precompiled____print_813+38
000000000003746f         mov        rax, qword [r14+0xc8]
0000000000037476         mov        rsp, rbp
0000000000037479         pop        rbp
000000000003747a         ret
                        ; endp

                     loc_3747b:
000000000003747b         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____print_813+8
0000000000037482         jmp        loc_3746a
```

okay that was unexpected! the main function called this function and all this function is doing is just calling another function called `Precompiled____printToConsole_146` and nowhere here call we find any reference to our string. I find this to be one layer too much, but I could be wrong here. let's see what's happening inside the `Precompiled____printToConsole_146` function:

```asm
000000000000f2ec         push       rbp                                         ; CODE XREF=Precompiled____print_813+14
000000000000f2ed         mov        rbp, rsp
000000000000f2f0         cmp        rsp, qword [r14+0x40]
000000000000f2f4         jbe        loc_f346

                     loc_f2fa:
000000000000f2fa         mov        rax, qword [r14+0x88]                       ; CODE XREF=Precompiled____printToConsole_146+97
000000000000f301         mov        rax, qword [rax+0x38]
000000000000f305         cmp        rax, qword [r15+0x27]
000000000000f309         jne        loc_f31b

000000000000f30f         mov        rax, qword [r15+0x1867]
000000000000f316         call       Precompiled_Stub__iso_stub_InitStaticFieldStub ; Precompiled_Stub__iso_stub_InitStaticFieldStub

                     loc_f31b:
000000000000f31b         mov        rcx, qword [rax+0x1f]                       ; CODE XREF=Precompiled____printToConsole_146+29
000000000000f31f         push       rax
000000000000f320         mov        r11, qword [r15+0x20af]
000000000000f327         push       r11
000000000000f329         mov        rax, rcx
000000000000f32c         mov        r10, qword [r15+0x7f]
000000000000f330         mov        rcx, qword [rax+0xf]
000000000000f334         call       rcx
000000000000f336         pop        r11
000000000000f338         pop        r11
000000000000f33a         mov        rax, qword [r14+0xc8]
000000000000f341         mov        rsp, rbp
000000000000f344         pop        rbp
000000000000f345         ret
                        ; endp

                     loc_f346:
000000000000f346         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____printToConsole_146+8
000000000000f34d         jmp        loc_f2fa
```

the part we're interested in is this:

```asm
                     loc_f31b:
000000000000f31b         mov        rcx, qword [rax+0x1f]                       ; CODE XREF=Precompiled____printToConsole_146+29
000000000000f31f         push       rax
000000000000f320         mov        r11, qword [r15+0x20af]
000000000000f327         push       r11
000000000000f329         mov        rax, rcx
000000000000f32c         mov        r10, qword [r15+0x7f]
000000000000f330         mov        rcx, qword [rax+0xf]
000000000000f334         call       rcx
```

it's interesting that in the other cases, the print function was being called with a call function and just a reference to the label (function name) but in this case the pointer to the print function is being loaded into the 64-bit register `ecx` (first line of code) and then called finally on the last line! the strange part is that `[rax+0x1f]` is literally a pointer to `000000000000f309 jne loc_f31b` which is jump short if not equal (if zero flag is 0, check EFLAGS in Intel references). this is way above my head and I don't really understand what `Precompiled_Stub__iso_stub_InitStaticFieldStub` does to be honest, but from the looks of it, it seems like it loads a given (through a push on the stack) static string from the data sector into the memory so that it can be used. But if you know better, please let me know too. Here is the data sector's placement of our string though if you can solve this mystery out yourself

```asm
0000000000084b30         db  0x48 ; 'H'
0000000000084b31         db  0x65 ; 'e'
0000000000084b32         db  0x6c ; 'l'
0000000000084b33         db  0x6c ; 'l'
0000000000084b34         db  0x6f ; 'o'
0000000000084b35         db  0x2c ; ','
0000000000084b36         db  0x20 ; ' '
0000000000084b37         db  0x57 ; 'W'
0000000000084b38         db  0x6f ; 'o'
0000000000084b39         db  0x72 ; 'r'
0000000000084b3a         db  0x6c ; 'l'
0000000000084b3b         db  0x64 ; 'd'
0000000000084b3c         db  0x21 ; '!'
```

with the data section header reading as follows:

```asm
        ; Segment Segment 6
        ; Range: [0x62000; 0xb2840[ (329792 bytes)
        ; File offset : [401408; 731200[ (329792 bytes)
        ; Permissions: readable
        ; Flags: 0x4



        ; Section .rodata
        ; Range: [0x62000; 0xb26f0[ (329456 bytes)
        ; File offset : [401408; 730864[ (329456 bytes)
        ; Flags: 0x2
        ;   SHT_PROGBITS
        ;   SHF_ALLOC

                     _kDartIsolateSnapshotData:
0000000000062000         db  0xf5 ; '.'
0000000000062001         db  0xf5 ; '.'
0000000000062002         db  0xdc ; '.'
```

## Curious case of mixing `const int` and `const double`

given the following Dart code:

```dart
import 'dart:io' show exit;

const intConst = 0xDEADBEEF;
const doubleConst = 1.2;
void main(List<String> args) {
  print(intConst);
  print(doubleConst);
  exit(0);
}
```

i would expect the `intConst` to be loaded into `eax` as it was before in the previous section where we talked about constant integers. but that's not the case! let's look at the AOT:

```asm
                     Precompiled____main_1558:
000000000005fad8         push       rbp                                         ; CODE XREF=Precompiled____main_main_1559+17
000000000005fad9         mov        rbp, rsp
000000000005fadc         cmp        rsp, qword [r14+0x40]
000000000005fae0         jbe        loc_5fb15

                     loc_5fae6:
000000000005fae6         mov        r11, qword [r15+0x207f]                     ; CODE XREF=Precompiled____main_1558+68
000000000005faed         push       r11
000000000005faef         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005faf4         pop        rcx
000000000005faf5         mov        r11, qword [r15+0x2087]
000000000005fafc         push       r11
000000000005fafe         call       Precompiled____print_813                    ; Precompiled____print_813
000000000005fb03         pop        rcx
000000000005fb04         call       Precompiled____exit_1070                    ; Precompiled____exit_1070
000000000005fb09         mov        rax, qword [r14+0xc8]
000000000005fb10         mov        rsp, rbp
000000000005fb13         pop        rbp
000000000005fb14         ret
                        ; endp

                     loc_5fb15:
000000000005fb15         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1558+8
000000000005fb1c         jmp        loc_5fae6
```

now all of a sudden the `intConst` is not placed in the `eax` register anymore, instead it is loaded like this:

```asm
000000000005fae6         mov        r11, qword [r15+0x207f]                     ; CODE XREF=Precompiled____main_1558+68
000000000005faed         push       r11
000000000005faef         call       Precompiled____print_813                    ; Precompiled____print_813
```

this is pretty much another way of saying:

```asm
stack[-16] = *(r15 + 0x207f);
Precompiled____print_813(rdi, rsi, rdx, rcx, r8, r9, stack[-16]);
```

so this way we are loading the pointer to the `intConst` into the stack and then calling the `Precompiled____print_813` function with that value placed in the stack. the `intConst` got demoted from a constant register value to a stack value for some reason. I think only a dart compiler engineer at Google can answer why this demotion happened to be honest. If you know the answer please let me know.

## `const` custom classes

let's have a look at a constant custom class:

```dart
import 'dart:io' show exit;

class Person {
  final int age;
  const Person(this.age);
}
void main(List<String> args) {
  final foo = Person(0xDEADBEEF);
  print(foo.age);
  exit(0);
}
```

that compiles to the following AOT:

```asm
000000000005faec         push       rbp                                         ; CODE XREF=Precompiled____main_main_1559+17
000000000005faed         mov        rbp, rsp
000000000005faf0         cmp        rsp, qword [r14+0x40]
000000000005faf4         jbe        loc_5fb17

                     loc_5fafa:
000000000005fafa         mov        eax, 0xdeadbeef                             ; CODE XREF=Precompiled____main_1558+50
000000000005faff         push       rax
000000000005fb00         call       Precompiled____print_812                    ; Precompiled____print_812
000000000005fb05         pop        rcx
000000000005fb06         call       Precompiled____exit_1066                    ; Precompiled____exit_1066
000000000005fb0b         mov        rax, qword [r14+0xc8]
000000000005fb12         mov        rsp, rbp
000000000005fb15         pop        rbp
000000000005fb16         ret
                        ; endp

                     loc_5fb17:
000000000005fb17         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1558+8
000000000005fb1e         jmp        loc_5fafa
```

this is great, and just as you'd expect it. the 32-bit value of `0xdeadbeef` that was assigned to the `age` of the `Person` instance is placed right into the `eax` 32-bit register and then printed to the screen. nice, no instance of the `Person` class was even created.

now let's make it more complicated and have some logic in the initializer of the `Person` class:

```dart
import 'dart:io' show exit;

class Person {
  final int age;
  const Person(int age) : this.age = age + 0xFEEDFEED;
}
void main(List<String> args) {
  final foo = Person(0xDEADBEEF);
  print(foo.age);
  exit(0);
}
```

and get the following AOT:

```asm

                     Precompiled____main_1558:
000000000005faec         push       rbp                                         ; CODE XREF=Precompiled____main_main_1559+17
000000000005faed         mov        rbp, rsp
000000000005faf0         cmp        rsp, qword [r14+0x40]
000000000005faf4         jbe        loc_5fb1c

                     loc_5fafa:
000000000005fafa         movabs     rax, 0x1dd9bbddc                            ; CODE XREF=Precompiled____main_1558+55
000000000005fb04         push       rax
000000000005fb05         call       Precompiled____print_812                    ; Precompiled____print_812
000000000005fb0a         pop        rcx
000000000005fb0b         call       Precompiled____exit_1066                    ; Precompiled____exit_1066
000000000005fb10         mov        rax, qword [r14+0xc8]
000000000005fb17         mov        rsp, rbp
000000000005fb1a         pop        rbp
000000000005fb1b         ret
                        ; endp

                     loc_5fb1c:
000000000005fb1c         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1558+8
000000000005fb23         jmp        loc_5fafa
```

this is also really neat. what Dart did here was that it took `0xdeadbeef` and found out that the constructor of the `Person` class is doing some calculation with that value, which in this case is to add `0xfeedfeed` to it and if you calculate that yourself you'll get `0x1DD9BBDDC` and it placed that value directly in the `rax` 32-bit register and passed it to the print statement. Neat huh?

let's make the `Person` class more complicated and exciting:

```dart
import 'dart:io' show exit;

class Person {
  final String firstName;
  final String lastName;
  final String fullName;

const Person(this.firstName, this.lastName) : fullName = '$firstName $lastName';
}
void main(List<String> args) {
  final foo = Person('Foo', 'Bar');
  print(foo);
  exit(0);
}
```

to get the following AOT:

```asm
                     Precompiled____main_1558:
000000000005fac0         push       rbp                                         ; CODE XREF=Precompiled____main_main_1560+17
000000000005fac1         mov        rbp, rsp
000000000005fac4         cmp        rsp, qword [r14+0x40]
000000000005fac8         jbe        loc_5faeb

                     loc_5face:
000000000005face         call       Precompiled_AllocationStub_Person_1559      ; Precompiled_AllocationStub_Person_1559, CODE XREF=Precompiled____main_1558+50
000000000005fad3         push       rax
000000000005fad4         call       Precompiled____print_812                    ; Precompiled____print_812
000000000005fad9         pop        rcx
000000000005fada         call       Precompiled____exit_1066                    ; Precompiled____exit_1066
000000000005fadf         mov        rax, qword [r14+0xc8]
000000000005fae6         mov        rsp, rbp
000000000005fae9         pop        rbp
000000000005faea         ret
                        ; endp

                     loc_5faeb:
000000000005faeb         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1558+8
000000000005faf2         jmp        loc_5face
```

oh, spicy! now we got some juice out of the example. you can see how the first line of the `loc_5face` label is making a call to the `Precompiled_AllocationStub_Person_1559` procedure. In this case, since our example got more complicated and involves two `String` final values to construct the eventual instance of the `Person` class, Dart doesn't seem to be able to optimize the implementation since `print(foo)` will internally call the `toString()` function of `Person` which we haven't overwritten, so it will have to make an instance of the `Person` class which is an `Object` and internally call that function to print out the `toString()` result. lots going on here but one thing at a time, let's look at `Precompiled_AllocationStub_Person_1559` and see what's going on there:

```asm
                     Precompiled_AllocationStub_Person_1559:
000000000005faf4         mov        r8d, 0xc70104                               ; CODE XREF=Precompiled____main_1558+14
000000000005fafa         jmp        qword [r14+0x228]
                        ; endp
```

which is another interpretation of the following pseudo-code:

```asm
void Precompiled_AllocationStub_Person_1559() {
    (*(r14 + 0x228))();
    return;
}
```

## Conclusion

- constant `int` are _sometimes_ placed inside a register (not even in the stack) directly and then worked with. as shown in this article `int` constants can be demoted to stack variables in some certain conditions and I don't really know the reason why!
- constant `double` values are loaded from memory (not placed directly inside a register, unlike constant `int` values) and then used
- const `String` instances are first loaded into the memory through 2 layers of function calls and then printed to the screen

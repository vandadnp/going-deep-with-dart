# Prefer `const` over `final`

Let's discuss the optimizations that the Dart compiler applies to using constants over finals.

## What's the difference between `const` and `final`?

A `const` in Dart is a compile-time constant, meaning that all values that comprise the final value should be constants. For instance, the value `123` is a constant, but the value `123` read from the console into a variable of type `int` is **not** a constant since it's value is **not** known at compile-time.

A `final` value on the other hand cannot be assigned a new value after it has received its initial value. In Swift and Rust, this is similar to the `let` statement. A `final` variable's internals can change, but the variable cannot be overwritten by a new one.

## Diving into `const`

With the following Dart code:

```dart
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

```
        ; ================ B E G I N N I N G   O F   P R O C E D U R E ================

        ; Variables:
        ;    arg_0: int, 16


                     Precompiled__GrowableList_0150898_toString_1173:
000000000004d0b8         push       rbp
000000000004d0b9         mov        rbp, rsp
000000000004d0bc         cmp        rsp, qword [r14+0x40]
000000000004d0c0         jbe        loc_4d0d4

                     loc_4d0c6:
000000000004d0c6         push       qword [rbp+arg_0]                           ; CODE XREF=Precompiled__GrowableList_0150898_toString_1173+35
000000000004d0c9         call       Precompiled_ListBase_listToString_228       ; Precompiled_ListBase_listToString_228
000000000004d0ce         pop        rcx
000000000004d0cf         mov        rsp, rbp
000000000004d0d2         pop        rbp
000000000004d0d3         ret
                        ; endp

                     loc_4d0d4:
000000000004d0d4         call       qword [r14+0x240]                           ; CODE XREF=Precompiled__GrowableList_0150898_toString_1173+8
000000000004d0db         jmp        loc_4d0c6
```
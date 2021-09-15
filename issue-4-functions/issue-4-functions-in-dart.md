# Functions in Dart

in this issue I want to explore how functions are compiled and are passed their arguments internally in Dart. I'm going to compile the code into x86_64 AOT with `dart compile` and then see the instructions generated for the Dart code.

## Global functions without 0 arguments

given the following Dart code with a loose function with 0 arguments:

```dart
import 'dart:io' show exit;

void foo() {
  print(0xDEADBEEF);
}

void main(List<String> args) {
  foo();
  exit(0);
}
```

we will get the following AOT:

```asm
                     Precompiled____foo_1436:
000000000009a730         push       rbp                                         ; CODE XREF=Precompiled____main_1435+14
000000000009a731         mov        rbp, rsp
000000000009a734         mov        eax, 0xdeadbeef
000000000009a739         cmp        rsp, qword [r14+0x40]
000000000009a73d         jbe        loc_9a756

                     loc_9a743:
000000000009a743         push       rax                                         ; CODE XREF=Precompiled____foo_1436+45
000000000009a744         call       Precompiled____print_911                    ; Precompiled____print_911
000000000009a749         pop        rcx
000000000009a74a         mov        rax, qword [r14+0xc8]
000000000009a751         mov        rsp, rbp
000000000009a754         pop        rbp
000000000009a755         ret
                        ; endp

                     loc_9a756:
000000000009a756         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____foo_1436+13
000000000009a75d         jmp        loc_9a743
```

let's dig into this beauty and see what happened! the first part of the code is a so-called StackOverflowCheck. I had no idea about this. just to be clear, this is the code I'm talking about:

```asm
                     Precompiled____foo_1436:
000000000009a730         push       rbp                                         ; CODE XREF=Precompiled____main_1435+14
000000000009a731         mov        rbp, rsp
000000000009a734         mov        eax, 0xdeadbeef
000000000009a739         cmp        rsp, qword [r14+0x40]
000000000009a73d         jbe        loc_9a756
```

minus the `mov eax 0xdeadbeef`, that's just the value to printed later, but the rest is a stackoverflow check. I asked on Twitter about this and got the following response from Vyacheslav Egorov:

> it's called StackOverflowCheck - it serves two main purposes: 1) checking for stack overflow 2) checking for internal interrupts scheduled by VM (e.g. might be used by GC to pause the execution on a safepoint).

so I cloned the Dart SDK repo on GitHub and found this:

```cpp
COMPILER_PASS(EliminateStackOverflowChecks, {
  if (!flow_graph->IsCompiledForOsr()) {
    CheckStackOverflowElimination::EliminateStackOverflow(flow_graph);
  }
});
```

in the `compiler_pass.cc` file. you can dig deeper into the `EliminateStackOverflow` function to see this:

```cpp
void CheckStackOverflowElimination::EliminateStackOverflow(FlowGraph* graph) {
  const bool should_remove_all = IsMarkedWithNoInterrupts(graph->function());

  CheckStackOverflowInstr* first_stack_overflow_instr = NULL;
  for (BlockIterator block_it = graph->reverse_postorder_iterator();
       !block_it.Done(); block_it.Advance()) {
    BlockEntryInstr* entry = block_it.Current();

    for (ForwardInstructionIterator it(entry); !it.Done(); it.Advance()) {
      Instruction* current = it.Current();

      if (CheckStackOverflowInstr* instr = current->AsCheckStackOverflow()) {
        if (should_remove_all) {
          it.RemoveCurrentFromGraph();
          continue;
        }

        if (first_stack_overflow_instr == NULL) {
          first_stack_overflow_instr = instr;
          ASSERT(!first_stack_overflow_instr->in_loop());
        }
        continue;
      }

      if (current->IsBranch()) {
        current = current->AsBranch()->comparison();
      }

      if (current->HasUnknownSideEffects()) {
        return;
      }
    }
  }

  if (first_stack_overflow_instr != NULL) {
    first_stack_overflow_instr->RemoveFromGraph();
  }
}
```

so what all of this is doing is, with excerpts from Vyacheslav, this is checking the current stack pointer against *a limit* to make sure it's not gone over that, so that a runtime routine can catch such stack overflows and handle that internally! so basically we don't have to worry about that part of the code, is what I'm trying to say 🤠

so until this point, before the `loc_9a63f` label, the 64-bit CPU register `eax` (the lower 32-bits of `rax`) is holding the value we are going to print to the console using the `print` function.

then when we do get to `loc_9a63f` label eventually, you can see this:

```asm
                     loc_9a63f:
000000000009a63f         push       rax                                         ; CODE XREF=Precompiled____foo_1430+45
000000000009a640         call       Precompiled____print_911                    ; Precompiled____print_911
000000000009a645         pop        rcx
000000000009a646         mov        rax, qword [r14+0xc8]
000000000009a64d         mov        rsp, rbp
000000000009a650         pop        rbp
000000000009a651         ret
                        ; endp
```

that is pushing the value of `0xdeadbeef` into the stack as a 32-bit register, which tells me `Precompiled____print_911` is taking in 32-bit pointers only!? I could be wrong about this but this basically pushes the stack pointer down by the length of `eax` which then the `Precompiled____print_911` function can use.

if you call this function multiple times like this:

```dart
import 'dart:io' show exit;

void foo() {
  print(0xDEADBEEF);
}

void main(List<String> args) {
  foo();
  foo();
  foo();
  exit(0);
}
```

the output AOT would be:

```asm
                     Precompiled____main_1435:
000000000009a700         push       rbp                                         ; CODE XREF=Precompiled____main_main_1437+17
000000000009a701         mov        rbp, rsp
000000000009a704         cmp        rsp, qword [r14+0x40]
000000000009a708         jbe        loc_9a72e

                     loc_9a70e:
000000000009a70e         call       Precompiled____foo_1436                     ; Precompiled____foo_1436, CODE XREF=Precompiled____main_1435+53
000000000009a713         call       Precompiled____foo_1436                     ; Precompiled____foo_1436
000000000009a718         call       Precompiled____foo_1436                     ; Precompiled____foo_1436
000000000009a71d         call       Precompiled____exit_1024                    ; Precompiled____exit_1024
000000000009a722         mov        rax, qword [r14+0xc8]
000000000009a729         mov        rsp, rbp
000000000009a72c         pop        rbp
000000000009a72d         ret
                        ; endp

                     loc_9a72e:
000000000009a72e         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1435+8
000000000009a735         jmp        loc_9a70e
```

you can see the compiler is literally calling that function 3 times in a row. I know for sure that the Dart compiler *can* in fact optimize simple functions so that instead of a function call being done here it would do it inline, or even at compile-time if the function works with constant values for instance, but in this case, the compiler has decided that the `foo` function is not optimizable such that it can be done inline. I'm not so sure about the internals at the Dart SDK level how optimizations are done to global functions like this but I am almost sure there is a reason behind this!

I think it's important to have a look at other global functions with 0 arguments of a different sort just to get a better understand of how Dart optimizations work under the hood because the above example is not a good representation of capabilities of the Dart compiler, I feel like!

given the following Dart code:

```dart
import 'dart:io' show exit;

int foo() => 0xDEADBEEF;

void main(List<String> args) {
  print(foo);
  exit(0);
}
```

we get the following AOT:

```asm
                     Precompiled____main_1435:
000000000009a6ec         push       rbp                                         ; CODE XREF=Precompiled____main_main_1437+17
000000000009a6ed         mov        rbp, rsp
000000000009a6f0         cmp        rsp, qword [r14+0x40]
000000000009a6f4         jbe        loc_9a71a

                     loc_9a6fa:
000000000009a6fa         mov        r11, qword [r15+0x1e07]                     ; CODE XREF=Precompiled____main_1435+53
000000000009a701         push       r11
000000000009a703         call       Precompiled____print_911                    ; Precompiled____print_911
000000000009a708         pop        rcx
000000000009a709         call       Precompiled____exit_1024                    ; Precompiled____exit_1024
000000000009a70e         mov        rax, qword [r14+0xc8]
000000000009a715         mov        rsp, rbp
000000000009a718         pop        rbp
000000000009a719         ret
                        ; endp

                     loc_9a71a:
000000000009a71a         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1435+8
000000000009a721         jmp        loc_9a6fa
```

well this was interesting! there is no mention of the `foo` function anywhere in this code. the interesting part of the code for me is this:

```asm
000000000009a6fa         mov        r11, qword [r15+0x1e07]                     ; CODE XREF=Precompiled____main_1435+53
000000000009a701         push       r11
000000000009a703         call       Precompiled____print_911                    ; Precompiled____print_911
```

I can see that the `r11` 64-bit GPR is getting set to `[r15+0x1e07]`. I was unsure at first about what `r15` actually is here so again, I asked Vyacheslav Egorov and in his words:

> R15 is reserved as "object pool" register in Dart calling conventions, it contains a pointer to a pool object through which generated code accesses different constant and auxiliary objects in the isolate group's GC managed heap.

I could trace what Vyacheslav said, to the `DecodeLoadObjectFromPoolOrThread` function in Dart SDK's source code, under the `instructions_x64.cc` file, as shown here:

```cpp
bool DecodeLoadObjectFromPoolOrThread(uword pc, const Code& code, Object* obj) {
  ASSERT(code.ContainsInstructionAt(pc));

  uint8_t* bytes = reinterpret_cast<uint8_t*>(pc);

  COMPILE_ASSERT(THR == R14);
  if ((bytes[0] == 0x49) || (bytes[0] == 0x4d)) {
    if ((bytes[1] == 0x8b) || (bytes[1] == 0x3b)) {   // movq, cmpq
      if ((bytes[2] & 0xc7) == (0x80 | (THR & 7))) {  // [r14+disp32]
        int32_t offset = LoadUnaligned(reinterpret_cast<int32_t*>(pc + 3));
        return Thread::ObjectAtOffset(offset, obj);
      }
      if ((bytes[2] & 0xc7) == (0x40 | (THR & 7))) {  // [r14+disp8]
        uint8_t offset = *reinterpret_cast<uint8_t*>(pc + 3);
        return Thread::ObjectAtOffset(offset, obj);
      }
    }
  }

  if (((bytes[0] == 0x41) && (bytes[1] == 0xff) && (bytes[2] == 0x76))) {
    // push [r14+disp8]
    uint8_t offset = *reinterpret_cast<uint8_t*>(pc + 3);
    return Thread::ObjectAtOffset(offset, obj);
  }

  COMPILE_ASSERT(PP == R15);
  if ((bytes[0] == 0x49) || (bytes[0] == 0x4d)) {
    if ((bytes[1] == 0x8b) || (bytes[1] == 0x3b)) {  // movq, cmpq
      if ((bytes[2] & 0xc7) == (0x80 | (PP & 7))) {  // [r15+disp32]
        intptr_t index = IndexFromPPLoadDisp32(pc + 3);
        const ObjectPool& pool = ObjectPool::Handle(code.GetObjectPool());
        if (!pool.IsNull() && (index < pool.Length()) &&
            (pool.TypeAt(index) == ObjectPool::EntryType::kTaggedObject)) {
          *obj = pool.ObjectAt(index);
          return true;
        }
      }
      if ((bytes[2] & 0xc7) == (0x40 | (PP & 7))) {  // [r15+disp8]
        intptr_t index = IndexFromPPLoadDisp8(pc + 3);
        const ObjectPool& pool = ObjectPool::Handle(code.GetObjectPool());
        if (!pool.IsNull() && (index < pool.Length()) &&
            (pool.TypeAt(index) == ObjectPool::EntryType::kTaggedObject)) {
          *obj = pool.ObjectAt(index);
          return true;
        }
      }
    }
  }

  return false;
}
```

the part which I think we need to look at is everything after this point:

```cpp
COMPILE_ASSERT(PP == R15);
...
```

what's interesting to me is this comment `[r15+disp32]` which tells me that the value that is added to the location of `r15` (object pool) is the effective displacement address of where the object resides in memory. looking at this `intptr_t index = IndexFromPPLoadDisp32(pc + 3);` i can see that the compiler is moving 3 bytes over `pc` which is set initially as the value of `bytes`, since the first 3 bytes inside the `bytes`/`pc` values are some magic numbers that only the compiler understands! I can't pretend like I know what this code is actually doing even though the team has left some comments on it, do you? 🤷🏻‍♂️

```cpp
if ((bytes[0] == 0x49) || (bytes[0] == 0x4d)) {
    if ((bytes[1] == 0x8b) || (bytes[1] == 0x3b)) {  // movq, cmpq
        if ((bytes[2] & 0xc7) == (0x80 | (PP & 7))) {  // [r15+disp32]
```

what I *do* know with almost certainty (again I could be wrong about this!) is that we will end up in this code block:

```cpp
intptr_t index = IndexFromPPLoadDisp32(pc + 3);
const ObjectPool& pool = ObjectPool::Handle(code.GetObjectPool());
if (!pool.IsNull() && (index < pool.Length()) &&
    (pool.TypeAt(index) == ObjectPool::EntryType::kTaggedObject)) {
    *obj = pool.ObjectAt(index);
    return true;
}
```

and Dart is getting an object pointer to our value returned from the `foo()` function using `IndexFromPPLoadDisp32` and then getting the pointer to the object pool using `code.GetObjectPool()` and then finally retrieving our object using `pool.ObjectAt(index)`. so this was interesting! even though the `foo()` function is returning a constant value (in our eyes a constant), the compiler is not understanding that and cannot promote the `0xdeadbeef` value to a constant. let's change that:

```dart
import 'dart:io' show exit;

const FOO = 0xDEADBEEF;

int foo() => FOO;

void main(List<String> args) {
  print(foo);
  exit(0);
}
```

for this code we get the following AOT:

```asm
                     Precompiled____main_1435:
000000000009a6ec         push       rbp                                         ; CODE XREF=Precompiled____main_main_1437+17
000000000009a6ed         mov        rbp, rsp
000000000009a6f0         cmp        rsp, qword [r14+0x40]
000000000009a6f4         jbe        loc_9a71a

                     loc_9a6fa:
000000000009a6fa         mov        r11, qword [r15+0x1e07]                     ; CODE XREF=Precompiled____main_1435+53
000000000009a701         push       r11
000000000009a703         call       Precompiled____print_911                    ; Precompiled____print_911
000000000009a708         pop        rcx
000000000009a709         call       Precompiled____exit_1024                    ; Precompiled____exit_1024
000000000009a70e         mov        rax, qword [r14+0xc8]
000000000009a715         mov        rsp, rbp
000000000009a718         pop        rbp
000000000009a719         ret
                        ; endp

                     loc_9a71a:
000000000009a71a         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1435+8
000000000009a721         jmp        loc_9a6fa
```

woopsy daisy, can't say I wasn't surprised! it seems like the Dart compiler didn't really understand that the only thing the `foo()` function is doing is to return a constant global value so it's trying to retrieve that value from the object pool anyways. to be honest I was expecting the `0xdeadbeef` value to be put right into a general purpose register at this point and then passed directly to the `r11` register to then be consumed by the `Precompiled____print_911` procedure but that wasn't the case!

digging deper if we change the code to the following:

```dart
import 'dart:io' show exit;

const FOO = 0xDEADBEEF;

int get foo => FOO;

void main(List<String> args) {
  print(foo);
  exit(0);
}
```

we will get this AOT:

```asm
                     Precompiled____main_1435:
000000000009a700         push       rbp                                         ; CODE XREF=Precompiled____main_main_1436+17
000000000009a701         mov        rbp, rsp
000000000009a704         mov        eax, 0xdeadbeef
000000000009a709         cmp        rsp, qword [r14+0x40]
000000000009a70d         jbe        loc_9a72b

                     loc_9a713:
000000000009a713         push       rax                                         ; CODE XREF=Precompiled____main_1435+50
000000000009a714         call       Precompiled____print_911                    ; Precompiled____print_911
000000000009a719         pop        rcx
000000000009a71a         call       Precompiled____exit_1024                    ; Precompiled____exit_1024
000000000009a71f         mov        rax, qword [r14+0xc8]
000000000009a726         mov        rsp, rbp
000000000009a729         pop        rbp
000000000009a72a         ret
                        ; endp

                     loc_9a72b:
000000000009a72b         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1435+13
000000000009a732         jmp        loc_9a713
```

so this is what I want to see from the compiler even with the `foo()` function, rather than it being a getter. in this code I created a `foo` getter that simply returns the constant value of `FOO` but when foo was a function, the Dart compiler couldn't optimize the whole function call out and substitute it with the `FOO` constant!

## Global functions with 1 compile-time constant argument

given the following Dart code:

```dart
import 'dart:io' show exit;

int increment(int value) {
  return value + 1;
}

void main(List<String> args) {
  print(increment(increment(increment(1))));
  exit(0);
}
```

we will get this AOT:

```asm
000000000009a700         push       rbp                                         ; CODE XREF=Precompiled____main_main_1436+17
000000000009a701         mov        rbp, rsp
000000000009a704         mov        eax, 0x4
000000000009a709         cmp        rsp, qword [r14+0x40]
000000000009a70d         jbe        loc_9a72b

                     loc_9a713:
000000000009a713         push       rax                                         ; CODE XREF=Precompiled____main_1435+50
000000000009a714         call       Precompiled____print_911                    ; Precompiled____print_911
000000000009a719         pop        rcx
000000000009a71a         call       Precompiled____exit_1024                    ; Precompiled____exit_1024
000000000009a71f         mov        rax, qword [r14+0xc8]
000000000009a726         mov        rsp, rbp
000000000009a729         pop        rbp
000000000009a72a         ret
                        ; endp

                     loc_9a72b:
000000000009a72b         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1435+13
000000000009a732         jmp        loc_9a713
```

you see that little `mov eax, 0x4` instruction up there? well that's the result to be printed to the screen using `Precompiled____print_911`. The Dart compiler just took the value of 0x01 that we passed to the inner function, calculated that the function just adds 1 to 0x01 so it becomes 0x02, and passed 0x02 to the function again to see it becomes 0x03 and finally 0x04 on the last pass; so it didn't even compile the `increment()` function and I can see inside the symbols i the resulting AOT that the `increment()` function is indeed not present at all in the binary.

## Global functions with 1 non-compile-time-constant argument

let's make it a bit harded for the compiler to optimize this function so let's take this as an example:

```dart
import 'dart:io' show exit;

int increment(int value) {
  return value + 1;
}

void main(List<String> args) {
  final number = int.tryParse(
        args.firstWhere(
          (_) => true,
          orElse: () => '',
        ),
      ) ??
      0;
  print(increment(increment(increment(number))));
  exit(0);
}
```

the `number` variable is being set to the first number passed to our program as an argument. this is just a way for me to make sure the Dart compiler cannot guess the value inside `number`, but it has no choice to compile that as a variable to be calculated at runtime and then passed to the `increment` function, let's check out the AOT for the main function now (note that I'm not going to paste the entire main function's AOT since it includes a **lot** of code for the `tryParse()` function and I don't see that as relevant to the point of this article so let's just look at the relevant part)

```asm
000000000009f8d5         push       rax
000000000009f8d6         call       Precompiled_int_tryParse_559                ; Precompiled_int_tryParse_559
000000000009f8db         pop        rcx
000000000009f8dc         cmp        rax, qword [r14+0xc8]
000000000009f8e3         jne        loc_9f8f0

000000000009f8e9         xor        eax, eax
000000000009f8eb         jmp        loc_9f8fd

                     loc_9f8f0:
000000000009f8f0         sar        rax, 0x1                                    ; CODE XREF=Precompiled____main_1440+111
000000000009f8f3         jae        loc_9f8fd

000000000009f8f5         mov        rax, qword [0x8+rax*2]

                     loc_9f8fd:
000000000009f8fd         add        rax, 0x1                                    ; CODE XREF=Precompiled____main_1440+119, Precompiled____main_1440+127
000000000009f901         add        rax, 0x1
000000000009f905         add        rax, 0x1
000000000009f909         push       rax
000000000009f90a         call       Precompiled____print_845                    ; Precompiled____print_845
```

let's break it down one bit at a time, it seems like the `cmp` and `jne` instruction (jump short if equal `ZF=0`, refer to EFLAGS in Intel instructions handbook) is checking the result of `Precompiled_int_tryParse_559` with `null` and if `ZF==0` (the result of `Precompiled_int_tryParse_559` was `null`), then it jumps to `loc_9f8f0`. but if the result was `null`, or in other words `ZF=0`, then it goes to this code:

```asm
000000000009f8e9         xor        eax, eax
000000000009f8eb         jmp        loc_9f8fd
```

which is a pretty clever way of saying that `eax` is 0 at this point. I don't know if this is an optimization on the Dart compiler's side, but it seems like Dart is setting `eax` to 0 using `xor` on x86_64 instead of saying `mov eax, 0`, and I remember from many years ago where I programmed in Assembly that indeed `xor` could be faster than `mov gpr, const` so it could very well be an optimization.

Now that `eax` is set to either 0 or the result of `tryParse()` we get to `loc_9f8fd` which is this code:

```asm
                     loc_9f8fd:
000000000009f8fd         add        rax, 0x1                                    ; CODE XREF=Precompiled____main_1440+119, Precompiled____main_1440+127
000000000009f901         add        rax, 0x1
000000000009f905         add        rax, 0x1
000000000009f909         push       rax
000000000009f90a         call       Precompiled____print_845                    ; Precompiled____print_845
```

I would be lying if I said I didn't chuckle but this doesn't seem like the best way to increment `rax` by 3 😂 it seems like the Dart compiler understood that `increment()` increments by 1, but it can't quite literally put together that calling this function N times should add N to `eax` so it's just repeating itself 3 times. Maybe this is a bug, what do I know! or maybe it's just such a difficult task to do on the compiler side to fix this that the Dart team doesn't think it's worth doing. I don't know! Do you?

## One-liner optimized `static` functions

given the following Dart code:

```dart
import 'dart:io' show exit;
import 'dart:math' show Random;

class Foo {
  static int increment(int value1, int value2) {
    return value1 + value2;
  }
}

void main(List<String> args) {
  final rnd = Random();
  final value1 = rnd.nextInt(0xDEADBEEF);
  final value2 = rnd.nextInt(0xCAFEBABE);
  final result = Foo.increment(value1, value2);
  print(result);
  exit(0);
}
```

we get the following AOT:

```asm
                     Precompiled____main_1436:
000000000009a8fc         push       rbp                                         ; CODE XREF=Precompiled____main_main_1437+17
000000000009a8fd         mov        rbp, rsp
000000000009a900         sub        rsp, 0x10
000000000009a904         cmp        rsp, qword [r14+0x40]
000000000009a908         jbe        loc_9a960

                     loc_9a90e:
000000000009a90e         push       qword [r14+0xc8]                            ; CODE XREF=Precompiled____main_1436+107
000000000009a915         call       Precompiled_Random_Random__1165             ; Precompiled_Random_Random__1165
000000000009a91a         pop        rcx
000000000009a91b         mov        qword [rbp+var_8], rax
000000000009a91f         push       rax
000000000009a920         mov        ecx, 0xdeadbeef
000000000009a925         push       rcx
000000000009a926         call       Precompiled__Random_11383281_nextInt_1164   ; Precompiled__Random_11383281_nextInt_1164
000000000009a92b         pop        rcx
000000000009a92c         pop        rcx
000000000009a92d         mov        qword [rbp+var_10], rax
000000000009a931         push       qword [rbp+var_8]
000000000009a934         mov        ecx, 0xcafebabe
000000000009a939         push       rcx
000000000009a93a         call       Precompiled__Random_11383281_nextInt_1164   ; Precompiled__Random_11383281_nextInt_1164
000000000009a93f         pop        rcx
000000000009a940         pop        rcx
000000000009a941         mov        rcx, qword [rbp+var_10]
000000000009a945         add        rcx, rax
000000000009a948         push       rcx
000000000009a949         call       Precompiled____print_911                    ; Precompiled____print_911
000000000009a94e         pop        rcx
000000000009a94f         call       Precompiled____exit_1024                    ; Precompiled____exit_1024
000000000009a954         mov        rax, qword [r14+0xc8]
000000000009a95b         mov        rsp, rbp
000000000009a95e         pop        rbp
000000000009a95f         ret
                        ; endp

                     loc_9a960:
000000000009a960         call       qword [r14+0x240]                           ; CODE XREF=Precompiled____main_1436+12
000000000009a967         jmp        loc_9a90e


        ; ================ B E G I N N I N G   O F   P R O C E D U R E ================


                     sub_9a969:
000000000009a969         int3
```

the first part of the code is again setting up the stack and checking for stack overflows as I explained before so I won't go into that again. The juicy part is inside the `loc_9a90e` label. that block of code starts with the following AOT:

```asm
000000000009a90e         push       qword [r14+0xc8]                            ; CODE XREF=Precompiled____main_1436+107
000000000009a915         call       Precompiled_Random_Random__1165             ; Precompiled_Random_Random__1165
000000000009a91a         pop        rcx
```

`r14` 64-bit GPR is being used here to point to the object pool for this thread and I believe internally in the Dart SDK it hits this point in the `instructions_x64.cc` file:

```cpp
bool DecodeLoadObjectFromPoolOrThread(uword pc, const Code& code, Object* obj) {
  ASSERT(code.ContainsInstructionAt(pc));

  uint8_t* bytes = reinterpret_cast<uint8_t*>(pc);

  COMPILE_ASSERT(THR == R14);
  if ((bytes[0] == 0x49) || (bytes[0] == 0x4d)) {
    if ((bytes[1] == 0x8b) || (bytes[1] == 0x3b)) {   // movq, cmpq
      if ((bytes[2] & 0xc7) == (0x80 | (THR & 7))) {  // [r14+disp32]
        int32_t offset = LoadUnaligned(reinterpret_cast<int32_t*>(pc + 3));
        return Thread::ObjectAtOffset(offset, obj);
      }
      if ((bytes[2] & 0xc7) == (0x40 | (THR & 7))) {  // [r14+disp8]
        uint8_t offset = *reinterpret_cast<uint8_t*>(pc + 3);
        return Thread::ObjectAtOffset(offset, obj);
      }
    }
  }
```

you see the `if ((bytes[2] & 0xc7) == (0x80 | (THR & 7))) {  // [r14+disp32]` code? that's the code responsible for loading the `Random` class into the stack using the `push` instruction and then `Precompiled_Random_Random__1165` call will initialize the Random instance for us. But let's not get side-tracked here, let's just focus on the static `increment()` function here. the AOT for that function is shown here:

```asm
000000000009a941         mov        rcx, qword [rbp+var_10]
000000000009a945         add        rcx, rax
000000000009a948         push       rcx
000000000009a949         call       Precompiled____print_911                    ; Precompiled____print_911
```

the `mov` instruction is simply loading the value in `[rbp+var_10]` into `rcx` and just so you know, `[rbp+var_10]` contains the result of `Precompiled__Random_11383281_nextInt_1164`, the first `nextInt()` call, and then Dart is doing `add rcx, rax` because `rax` is holding the result of the second call to the `nextInt()` function so the `add` instruction here is literally what we are doing inside the `increment()` function, brought into the lexical scope of the `main` function. this was quite a nice optimization by the Dart compiler so the static function got inlined in other words.

## More complex `static` functions

let's make the previous example a bit more complex for the compiler:

```dart
import 'dart:io' show exit;
import 'dart:math' show Random;

class Foo {
  static int increment(int value1, int value2) {
    print(value1);
    print(value2);
    return value1 + value2;
  }
}

void main(List<String> args) {
  final rnd = Random();
  final value1 = rnd.nextInt(0xDEADBEEF);
  final value2 = rnd.nextInt(0xCAFEBABE);
  final result = Foo.increment(value1, value2);
  print(result);
  exit(0);
}
```

and for this we get the following AOT for the main function, which I have reduced literally just to a few lines of asm code, since otherwise it's going to be repetitive:

```asm
...
000000000009a948         push       rax
000000000009a949         push       rcx
000000000009a94a         call       Precompiled_Foo_increment_1437              ; Precompiled_Foo_increment_1437
000000000009a94f         pop        rcx
000000000009a950         pop        rcx
...
```

## Conclusions

- some global functions with 0 arguments, even if a 1 liner, may not get optimized at compile time, rather they will become procedures at the asm level and then called using the `call` instruction in x86_64
- one liner getters returning a constant value tend to be optimized better by the Dart compiler, vs one-liner functions that return the same constant where the function variant stores its constant value in the object pool which then has to be retrieved by the CPU with more instructions!
- parameters passed to functions that cannot be optimized at compile-time to be inlined, are passed into the stack, using Dart's custom calling convention. I haven't been able to find a single place in the Dart SDK source code where the calling convention is documented!
- one-liner static functions, depending on their complexity, can, just like any other global function, be optimized as an inline function.

## References

- Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 1: Basic Architecture
- Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 2 (2A, 2B, 2C & 2D): Instruction Set Reference, A-Z

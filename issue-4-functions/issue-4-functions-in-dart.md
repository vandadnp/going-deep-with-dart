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

I could trace what Vyacheslav said, to the `DecodeLoadObjectFromPoolOrThread` function in Dart SDK's source code, under the `instructions_x86.cc` file, as shown here:

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

## Global functions with 1 or more arguments

given the following Dart code:

```dart

```

## Conclusions

- some global functions with 0 arguments, even if a 1 liner, may not get optimized at compile time, rather they will become procedures at the asm level and then called using the `call` instruction in x86_64
- one liner getters returning a constant value tend to be optimized better by the Dart compiler, vs one-liner functions that return the same constant where the function variant stores its constant value in the object pool which then has to be retrieved by the CPU with more instructions!

## References

- Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 1: Basic Architecture
- Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 2 (2A, 2B, 2C & 2D): Instruction Set Reference, A-Z

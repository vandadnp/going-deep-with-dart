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

## Conclusions

- some global functions with 0 arguments, even if a 1 liner, may not get optimized at compile time, rather they will become procedures at the asm level and then called using the `call` instruction in x86_64

## References

- Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 1: Basic Architecture
- Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 2 (2A, 2B, 2C & 2D): Instruction Set Reference, A-Z

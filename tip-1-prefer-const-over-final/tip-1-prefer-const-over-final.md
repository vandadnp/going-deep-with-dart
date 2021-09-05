# Prefer `const` over `final`

Let's discuss the optimizations that the Dart compiler applies to using constants over finals.

## What's the difference between `const` and `final`?

A `const` in Dart is a compile-time constant, meaning that all values that comprise the final value should be constants. For instance, the value `123` is a constant, but the value `123` read from the console into a variable of type `int` is **not** a constant since it's value is **not** known at compile-time.
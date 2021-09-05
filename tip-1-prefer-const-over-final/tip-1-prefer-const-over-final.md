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
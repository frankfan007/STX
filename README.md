<div align="center"><img src="assets/stx.png"/> </div>

<div align="center"><i> C++ 20 error-handling and utility extensions to the C++ standard library.</i>
</div>

## Overview

STX...

<div align="center"> <h3>
<a href="http://lamarrr.github.io/blog/error-handling">READ THE DOCUMENTATION</a></h3> 
</div>

* Panicking
* Result
* Option
* Manual Backtracing

- 

## Features

* Efficient `Result<T, E>` (error-handling) and `Option<T>` (optional-value) implementation with monadic methods
* Fatal failure reporting via Panicking
* Runtime panic hooks
* Panic backtraces
* Signal backtraces i.e. `SIGSEGV` , `SIGILL` , `SIGFPE` 
* Backtrace library
* Suitable and easily adoptable for embedded systems
* Easy debugging
* Easy to use, Hard to misuse
* Exception-free, RTTI-free, and malloc-free ( `no-std` )
* SFINAE-free
* Deterministic value lifetimes
* Eliminates repitive code and abstractable error-handling logic code via monadic extensions

## Basic Examples

### Option

``` cpp

#include <iostream>

#include "stx/option.h"

using stx::Option, stx::Some, stx::None;

auto safe_divide(double numerator, double denominator) -> Option<double> {
  if (denominator == 0.0) return None;
  return Some(numerator / denominator);
}

int main() {
  safe_divide(5.0, 2.0).match(
      [](auto value) { std::cout << "Result: " << value << std::endl; },
      []() { std::cout << "Cannot divide by zero" << std::endl; });
}

```

### Result

``` cpp

#include <array>
#include <iostream>

#include "stx/result.h"

using std::array, std::string_view;
using stx::Result, stx::Ok, stx::Err;
using namespace std::literals;

enum class Version { Version1 = 1, Version2 = 2 };

Result<Version, string_view> parse_version(array<uint8_t, 6> const& header) {
  switch (header.at(0)) {
    case 1:
      return Ok(Version::Version1);
    case 2:
      return Ok(Version::Version2);
    default:
      return Err("Unknown Version"sv);
  }
}

int main() {
  auto version =
      parse_version({2, 3, 4, 5, 6, 7}).unwrap_or("<Unknown Version>");

  std::cout << version << std::endl;
}

```

## Guidelines

* Some methods like `match` , `map` , `unwrap` and most of the `Result` , and `Option` methods **consume** the stored value and thus the `Result` or `Option` has to be destroyed as its lifetime has ended. For example:

	Say we define a function named `safe_divide` as in the example above, with the following prototype:

``` cpp
auto safe_divide(float n, float d) -> Option<float>;
```

And we call:

``` cpp
float result = safe_divide(n, d).unwrap(); // compiles, because safe_divide returns a temporary
```

``` cpp
Option option = safe_divide(n, d); 
float result = option.unwrap();  // will not compile, because unwrap consumes the value and is only usable with temporaries (as above) or r-value references (as below)
```

Alternatively, suppose the `Option` or `Result` is no longer needed, we can obtain an r-value reference:

``` cpp

Option option = safe_divide(n, d);
float result  = std::move(option).unwrap(); // will compile, the value is moved out of `option` , `option` should not be used any more

```

<span style="color:red">**NOTE**</span>: Just as any moved-from object, `Option` and `Result` are not to be used after a `std::move` ! (as the objects will be left in an unspecified state).

* `Option` and `Result` do not perform any implicit copies of the contained object as they are designed as purely forwarding types, this is especially due to their primary purpose as return channels in which we do not want duplication or implicit copies of the returned values. 

To make explicit copies:

``` cpp

Option option = safe_divide(n, d);
float result = option.clone().unwrap(); // note that `clone()` explicitly makes a copy of the `Option` 

```

We can also obtain an l-value reference to copy the value:

``` cpp

Option option = safe_divide(n, d);
float result = option.value(); // note that `value()` returns an l-value reference and `result` is copied from `option` 's value in the process

```

``` cpp

float result = safe_divide(n, d).value(); // this won't compile as `value` always returns an l-value reference, use `unwrap()` instead

```

## Build Requirements

* CMake
* Doxygen and Graphviz
* Make or Ninja Build
* C++ 20 Compiler

## Tested On

* Clang-10 + libstdc++-9
* Clang-11 + libstdc++-9

## CMake Configuration Options

* `STX_BUILD_SHARED` - Build STX as a shared library
* `STX_BUILD_TESTS` - Build test suite
* `STX_BUILD_DOCS` - Build documentation
* `STX_BUILD_BENCHMARKS` - Build benchmarks
* `STX_SANITIZE_TESTS` - Sanitize tests if supported. Builds address-sanitized, thread-sanitized, leak-sanitized, and undefined-sanitized tests
* `STX_OVERRIDE_PANIC_HANDLER` - Override the global panic handler
* `STX_DISABLE_BACKTRACE` - Disable the backtrace backend
* `STX_DISABLE_PANIC_BACKTRACE` - Disables panic backtraces. It depends on the backtrace backend.

## FAQ

* libstdc++, Concepts

## License

[**MIT License**](LICENSE)

# Duktape.cr

[![GitHub version](https://badge.fury.io/gh/jessedoyle%2Fduktape.cr.svg)](http://badge.fury.io/gh/jessedoyle%2Fduktape.cr)
[![Build Status](https://travis-ci.org/jessedoyle/duktape.cr.svg?branch=master)](https://travis-ci.org/jessedoyle/duktape.cr)

Duktape.cr provides Crystal bindings to the [Duktape](https://github.com/svaarala/duktape) javascript engine.

## Installation

Duktape.cr is best installed using [Shards](https://github.com/ysbaddaden/shards).

Add this to your `shard.yml`:

```yaml
name: example  # your project's name
version: 1.0.0 # your project's version

dependencies:
  duktape:
    github: jessedoyle/duktape.cr
    version: ~> 0.7.0
```

then execute:

```bash
shards install
```

Shards `v0.4.0` or greater will automatically make the native library. You can make the library manually by calling `make libduktape` from `libs/duktape/ext`.

## Usage

You must first create a Duktape context:

```crystal
require "duktape"

sbx = Duktape::Sandbox.new

sbx.eval! <<-JS
  var birthYear = 1990;

  function calcAge(birthYear){
    var current = new Date();
    var year = current.getFullYear();
    return year - birthYear;
  }

  print("You are " + calcAge(birthYear) + " years old.");
JS
```

An overwhelming majority of the [Duktape API](http://duktape.org/api.html) has been implemented. You can call the API functions directly on a `Duktape::Sandbox` or `Duktape::Context` instance:

```crystal
sbx = Duktape::Sandbox.new
sbx.push_global_object   # [ global ]
sbx.push_string "Math"   # [ global "Math" ]
sbx.get_prop -2          # [ global Math ]
sbx.push_string "PI"     # [ global Math "PI" ]
sbx.get_prop -2          # [ global Math PI ]
pi = sbx.get_number -1
puts "PI: #{pi}"         # => PI: 3.14159
sbx.pop_3
```

## Eval vs Eval!

All of the evaluation API methods have a corresponding bang-method (`!`). The bang method calls will raise when a javascript error occurs, the non-bang methods will not raise on invalid javascript.

For example:

```crystal
sbx = Duktape::Context.new
sbx.eval <<-JS
  var a =
JS
```

will not raise any errors, but will return a non-zero error code.

The following code:

```crystal
sbx = Duktape::Context.new
sbx.eval! <<-JS
  __invalid();
JS
```

will raise `Duktape::Error "SyntaxError"`.

## Sandbox vs Context

You should only execute untrusted javascript code from within a `Duktape::Sandbox` instance. A sandbox isolates code from insecure operations such as Duktape's internal `require` mechanism and the `Duktape` global javascript object.

Creating a `Duktape::Context` gives code access to internal Duktape properties:

```crystal
ctx = Duktape::Context.new
ctx.eval! <<-JS
  print(Duktape.version);
JS
```

## Setting a Timeout

`Duktape::Sandbox` instances may optionally take an execution timeout limit in milliseconds. This provides protection against infinite loops when executing untrusted code.

A `Duktape::Error "RangeError"` exception is raised when the following code executes for longer than specified:

```crystal
sbx = Duktape::Sandbox.new 500 # 500ms execution time limit
sbx.eval! "while (true) {}"    # => RangeError
```

## Duktape::Runtime

An alternative interface for evaluating JS code is available via the `Duktape::Runtime` class. This class provides a streamlined evaluation API (similar to ExecJS) that allows easier access to javascript values without the need to call many low-level Duktape API functions.

The entire `Runtime` API is as follows:

* `call(property, *args)` - Call the property or function with the given arguments and return the result.
* `call([properties], *args)` - Call the property that is nested within an array of string property names.
* `eval(source)` - Evaluate the javascript source and return the last value.
* `exec(source)` - Evaluate the javascript source and always return `nil`.

`Duktape::Runtime` instances can also be provided an initialization block when created.

Here's an example:

```crystal
  require "duktape/runtime"

  # A Runtime (optionally) accepts an initialization block
  rt = Duktape::Runtime.new do |sbx|
    sbx.eval! <<-JS
      function test(a, b, c) { return a + b + c; }
    JS
  end

  rt.call("test", 3, 4, 5) # => 12 (same as test(3, 4, 5);)
  rt.call(["Math", "PI"])  # => 3.14159
  rt.eval("1 + 1")         # => 2
  rt.exec("1 + 1")         # => nil
```

Note that `duktape/runtime` is not loaded by the base `duktape` require, and may be used standalone if necessary (ie. replace your `require "duktape"` calls with `require "duktape/runtime"` if you want this functionality).

## Calling Crystal Code from Javascript

Note: This functionality is considered experimental and syntax/functionality may change dramatically between releases.

It is possible to call Crystal code from your javascript:

```crystal
  sbx = Duktape::Sandbox.new

  sbx.push_global_object

  sbx.push_proc(2) do |ptr| # 2 stack arguments
    env = Duktape::Sandbox.new ptr
    a = env.get_number 0
    b = env.get_number 1
    env.push_number a + b
    env.return 1 # return success
  end

  sbx.put_prop_string -2, "adder"
  sbx.eval! "print(adder(2, 3));" # => 5
```

## Contributing

I'll accept any pull requests that are well tested for bugs/features with Duktape.cr.

You should fork the main repo, create a feature branch, write tests and submit a pull request.

## License

Duktape.cr is licensed under the MIT License. Please see `LICENSE` for details.

# JavaScript Sandbox

This page provides documentation about our proprietary JavaScript interpreter for safe, multi-tenant execution of untrusted JavaScript code in a sandboxed environment.

`LAST UPDATED: 17 Apr 2024`

## Features

- 100% pure Java library
  - can be embedded in any Java project

- multi-tenancy with full isolation of each script

- memory-efficient
  - can keep millions of script contexts in a single Java process

- easily customizable
  - configurable access to custom Java functions

- syntax compatibility with the EcmaScript 6 standard
  - for TypeScript or latest JavaScript support, see the section below

- supports the built-in JavaScript classes
  - objects, arrays, strings, dates, math etc.

## Security measures

- configurable limits per script execution:
  - configurable timeout for the duration of the execution of a script
  - configurable limit for the maximum ops to be executed by a script
  - configurable limit for the total allocated memory per script execution

- no built-in I/O access from scripts
  - no network access
  - no file system acccess
  - no OS/threads/console/JVM access

- no built-in access to Java reflection

- no JIT compilation
  - avoids a massive attack surface

- no support for advanced built-in classes and functions
  - e.g. `Buffer`, `Uint8Array`, `setTimeout` etc.
  - this is by design - to reduce the attack surface
  - NOTE: the standard classes like objects, arrays, strings, dates, math etc. are supported

- no `RegExp` support
  - this is by design - to prevent regex-based DoS attacks


## How to support TypeScript or latest JavaScript syntax

Our JavaScript Interpreter supports the syntax defined by the EcmaScript 6 standard.

In order to execute scripts written in TypeScript or later versions of JavaScript, we recommend transpiling the script's source code to ES6. For instance, this can be done on client side (in the browser), by using the TypeScript compiler or Babel.


## Examples:

### Simple expressions

This example illustrates a simple evaluation of JavaScript expressions:

```java
// Basic usage:
long result = (long) JSEngine.eval("20 + 30"); // returns 50
```

### Reusing a parsed script

Often, the same script might be needed to be executed multiple times. The `Abstract Syntax Tree` of a parsed script can be reused to avoid parsing it multiple times for multiple executions of the same script:

```java
// Parse the script:
ParsedScript script = JSEngine.parse("let x = 1; ++x");

// Execute the parsed script:
long result = (long) script.eval(); // returns 2

// Execute the parsed script again:
long result2 = (long) script.eval(); // returns 2 again
```

NOTE: Each invocation of `eval()` creates a new, isolated evaluation context and executes the script in it. Thus, multiple executions of the same script in different contexts are fully isolated.

### Add custom global constants

Besides the built-in global objects/classes like String, Math, etc. that are available in the global scope, additional custom values or objects can be added to the global scope:

```java
ParsedScript script = JSEngine.parse("X + Y");

// Set up custom globals:
Map<String, Object> customGlobals = Map.of(
        "X", 100,
        "Y", 200
);

// Execute the script, giving it context
long result = (long) script.eval(customGlobals); // returns 300
```

### Defining custom objects (without reflection)

In addition to custom constants, custom objects can be added to the global scope. The following examples illustrate the creation of custom objects with various numbers and types of parameters.

```java
CustomDefinitions definitions = def -> {
    Obj myObj = def.builder("myObj")
            .withMethod("hi", (EvalCtx ctx) -> ctx.make().str("Hello!"))
            .withMethod("next", (EvalCtx ctx, Val x) -> Vals.num(x.asLong() + 1))
            .withMethod("sum", (EvalCtx ctx, Val x, Val y) -> Vals.num(x.asLong() + y.asLong()))
            .withMethod("abc", (EvalCtx ctx, Val a, Val b, Val c) -> ctx.make().arr(List.of(a, b, c)))
            .build();

    return Map.of("myObj", myObj);
};

// Prints "Hello!"
System.out.println(JSEngine.eval("myObj.hi()", definitions));

// Prints "8"
System.out.println(JSEngine.eval("myObj.next(7)", definitions));

// Prints "5"
System.out.println(JSEngine.eval("myObj.sum(2, 3)", definitions));
```

### Defining custom objects with reflection

For increased security, the reflection calls are performed outside the script execution engine, in a high-level API wrapper which builds on top of the low-level API for custom object definitions.

The following example defines two custom objects: `console` and `utils`. Their methods are available to the script context and are being called by the sample JavaScript.

```java
static class Console {

    public static void log(String msg, Object... args) {
        System.out.println(msg + " " + Arrays.toString(args));
    }

    public static void info(String msg, Object... args) {
        System.out.println("INFO " + msg + " " + Arrays.toString(args));
    }

}

static class Utils {

    public static int sum(byte x, int y) {
        return x + y;
    }

    public static int len(Object... items) {
        return items.length;
    }

}

public static void main(String[] programArgs) {
    CustomDefinitions definitions = def -> {
        Obj console = def.builder("console")
                .withStaticMethodsFromClass(Console.class)
                .build();

        Obj utils = def.builder("utils")
                .withStaticMethodsFromClass(Utils.class)
                .build();

        return Map.of("console", console, "utils", utils);
    };

    ParsedScript script = JSEngine.parse("""
                            console.log('Hello', 'World', 123);
                            console.info('Foo', 'bar');
                            console.log('Length', utils.len(1, 2, 'f', true, [3]));
            """);

    script.eval(definitions);
}
```

### Defining a custom object with varargs method (without reflection)

This example illustrates the creation of custom object with `varargs` method:

```java
CustomDefinitions definitions = def -> {
    Obj myUtils = def.builder("myUtils")
            .withVarargsMethod("concat", (EvalCtx ctx, List<Val> args) -> {
                String concatenated = args.stream()
                        .map(Val::asStr)
                        .collect(Collectors.joining(""));

                return ctx.make().str(concatenated);
            })
            .build();


    return Map.of("myUtils", myUtils);
};

String script = "myUtils.concat('x', 1, 'y', 2, 'z')";

// Prints "x1y2z"
System.out.println(JSEngine.eval(script, definitions));
```

NOTE: For reflection-based method definitions please see the example  `Defining custom objects with reflection`.

### Add custom global object: console

This example defines a custom global object `console` with a method `log` can be configured like this:

```java
CustomDefinitions definitions = def -> {
    Obj console = def.builder("console")
            .withVarargsMethod("log", (EvalCtx ctx, List<Val> args) -> {
                String msg = args.stream()
                        .map(Val::asStr)
                        .collect(Collectors.joining(" "));

                System.out.println(msg);

                return Vals.UNDEFINED;
            })
            .build();

    return Map.of("console", console);
};

JSEngine.eval("console.log('Hello')", definitions); // prints 'Hello'
```

NOTE: For reflection-based object definitions please see the example  `Defining custom objects with reflection`.

### Add custom global object: a context with variables

This example illustrates the configuration of a custom global object `ctx` with methods `getVar` and `setVar`, for setting and accessing variables in a custom context:

```java
Map<String, Object> vars = new ConcurrentHashMap<>();
vars.put("firstName", "John");
vars.put("lastName", "Doe");

CustomDefinitions definitions = def -> {
    Obj ctx = def.builder("ctx")
            .withConstants(Map.of("x", 100, "y", 200))
            .withMethod("getVar", (EvalCtx evalCtx, Val name) -> {
                Object javaVal = vars.get(name.asStr());
                Optional<Val> jsVal = evalCtx.make().optional(javaVal);
                return jsVal.orElse(Vals.UNDEFINED);
            })
            .withMethod("setVar", (EvalCtx evalCtx, Val name, Val jsVal) -> {
                Object javaVal = jsVal.getValue();
                vars.put(name.asStr(), javaVal);
                return Vals.UNDEFINED;
            })
            .build();

    return Map.of("ctx", ctx);
};

// Sets a new variable "fullName":
String js = """
        const firstName = ctx.getVar('firstName').toUpperCase();
        const lastName = ctx.getVar('lastName').toUpperCase();
        ctx.setVar('fullName', firstName + ' ' + lastName);
        """;

JSEngine.eval(js, definitions);

// Returns "JOHN DOE"
Object fullName = vars.get("fullName");
```

NOTE: For reflection-based object definitions please see the example  `Defining custom objects with reflection`.

### Add custom global object with dynamic properties

This example illustrates the configuration of a custom global object `env` with custom, dynamic properties. The custom logic for accessing, updating, deleting and enumerating the properties is specified by implementing the `DynamicPropResolver` interface:

```java
Map<String, Object> vars = new ConcurrentHashMap<>();
vars.put("firstName", "John");
vars.put("lastName", "Doe");

CustomDefinitions definitions = def -> {
    Obj env = def.createDynamicObj("env", new DynamicPropResolver() {

        @Override
        public Optional<Val> getProp(EvalCtx ctx, String propName) {
            System.out.println("Reading property: " + propName);
            Object value = vars.get(propName);

            return ctx.make().optional(value);
        }

        @Override
        public boolean setProp(EvalCtx ctx, String propName, Val value) {
            System.out.println("Writing property: " + propName);
            Object javaVal = value.getValue();
            vars.put(propName, javaVal);

            return true; // `true` means the operation was successful
        }

        @Override
        public boolean deleteProp(EvalCtx ctx, String propName) {
            return false; // do not allow deleting props
        }

        @Override
        public Map<String, Val> getAllProps(EvalCtx ctx) {
            return vars.entrySet().stream()
                    .collect(To.map(Map.Entry::getKey, e -> ctx.make().from(e.getValue())));
        }
    });

    return Map.of("env", env);
};

// Sets a new variable "fullName":
String js = """
        const firstName = env.firstName.toUpperCase();
        const lastName = env.lastName.toUpperCase();
        env.fullName = firstName + ' ' + lastName;
                        
        Object.keys(env);
        """;

// Returns "JOHN DOE"
Object fullName = vars.get("fullName");

System.out.println("Full name: " + fullName);

// The result is the list of keys of the `env` object
List<String> keys = (List<String>) JSEngine.eval(js, definitions);

// Prints All keys: [firstName, lastName, fullName]
System.out.println("All keys: " + keys);
```

### Safety: execution limits

This example illustrates the securty measures for limiting the number of executed ops that can be executed for each script execution. For simpicity, this example uses the default execution limits, but they can be customized for each script.

```java
// Throws LimitsError: 'Reached the execution limit!'
JSEngine.eval("while (true) {}");
```

### Safety: memory limits

This example illustrates the securty measures for limiting the allocated memory for each script execution. For simpicity, this example uses the default memory limits, but they can be customized for each script.

```java
// Throws LimitsError: 'Reached the memory limit!'
JSEngine.eval("'x'.repeat(1000000)");
```

```java
// Throws LimitsError: 'Reached the memory limit!'
JSEngine.eval("""
        let arr = [];
        for (let i = 0; i < 1000; i++) {
            arr.push('x'.repeat(100000));
        }
        """);
```

```java
// Throws LimitsError: 'Reached the memory limit!'
JSEngine.eval("""
        let s = '';
        for (let i = 0; i < 1000; i++) {
            s += 'x'.repeat(100000);
        }
        """);
```

### Safety: custom limits for memory and ops

This example illustrates the securty measures for limiting the allocated memory and the number of executed ops for each script execution, with custom-defined limits for each script.

```java
ParsedScript script = JSEngine.parse("'x'.repeat(10000)");

// Throws LimitsError: 'Reached the memory limit!'
script.eval(EvalOpts.builder()
        .maxMemInBytes(9000)
        .maxOps(5000)
        .build());
```

### Getting statistics about a script execution

For each execution of a script, the statistics about the used memory and the number of executed ops can be retrieved. The statistics are retrieved only for the last execution of the script, together with the result.

```java
ParsedScript script = JSEngine.parse("x + 20");

// Execute the script and retrieve the results and execution info
ResultAndInfo resultAndInfo = script.evalAndGetDetails(EvalOpts.builder()
        .customGlobals(Map.of("x", 50))
        .build());

// get the result and convert it to long
long result = (long) resultAndInfo.getResult();

// get the stats about the execution of the script
ExecutionStats stats = resultAndInfo.getStats();

// prints the result (number 70)
System.out.println(result);

// prints the number of executed ops
System.out.println(stats.getOpsCount());

// prints the used memory in bytes
System.out.println(stats.getUsedMemInBytes());
```

### Handling syntax errors

This example illustrates the catching and handling of syntax errors:

```java
try {
    // Try to evaluate a script with syntax error:
    JSEngine.eval("123 +");

} catch (SyntaxError e) {
    // Prints the error message:
    // [line: 1, column: 5] Unexpected end of file
    System.out.println(e.getMessage());

    // Get the position, if needed:
    Position pos = e.getPosition();
    int line = pos.getLine();
    int column = pos.getColumn();
}
```

Here is the resulting console output:

```
[line: 1, column: 5] Unexpected end of file
```

### Handling runtime errors

This example illustrates the catching and handling of runtime errors:

```java
try {
    // Try to evaluate a script with runtime error:
    JSEngine.parse("""
                    function a(foo) {
                      foo.x = 1
                    }

                    function b(x) {
                      a(x);
                    }

                    b(null);
                    """.trim(),
                   ScriptInfo.filename("my-script.js"))
            .eval();

} catch (EvalError e) {
    // Prints the error message:
    System.out.println(e.getMessage());

    // The full stack trace of the JS script can be accessed:
    e.getCallStack().print();
}
```

Executing the script above will throw a runtime error. Here is the resulting console output:

```
Type NULL has no properties
foo.x = 1 (my-script.js:2)
a(x) (my-script.js:6)
b(null) (my-script.js:9)
```

## Benchmarks

The following micro-benchmarks have been performed on a laptop with Intel® Core™ i7 with 8 cores @ 1.30GHz and 16 GB RAM.

Each of the scripts was parsed once, and then it was iteratively executed many times (typically millions) on a pool of 8 threads.

The micro-benchmarks were performed without prior warm-up.

### Benchmark setup

The following setup was used:

```java
Obj myObj = JSObjects.builder("myObj")
        .withMethod("plaintext", () -> Vals.str("Some text"))
        .withMethod("next", (Val x) -> Vals.num(x.asLong() + 1))
        .withMethod("sum", (Val x, Val y) -> Vals.num(x.asLong() + y.asLong()))
        .build();

Map<String, Val> customGlobals = Map.of(
        "X", Vals.num(1),
        "Y", Vals.num(2),
        "S", Vals.str("Hello, world!"),
        "myObj", myObj
);

ParsedJS parsedJS = JSEngine.parse(js);

runBenchmark(() -> parsedJS.eval(customGlobals));
```

### Simple math expression

The following script was benchmarked:

```javascript
X + Y
```

Performance:
 - parsing: `1.6M executions/sec`
 - executing: `3.4M executions/sec`

### Simple string expression

The following script was benchmarked:

```javascript
`${X} ${S}`.toUpperCase()
```

Performance:
 - parsing: `784K executions/sec`
 - executing: `1.7M executions/sec`

### Sum of numbers (iteration)

The following script was benchmarked:

```javascript
function iter (n) {
  let sum = 0;

  for (let i = 0; i < 100; i++) {
    sum += i;
  }

  return sum;
}
```

Performance:
 - parsing: `173K executions/sec`
 - executing `iter(5)`: `748K executions/sec`
 - executing `iter(10)`: `595K executions/sec`
 - executing `iter(100)`: `147K executions/sec`
 - executing `iter(1000)`: `17K executions/sec`

### Sum of numbers (recursion)

The following script was benchmarked:

```javascript
function sum (n) {
  return n > 1 ? sum(n - 1) * n : 1
}
```

Performance:
 - parsing: `260K executions/sec`
 - executing `sum(5)`: `220K executions/sec`
 - executing `sum(10)`: `96K executions/sec`
 - executing `sum(100)`: `2K executions/sec`
 - executing `sum(200)`: `685 executions/sec`
 - executing `sum(300)`: `438 executions/sec`
 - executing `sum(400)`: throws an error: 'Reached the call stack limit!'

NOTE: The recursion performs much slower than iteration.

### Complex expression: map, join and string comprison

The following script was benchmarked:

```javascript
[].map.call([1,2,3], String).join(', ') === '1, 2, 3'
```

Performance:
 - parsing: `342K executions/sec`
 - executing: `676K executions/sec`

### Reading and writing dynamic object properties

The following script was benchmarked:

```javascript
const firstName = env.firstName.toUpperCase();
const lastName = env.lastName.toUpperCase();

env.fullName = firstName + ' ' + lastName;
```

Performance:
 - parsing: `284K executions/sec`
 - executing: `1.2M executions/sec`

### Calling custom methods

The following scripts are benchmarking the calling of the custom methods provided in the `myObj` custom object. Please see their source code in the `Benchmark setup` section above.

Performance:
 - executing `myObj.plaintext()`: `2.8M executions/sec`
 - executing `myObj.next(12345)`: `2.8M executions/sec`
 - executing `myObj.sum(40, 50)`: `2.7M executions/sec`

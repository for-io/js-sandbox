# JavaScript Sandbox

This page provides documentation about our proprietary JavaScript interpreter for safe, multi-tenant execution of untrusted JavaScript code in a sandboxed environment.

`LAST UPDATED: 8 Dec 2023`

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

- no support for advanced objects and functions
  - e.g. `setTimeout`, `Uint8Array`, `Buffer` etc.
  - this is by design - to reduce the attack surface

- no `RegExp` support
  - this is by design - to prevent regex-based DoS attacks


## How to support TypeScript or latest JavaScript syntax

Our JavaScript Interpreter supports the syntax defined by the EcmaScript 6 standard.

In order to execute scripts written in TypeScript or later versions of JavaScript, we recommend transpiling the script's source code to ES6. For instance, this can be done on client side (in the browser), by using the TypeScript compiler or Babel.


## Examples:

### Simple expressions

This example illustrates a simple evaluation of JavaScript expressions:

```java
// Returns 50 (wrapped in Val object)
Val result = JSEngine.eval("20 + 30");

// Returns 50
long num = result.asLong();
```

### Reusing a parsed script

Often, the same script might be needed to be executed multiple times. The `Abstract Syntax Tree` of a parsed script can be reused to avoid parsing it multiple times for multiple executions of the same script:

```java
// Parse the script:
ParsedJS parsedJs = JSEngine.parse("let x = 1; ++x");

// Execute the parsed script:
Val result = parsedJs.eval(); // returns 2

// Execute the parsed script again:
Val result2 = parsedJs.eval(); // returns 2 again
```

### Add custom global constants

Besides the built-in global objects/classes like String, Math, etc. that are available in the global scope, additional custom values or objects can be added to the global scope:

```java
ParsedJS script = JSEngine.parse("X + Y");

// Set up custom globals:
Map<String, Object> customGlobals = Map.of(
        "X", 100,
        "Y", 200
);

// Execute the script, giving it context
Val result = script.eval(customGlobals); // returns 300
```

### Defining custom objects

In addition to custom constants, custom objects can be added to the global scope. The following examples illustrate the creation of custom objects with various numbers and types of parameters.

```java
Obj myObj = JSObjects.builder("myObj")
        .withMethod("hi", () -> Vals.str("Hello!"))
        .withMethod("next", (Val x) -> Vals.num(x.asLong() + 1))
        .withMethod("sum", (Val x, Val y) -> Vals.num(x.asLong() + y.asLong()))
        .build();

Map<String, Obj> customGlobals = Map.of("myObj", myObj);

// Prints "Hello!"
System.out.println(JSEngine.eval("myObj.hi()", customGlobals));

// Prints "8"
System.out.println(JSEngine.eval("myObj.next(7)", customGlobals));

// Prints "5"
System.out.println(JSEngine.eval("myObj.sum(2, 3)", customGlobals));
```

### Defining custom objects with varargs methods

This example illustrates the creation of custom objects with `varargs` methods:

```java
Obj myUtils = JSObjects.builder("myUtils")
        .withVarargsMethod("concat", (List<Val> args) -> {
            String concatenated = args.stream()
                    .map(Val::asStr)
                    .collect(Collectors.joining(""));

            return Vals.str(concatenated);
        })
        .build();

Map<String, Obj> customGlobals = Map.of("myUtils", myUtils);

String script = "myUtils.concat('x', 1, 'y', 2, 'z')";
        
// Prints "x1y2z"
System.out.println(JSEngine.eval(script, customGlobals));
```

### Add custom global object: console

This example defines a custom global object `console` with a method `log` can be configured like this:

```java
Obj console = JSObjects.builder("console")
    .withVarargsMethod("log", (args) -> {
        System.out.println(args.stream()
                .map(Val::asStr)
                .collect(Collectors.joining(" "))
        );
        return Vals.UNDEFINED;
    })
    .build();

ParsedJS script = JSEngine.parse("console.log('Hello')");
script.eval(Map.of("console", console)); // prints 'Hello'
```

### Add custom global object: a context with variables

This example illustrates the configuration of a custom global object `ctx` with methods `getVar` and `setVar`, for setting and accessing variables in a custom context:

```java
Map<String, Object> vars = new ConcurrentHashMap<>();
vars.put("firstName", "John");
vars.put("lastName", "Doe");

Obj ctx = JSObjects.builder("ctx")
        .withConstants(Map.of("x", 100, "y", 200))
        .withMethod("getVar", (name) -> {
            Object javaVal = vars.get(name.asStr());
            Optional<Val> jsVal = Vals.optional(javaVal);
            return jsVal.orElse(Vals.UNDEFINED);
        })
        .withMethod("setVar", (name, jsVal) -> {
            Object javaVal = jsVal.getValue();
            vars.put(name.asStr(), javaVal);
            return Vals.UNDEFINED;
        })
        .build();

// This script sets a new variable "fullName" in the context:
String js = """
    const firstName = ctx.getVar('firstName').toUpperCase();
    const lastName = ctx.getVar('lastName').toUpperCase();

    ctx.setVar('fullName', firstName + ' ' + lastName);
""";

JSEngine.eval(js, Map.of("ctx", ctx));

Object fullName = vars.get("fullName"); // Returns "JOHN DOE"
```

### Add custom global object with dynamic properties

This example illustrates the configuration of a custom global object `env` with custom, dynamic properties. The custom logic for accessing, updating, deleting and enumerating the properties is specified by implementing the `DynamicPropResolver` interface:

```java
Map<String, Object> vars = new ConcurrentHashMap<>();
vars.put("firstName", "John");
vars.put("lastName", "Doe");

Obj env = JSObjects.createDynamicObj("env", new DynamicPropResolver() {

    @Override
    public Optional<Val> getProp(String propName) {
        System.out.println("Reading property: " + propName);
        Object value = vars.get(propName);

        return Vals.optional(value);
    }

    @Override
    public boolean setProp(String propName, Val value) {
        System.out.println("Writing property: " + propName);
        Object javaVal = value.getValue();
        vars.put(propName, javaVal);

        return true; // `true` means the operation was successful
    }

    @Override
    public boolean deleteProp(String propName) {
        return false; // do not allow deleting props
    }

    @Override
    public Map<String, Val> getAllProps() {
        return vars.entrySet().stream()
                .collect(To.map(Map.Entry::getKey, e -> Vals.from(e.getValue())));
    }
});

// Sets a new variable "fullName":
String js = """
        const firstName = env.firstName.toUpperCase();
        const lastName = env.lastName.toUpperCase();
        env.fullName = firstName + ' ' + lastName;
                        
        // Returns the names of the env properties:
        Object.keys(env);
          """;

Map<String, Obj> customGlobals = Map.of("env", env);
        
// The result is the list of keys of the `env` object
Val result = JSEngine.eval(js, customGlobals);

// Returns "JOHN DOE"
Object fullName = vars.get("fullName");

System.out.println("Full name: " + fullName);

List<String> keys = result.asList().stream()
        .map(Val::asStr)
        .toList();

System.out.println("All keys: " + keys);
```

The output is:

```
Reading property: firstName
Reading property: lastName
Writing property: fullName
Full name: JOHN DOE
All keys: [firstName, lastName, fullName]
```

### Safety: execution limits

This example illustrates the securty measures for limiting the number of executed ops that can be executed for each script execution. For simpicity, this example uses the default execution limits, but they can be customized for each script.

```java
// Throws an error: 'Reached the execution limit!'
JSEngine.eval("while (true) { }");
```

### Safety: memory limits

This example illustrates the securty measures for limiting the allocated memory for each script execution. For simpicity, this example uses the default memory limits, but they can be customized for each script.

```java
// Throws an error: 'Reached the memory limit!'
JSEngine.eval("'x'.repeat(1000000)");
```

```java
// Throws an error: 'Reached the memory limit!'
JSEngine.eval("""
  let arr = [];
  for (let i = 0; i < 1000; i++) {
      arr.push('x'.repeat(100000));
  }
""");
```

```java
// Throws an error: 'Reached the memory limit!'
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
ParsedJS parsedJS = JSEngine.parse("'x'.repeat(10000)");

// Throws an error: 'Reached the memory limit!'
parsedJS.eval(EvalOpts.builder()
        .maxMemInBytes(9000)
        .maxOps(5000)
        .build());
```

### Getting statistics about a script execution

For each execution of a script, the statistics about the used memory and the number of executed ops can be retrieved. The statistics are retrieved only for the last execution of the script, together with the result.

```java
ParsedJS script = JSEngine.parse("x + 20");

// Execute the script and retrieve the results and execution info
ResultAndInfo resultAndInfo = script.evalAndGetDetails(EvalOpts.builder()
        .customGlobals(Map.of("x", 50))
        .build());

// Get the result and convert it to long
long result = resultAndInfo.getResult().asLong();

// Get statistics about the last execution of the script
ExecutionStats stats = resultAndInfo.getStats();

// Print the result (number 70)
System.out.println(result);

// Print the number of executed ops
System.out.println(stats.getOpsCount());

// Print the used memory in bytes
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
    JSEngine.parse("my-script.js", """
        function a(foo) {
          foo.x = 1
        }

        function b(x) {
          a(x);
        }

        b(null);
        """.trim())
            .eval();

} catch (EvalError e) {
    // Prints the error message:
    System.out.println(e.getMessage());

    // The full stack trace of the JS script can be accessed:
    e.getCallStack().forEach(System.out::println);
}
```

Executing the script above will throw a runtime error. Here is the resulting console output:

```
Type NULL has no properties
foo.x = 1 (my-script.js:2)
a(x) (my-script.js:6)
b(null) (my-script.js:9)
```
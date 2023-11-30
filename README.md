# JavaScript Sandbox

This page provides documentation about our proprietary JavaScript interpreter for safe, multi-tenant execution of untrusted JavaScript code in a sandboxed environment.

## Features

- 100% pure Java library
  - can be embedded in any Java project

- multi-tenancy with full isolation of each script

- memory-efficient
  - can keep millions of script contexts in a single Java process

- easily customizable
  - configurable access to custom Java functions

- syntax compatibility with the ES6 standard

- supports the built-in JavaScript classes
  - objects, arrays, strings, dates, math etc.

## Security measures

- no JIT compilation
  - avoids a massive attack surface

- no support for advanced objects and functions
  - e.g. setTimeout, Uint8Array, Buffer etc.
  - this is by design - reduces the attack surface

- no RegExp support
  - this is by design - to prevent regex-based DoS attacks

- no built-in I/O access from scripts
  - no network access
  - no file system acccess
  - no OS/threads/console/JVM access

- no built-in access to Java reflection

- configurable limits per script execution:
  - configurable timeout for the duration of the execution of a script
  - configurable limit for the maximum ops to be executed by a script
  - configurable limit the total allocated memory per script execution


## Example 1: Simple expressions

This example illustrates a simple evaluation of JavaScript expressions:

```javascript
Val result = JSEngine.eval("20 + 30"); // returns 50 (wrapped in Val object)

long num = result.asLong(); // returns 50
```

## Example 2: Caching of a parsed script

The `Abstract Syntax Tree` of a parsed script can be cached, to avoid parsing it multiple times for multiple executions of the same script:

```javascript
// Parse the script:
ParsedJS parsedJs = JSEngine.parse("let x = 1; ++x");

// Execute the parsed script:
Val result = parsedJs.eval(); // returns 2

// Execute the parsed script again:
Val result2 = parsedJs.eval(); // returns 2 again
```

## Example 3: Add custom global constants

Besides the built-in global objects/classes like String, Math, etc. that are available in the global scope, additional custom values or objects can be added to the global scope:

```javascript
ParsedJS script = JSEngine.parse("X + Y");

// Set up custom globals:
Map<String, Object> customGlobals = Map.of(
        "X", 100,
        "Y", 200
);

// Execute the script, giving it context
Val result = script.eval(customGlobals); // returns 300
```

## Example 4: Add custom global object: console

Additionally, custom objects can be added to the global scope. For instance, creating a custom global object `console` with a method `log` can be configured like this:

```javascript
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

## Example 5: Add custom global object: a context with variables

This example illustrates the configuration of a custom global object `ctx` with methods `getVar` and `setVar`, for setting and accessing variables in a custom context:

```javascript
Map<Object, Object> vars = new ConcurrentHashMap<>();
vars.put("firstName", "John");
vars.put("lastName", "Doe");

Obj ctx = JSObjects.builder("ctx")
        .withConstants(Map.of("x", 100, "y", 200))
        .withMethod("getVar", (name) -> Vals.from(vars.get(name.asStr())))
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

JSEngine.parse(js)
        .eval(Map.of("ctx", ctx));

Object fullName = vars.get("fullName"); // Returns "JOHN DOE"
```

## Example 6: Safety - execution limits

This example illustrates the securty measures for limiting the number of operations that can be executed for each script execution. For simpicity, this example uses the default execution limits, but they can be customized for each script.

```javascript
// Throws an error: 'Reached the execution limit!'
JSEngine.eval("while (true) { }");
```

## Example 7: Safety - memory limits

This example illustrates the securty measures for limiting the allocated memory for each script execution. For simpicity, this example uses the default memory limits, but they can be customized for each script.

```javascript
// Throws an error: 'Reached the memory limit!'
JSEngine.eval("'x'.repeat(1000000)");

// Throws an error: 'Reached the memory limit!'
JSEngine.eval("let arr = []; for (let i = 0; i < 1000; i++) { arr.push('x'.repeat(100000)); }");

// Throws an error: 'Reached the memory limit!'
JSEngine.eval("let s = ''; for (let i = 0; i < 1000; i++) { s += 'x'.repeat(100000); }");
```

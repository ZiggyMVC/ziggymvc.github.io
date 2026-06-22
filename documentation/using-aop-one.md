---
layout: default
title: Using AOP/1 (Aspect-Oriented Programming)
---

# Using AOP/1 (Aspect-Oriented Programming)

AOP/1 extends DI/1 to provide aspect-oriented programming capabilities. It allows you to define interceptors that run before, after, or around method calls on your beans without modifying the bean code itself.

## What is Aspect-Oriented Programming?

AOP enables you to modularize cross-cutting concerns - functionality that affects multiple parts of an application, such as:

- Logging
- Security/Authorization
- Caching
- Transaction management
- Performance monitoring
- Error handling

## Getting Started

### Enabling AOP/1

Configure Ziggy MVC to use AOP/1 instead of DI/1:

```cfml
variables.framework = {
    diEngine = "aop1",
    diLocations = "model,controllers",
    diConfig = {
        interceptors = [
            // Interceptor definitions
        ]
    }
};
```

### Standalone Usage

```cfml
var beanFactory = new framework.aop("/model");

// Define interceptors
beanFactory.intercept("userService", "loggingInterceptor");
beanFactory.intercept("userService", "cacheInterceptor", "findById");

// Get the proxied bean
var userService = beanFactory.getBean("userService");
```

## Interceptor Types

AOP/1 supports four types of interceptors:

### Before Interceptors

Execute before the target method. Can modify arguments but not the return value.

```cfml
// model/interceptors/loggingInterceptor.cfc
component {
    function before(targetBean, methodName, args) {
        writeLog(
            file = "method-calls",
            text = "Calling #methodName# with args: #serializeJSON(args)#"
        );
        // Optionally modify args
        // arguments.args.someArg = "modified";
    }
}
```

### After Interceptors

Execute after the target method completes. Can modify the return value by returning a new value.

```cfml
// model/interceptors/cacheInterceptor.cfc
component {
    property cacheService;

    function after(targetBean, methodName, args, result) {
        // Cache the result
        var cacheKey = methodName & "_" & hash(serializeJSON(args));
        cacheService.put(cacheKey, result);

        // Return modified result (optional)
        return result;
    }
}
```

### Around Interceptors

Wrap the target method, controlling whether and how it executes. Must call `proceed()` to continue the chain.

```cfml
// model/interceptors/cachingInterceptor.cfc
component {
    property cacheService;

    function around(targetBean, methodName, args, proceed) {
        var cacheKey = methodName & "_" & hash(serializeJSON(args));

        // Check cache first
        if (cacheService.has(cacheKey)) {
            return cacheService.get(cacheKey);
        }

        // Call the actual method
        var result = proceed();

        // Cache the result
        cacheService.put(cacheKey, result);

        return result;
    }
}
```

### Error Interceptors

Handle exceptions thrown by the target method.

```cfml
// model/interceptors/errorInterceptor.cfc
component {
    property notificationService;

    function onError(targetBean, methodName, args, exception) {
        // Log the error
        writeLog(
            file = "errors",
            text = "Error in #methodName#: #exception.message#",
            type = "error"
        );

        // Notify administrators
        notificationService.alert("Method failure", exception);

        // Rethrow or return fallback value
        rethrow;
    }
}
```

## Configuring Interceptors

### Via intercept() Method

```cfml
var beanFactory = new framework.aop("/model");

// Intercept all methods
beanFactory.intercept("userService", "loggingInterceptor");

// Intercept specific methods
beanFactory.intercept("userService", "authInterceptor", "save,delete");

// Using regex for method matching
beanFactory.intercept("userService", "cacheInterceptor", "get*");
```

### Via Configuration

```cfml
diConfig = {
    interceptors = [
        {
            beanName = "userService",
            interceptorName = "loggingInterceptor"
            // All methods
        },
        {
            beanName = "userService",
            interceptorName = "cacheInterceptor",
            methods = "findById,findAll"
        },
        {
            beanName = ".*Service",  // Regex for bean names
            interceptorName = "securityInterceptor",
            methods = "save,update,delete"
        }
    ]
}
```

### Method Matching

The `methods` parameter accepts:
- Empty string or omitted: Intercept all methods
- Comma-separated list: `"save,update,delete"`
- Asterisk wildcard: `"*"` (all methods)
- Method name prefix: `"find*"` (requires regex bean matching)

## Interceptor Execution Order

When multiple interceptors are applied:

1. **Before** interceptors execute in definition order (like a queue)
2. **Around** interceptors execute in chain order (nested)
3. **After** interceptors execute in definition order
4. **Error** interceptors execute if an exception occurs

Example with multiple interceptors:

```
Request → Before1 → Before2 → Around1 → Around2 → Target Method
                                                        ↓
Response ← After2 ← After1 ← Around2 ← Around1 ← Method Return
```

## Combined Interceptors

A single interceptor can implement multiple types:

```cfml
// model/interceptors/transactionInterceptor.cfc
component {
    property datasource;

    function before(targetBean, methodName, args) {
        // Start transaction
        transaction action="begin";
    }

    function after(targetBean, methodName, args, result) {
        // Commit transaction
        transaction action="commit";
        return result;
    }

    function onError(targetBean, methodName, args, exception) {
        // Rollback transaction
        transaction action="rollback";
        rethrow;
    }
}
```

## Around Interceptor Details

### The proceed() Function

In around interceptors, `proceed()` invokes the next interceptor or the target method:

```cfml
function around(targetBean, methodName, args, proceed) {
    // Pre-processing
    var startTime = getTickCount();

    // Call next in chain (or target method)
    var result = proceed();

    // Post-processing
    var elapsed = getTickCount() - startTime;
    writeLog(file="perf", text="#methodName# took #elapsed#ms");

    return result;
}
```

### Skipping Execution

Don't call `proceed()` to skip the target method:

```cfml
function around(targetBean, methodName, args, proceed) {
    // Check permissions
    if (!hasPermission(methodName)) {
        throw(type="Unauthorized", message="Access denied");
        // proceed() never called
    }

    return proceed();
}
```

### Modifying Arguments

Pass modified arguments to `proceed()`:

```cfml
function around(targetBean, methodName, args, proceed) {
    // Sanitize input
    if (structKeyExists(args, "email")) {
        args.email = lcase(trim(args.email));
    }

    // Continue with modified args
    return proceed(args);
}
```

### Helper Methods

Around interceptors have access to:

- `isLast()` - Returns true if this is the last interceptor before the target
- `translateArgs(args, targetMethod)` - Converts positional to named arguments

## Common Use Cases

### Logging

```cfml
component {
    function before(targetBean, methodName, args) {
        writeLog(
            file = "audit",
            text = "User #session.userId ?: 'anonymous'# calling #methodName#"
        );
    }

    function after(targetBean, methodName, args, result) {
        writeLog(
            file = "audit",
            text = "#methodName# completed successfully"
        );
    }

    function onError(targetBean, methodName, args, exception) {
        writeLog(
            file = "audit",
            text = "#methodName# failed: #exception.message#",
            type = "error"
        );
        rethrow;
    }
}
```

### Caching

```cfml
component accessors="true" {
    property cacheService;

    function around(targetBean, methodName, args, proceed) {
        // Only cache read operations
        if (!reFindNoCase("^(get|find|load)", methodName)) {
            return proceed();
        }

        var cacheKey = createCacheKey(targetBean, methodName, args);

        // Return cached value if available
        if (cacheService.has(cacheKey)) {
            return cacheService.get(cacheKey);
        }

        // Execute and cache
        var result = proceed();
        cacheService.put(cacheKey, result, 3600); // 1 hour TTL

        return result;
    }

    private function createCacheKey(targetBean, methodName, args) {
        return hash(
            getMetadata(targetBean).name & "_" &
            methodName & "_" &
            serializeJSON(args)
        );
    }
}
```

### Authorization

```cfml
component accessors="true" {
    property securityService;

    function before(targetBean, methodName, args) {
        var requiredRole = getRequiredRole(targetBean, methodName);

        if (len(requiredRole) && !securityService.hasRole(requiredRole)) {
            throw(
                type = "AuthorizationException",
                message = "Access denied to #methodName#"
            );
        }
    }

    private function getRequiredRole(targetBean, methodName) {
        // Check metadata for @RequiresRole annotation
        var meta = getMetadata(targetBean);
        for (var func in meta.functions) {
            if (func.name == methodName && structKeyExists(func, "RequiresRole")) {
                return func.RequiresRole;
            }
        }
        return "";
    }
}
```

### Performance Monitoring

```cfml
component accessors="true" {
    property metricsService;

    function around(targetBean, methodName, args, proceed) {
        var startTime = getTickCount();
        var success = true;

        try {
            return proceed();
        } catch (any e) {
            success = false;
            rethrow;
        } finally {
            var elapsed = getTickCount() - startTime;
            metricsService.record(
                name = getMetadata(targetBean).name & "." & methodName,
                duration = elapsed,
                success = success
            );
        }
    }
}
```

### Retry Logic

```cfml
component {
    variables.maxRetries = 3;
    variables.retryDelay = 1000; // ms

    function around(targetBean, methodName, args, proceed) {
        var attempts = 0;
        var lastException = "";

        while (attempts < variables.maxRetries) {
            attempts++;
            try {
                return proceed();
            } catch (TransientError e) {
                lastException = e;
                writeLog(
                    file = "retry",
                    text = "#methodName# failed, attempt #attempts# of #maxRetries#"
                );
                if (attempts < variables.maxRetries) {
                    sleep(variables.retryDelay * attempts);
                }
            }
        }

        throw(
            type = "MaxRetriesExceeded",
            message = "Failed after #maxRetries# attempts",
            detail = lastException.message
        );
    }
}
```

## Best Practices

1. **Keep interceptors focused** - One interceptor, one concern
2. **Be mindful of performance** - Interceptors add overhead
3. **Specify methods explicitly** - Don't intercept everything
4. **Order matters** - Consider interceptor execution sequence
5. **Handle exceptions properly** - Rethrow or handle completely
6. **Use around sparingly** - Before/after are simpler when sufficient
7. **Test interceptors independently** - They're components too
8. **Document interceptor effects** - Others need to know what's intercepted

## Debugging

### Trace Interceptor Calls

```cfml
component {
    function before(targetBean, methodName, args) {
        writeLog(
            file = "interceptor-trace",
            text = "BEFORE: #getMetadata(targetBean).name#.#methodName#"
        );
    }

    function after(targetBean, methodName, args, result) {
        writeLog(
            file = "interceptor-trace",
            text = "AFTER: #getMetadata(targetBean).name#.#methodName#"
        );
    }
}
```

### Check Applied Interceptors

```cfml
var info = beanFactory.getBeanInfo("userService");
writeDump(info);
// Shows interceptors applied to the bean
```

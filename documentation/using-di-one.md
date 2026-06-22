---
layout: default
title: Using DI/1 (Dependency Injection)
---

# Using DI/1 (Dependency Injection)

DI/1 ("Inject One") is a lightweight, convention-based dependency injection framework bundled with Ziggy MVC. It automatically discovers, creates, and wires your application's components.

## What is Dependency Injection?

Dependency Injection (DI) is a design pattern where objects receive their dependencies from an external source rather than creating them internally. This promotes:

- **Loose coupling** - Components don't need to know how to create their dependencies
- **Testability** - Dependencies can be easily mocked for testing
- **Maintainability** - Changes to dependencies don't require changes to dependent code

## Core Concepts

### Beans

A **bean** is a CFC component managed by DI/1. DI/1 handles two types:

- **Singletons** - A single instance shared across the application
- **Transients** - A new instance created for each request

### Bean Factory

The **bean factory** creates and manages bean instances, handling their lifecycle and dependency wiring.

## Getting Started

### With Ziggy MVC

Ziggy MVC automatically creates a DI/1 bean factory. Configure it in Application.cfc:

```cfml
variables.framework = {
    diEngine = "di1",
    diLocations = "model,controllers",
    diConfig = {
        // DI/1 specific configuration
    }
};
```

Access beans in controllers:

```cfml
component accessors="true" {
    // Automatically injected
    property userService;
    property productService;

    function list(struct rc) {
        rc.users = userService.getAllUsers();
    }
}
```

### Standalone Usage

Use DI/1 outside of Ziggy MVC:

```cfml
// Create bean factory for a single folder
var beanFactory = new framework.ioc("/model");

// Multiple folders
var beanFactory = new framework.ioc("/model,/common");

// Or as array
var beanFactory = new framework.ioc(["/model", "/common"]);

// Get a bean
var userService = beanFactory.getBean("userService");
```

## Convention-Based Discovery

DI/1 follows naming conventions to discover and wire beans.

### Folder Structure

```
/model
  /beans           # Transients
    user.cfc       # → "user" or "userBean"
    product.cfc    # → "product" or "productBean"
  /services        # Singletons
    userService.cfc    # → "userService"
    orderService.cfc   # → "orderService"
  /gateways        # Singletons
    userGateway.cfc    # → "userGateway"
```

### Bean Naming Rules

- Bean name = filename without `.cfc` extension
- Alias = filename + singular folder name (e.g., `userBean`, `userService`)
- All beans in `beans` folder are transients
- All other beans are singletons

### Singular/Plural Mapping

DI/1 automatically handles common plural folder names:

| Folder | Singular |
|--------|----------|
| beans | bean |
| services | service |
| gateways | gateway |
| controllers | controller |

Custom mappings via configuration:

```cfml
diConfig = {
    singulars = {
        repositories = "repository",
        daos = "dao"
    }
}
```

## Autowiring

DI/1 automatically injects dependencies through:

### Property Injection

Declare properties with `accessors="true"`:

```cfml
component accessors="true" {
    property userService;      // Injected automatically
    property emailService;     // Injected automatically

    function createUser(data) {
        var user = userService.create(data);
        emailService.sendWelcome(user);
        return user;
    }
}
```

### Setter Injection

Define setter methods:

```cfml
component {
    function setUserService(userService) {
        variables.userService = userService;
    }

    function setLogger(loggingService) {
        // Custom initialization
        variables.logger = loggingService.getLogger("user");
    }
}
```

### Constructor Injection

Define an `init()` method with dependencies:

```cfml
component {
    function init(userDAO, cacheService) {
        variables.userDAO = userDAO;
        variables.cache = cacheService;
        return this;
    }
}
```

### Injection Priority

1. Constructor arguments (if `init()` exists)
2. Setter methods
3. Properties (with accessors)

### Transient Injection Note

**Important:** DI/1 injects both singletons and transients via constructors, but only singletons via setters and properties. Transients cannot be injected via setters.

## Bean Factory Access

Inject the bean factory itself to get beans dynamically:

```cfml
component accessors="true" {
    property beanFactory;

    function createReport(type) {
        // Get bean dynamically
        var reporter = beanFactory.getBean(type & "Reporter");
        return reporter.generate();
    }
}
```

## Configuration Options

Pass configuration when creating the bean factory:

```cfml
var config = {
    // Constant values
    constants = {
        dsn = "myDataSource",
        apiKey = "abc123"
    },

    // Exclude paths
    exclude = ["/model/legacy/"],

    // Additional transient folders
    transients = ["entities", "dtos"],

    // Custom singulars
    singulars = {
        repositories = "repository"
    },

    // Recurse into subfolders (default: true)
    recurse = true,

    // Throw on missing dependencies (default: false)
    strict = true,

    // Init method to call after injection
    initMethod = "setup",

    // Skip typed properties (default: true)
    omitTypedProperties = true,

    // Skip properties with defaults (default: true)
    omitDefaultedProperties = true
};

var beanFactory = new framework.ioc("/model", config);
```

### In Ziggy MVC

```cfml
variables.framework = {
    diEngine = "di1",
    diLocations = "model,controllers",
    diConfig = {
        constants = {
            dsn = "myDataSource"
        },
        strict = true
    }
};
```

## Programmatic Bean Declaration

### Builder Syntax (Recommended)

Use the fluent builder API to declare beans:

```cfml
// Alias for existing bean
beanFactory.declare("dataSource").aliasFor("mainDataSource");

// Constant value
beanFactory.declare("maxUsers").asValue(100);

// CFC by path
beanFactory.declare("specialService")
    .instanceOf("com.myapp.services.SpecialService");

// Factory method
beanFactory.declare("connection")
    .fromFactory("connectionFactory", "create");

// With arguments
beanFactory.declare("logger")
    .fromFactory("loggerFactory", "create")
    .withArguments(["userLog", "debug"]);

// Chain multiple declarations
beanFactory
    .declare("x").asValue(1).done()
    .declare("y").asValue(2).done()
    .declare("z").asValue(3).done()
    .load();
```

### Load Listeners

Organize declarations in a dedicated CFC:

```cfml
// LoadListener.cfc
component {
    function onLoad(beanFactory) {
        beanFactory
            .declare("appConfig").asValue(loadConfig()).done()
            .declare("logger").fromFactory("logFactory", "create").done()
            .load(); // Eagerly instantiate singletons
    }

    private function loadConfig() {
        return deserializeJSON(fileRead("/config/app.json"));
    }
}
```

Configure in Ziggy MVC:

```cfml
diConfig = {
    loadListener = "LoadListener"
}
```

## Working with Beans

### Getting Beans

```cfml
// Get a bean
var service = beanFactory.getBean("userService");

// With constructor arguments
var report = beanFactory.getBean("report", {startDate: now(), endDate: now()});

// Check if bean exists
if (beanFactory.containsBean("legacyService")) {
    var legacy = beanFactory.getBean("legacyService");
}

// Check if singleton
if (beanFactory.isSingleton("userService")) {
    // It's a singleton
}
```

### Bean Information

```cfml
// Get info about all beans
var info = beanFactory.getBeanInfo();

// Get info about specific bean
var info = beanFactory.getBeanInfo("userService");

// Flatten hierarchy (include parent factory beans)
var info = beanFactory.getBeanInfo(flatten=true);

// Filter by regex
var info = beanFactory.getBeanInfo(regex=".*Service");
```

### Injecting Properties

Populate a bean from a data struct:

```cfml
// Create and populate a bean
var user = beanFactory.injectProperties("user", formData);

// Populate existing instance
var user = beanFactory.getBean("user");
beanFactory.injectProperties(user, formData);

// With deep injection (nested objects)
beanFactory.injectProperties(user, formData, true);
```

## Parent Bean Factories

For modular applications, create parent/child relationships:

```cfml
// Parent factory with shared beans
var parentFactory = new framework.ioc("/common/model");

// Child factory with module-specific beans
var moduleFactory = new framework.ioc("/modules/admin/model");
moduleFactory.setParent(parentFactory);

// Child looks up beans in parent if not found locally
var sharedService = moduleFactory.getBean("sharedService");
```

Ziggy MVC automatically sets up parent/child relationships for subsystems.

## Extension Points

Override these methods to customize DI/1 behavior:

### missingBean()

Handle requests for unknown beans:

```cfml
// In a CFC that extends framework.ioc
function missingBean(beanName) {
    // Return a null object instead of throwing
    return new nullObject(beanName);
}
```

### construct()

Customize CFC instantiation:

```cfml
function construct(dottedPath) {
    // Custom instantiation logic
    return createObject("component", dottedPath);
}
```

### metadata()

Customize metadata retrieval:

```cfml
function metadata(dottedPath) {
    // Return custom metadata
    return getComponentMetadata(dottedPath);
}
```

## Common Patterns

### Service Layer

```cfml
// model/services/userService.cfc
component accessors="true" {
    property userDAO;
    property emailService;
    property validator;

    function createUser(struct userData) {
        // Validate
        var errors = validator.validate(userData, "user");
        if (arrayLen(errors)) {
            throw(type="ValidationError", message=errors[1]);
        }

        // Create user
        var user = userDAO.create(userData);

        // Send welcome email
        emailService.sendWelcome(user);

        return user;
    }
}
```

### Factory Pattern

```cfml
// model/services/reportFactory.cfc
component accessors="true" {
    property beanFactory;

    function create(required string type) {
        var beanName = type & "Report";
        if (!beanFactory.containsBean(beanName)) {
            throw(type="InvalidReportType", message="Unknown report: #type#");
        }
        return beanFactory.getBean(beanName);
    }
}
```

### Configuration Object

```cfml
// In Application.cfc
diConfig = {
    constants = {
        config = {
            dbHost = "localhost",
            dbName = "myapp",
            cacheEnabled = true
        }
    }
}

// In service
component accessors="true" {
    property config;

    function init() {
        // config struct is automatically injected
        if (config.cacheEnabled) {
            // Enable caching
        }
        return this;
    }
}
```

## Debugging

### Strict Mode

Enable strict mode to catch missing dependencies:

```cfml
diConfig = {
    strict = true  // Throws exception on missing dependencies
}
```

### Bean Info

Inspect discovered beans:

```cfml
var info = beanFactory.getBeanInfo();
writeDump(info);
```

### Load Event

Log when beans are loaded:

```cfml
// LoadListener.cfc
component {
    function onLoad(beanFactory) {
        var info = beanFactory.getBeanInfo();
        writeLog(file="di", text="Loaded #structCount(info.beanInfo)# beans");
    }
}
```

## Best Practices

1. **Use property injection** - Cleaner than setters for most cases
2. **Prefer constructor injection** for required dependencies
3. **Keep beans focused** - Single responsibility principle
4. **Avoid circular dependencies** - Restructure if needed
5. **Use transients for entities** - Domain objects should be fresh instances
6. **Use singletons for services** - Stateless services share well
7. **Inject the factory sparingly** - Only when dynamic lookup is needed
8. **Configure strict mode in development** - Catch issues early

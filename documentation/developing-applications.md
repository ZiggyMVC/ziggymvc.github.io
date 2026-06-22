---
layout: default
title: Developing Applications with Ziggy MVC
---

# Developing Applications with Ziggy MVC

This guide covers everything you need to know to build robust applications with Ziggy MVC.

## Application Structure

A Ziggy MVC application follows this conventional structure:

```
/myapp
  /controllers          # Request handlers
  /layouts              # Layout templates
  /model                # Business logic
    /beans              # Domain objects (transients)
    /services           # Service layer (singletons)
  /views                # View templates
    /section            # Views for each section
  /subsystems           # Optional: modular subsystems
  Application.cfc       # Application configuration
  index.cfm             # Entry point (usually empty)
```

### Alternative Structure

If you prefer different folder names, configure them:

```cfml
variables.framework = {
    controllersFolder = "handlers",
    viewsFolder = "pages",
    layoutsFolder = "wrappers"
};
```

## Views and Layouts

### Views

Views are CFM templates that render the response. They receive:

- `rc` - The request context (URL, form, and controller data)
- `local` - An empty struct for temporary view variables
- Access to all framework API methods

**Example view (`views/user/profile.cfm`):**

```html
<cfoutput>
<div class="profile">
    <h2>#rc.user.getName()#</h2>
    <p>Email: #rc.user.getEmail()#</p>
    <p>Member since: #dateFormat(rc.user.getCreatedDate(), "mmmm d, yyyy")#</p>
</div>
</cfoutput>
```

### Using the `local` Scope

Always use `local` for temporary variables in views to avoid conflicts:

```html
<cfset local.formattedDate = dateFormat(rc.createdAt, "yyyy-mm-dd")>
<cfoutput>#local.formattedDate#</cfoutput>
```

### Layouts

Layouts wrap views with common HTML structure. They receive a `body` variable containing the rendered view.

**Example layout (`layouts/default.cfm`):**

```html
<!DOCTYPE html>
<html>
<head>
    <title><cfoutput>#rc.pageTitle ?: "My Application"#</cfoutput></title>
</head>
<body>
    <nav>
        <cfoutput>
        <a href="#buildURL('main.default')#">Home</a>
        <a href="#buildURL('user.list')#">Users</a>
        </cfoutput>
    </nav>

    <main>
        <cfoutput>#body#</cfoutput>
    </main>
</body>
</html>
```

### Layout Cascade

For action `section.item`, Ziggy MVC searches for layouts in order:

1. `layouts/section/item.cfm` (view-specific)
2. `layouts/section.cfm` (section-specific)
3. `layouts/default.cfm` (site-wide)

Call `disableLayout()` in a controller to stop the cascade.

### Rendering Other Views

Include another view within a view using the `view()` function:

```html
<cfoutput>
<div class="sidebar">
    #view('widgets/recentPosts')#
</div>
<div class="main">
    #view('user/profile')#
</div>
</cfoutput>
```

## Controllers

Controllers handle requests and prepare data for views. Each controller is a CFC in the `/controllers` folder.

### Basic Controller

```cfml
component accessors="true" {

    // Services are automatically injected
    property userService;

    function list(struct rc) {
        rc.users = userService.getAllUsers();
    }

    function view(struct rc) {
        param name="rc.id" default="0";
        rc.user = userService.getUserById(rc.id);
        if (isNull(rc.user)) {
            rc.message = "User not found";
            setView("main.error");
        }
    }

    function save(struct rc) {
        var user = userService.save(rc);
        redirect("user.view", "id", user.getId());
    }

}
```

### Controller Lifecycle

For each request, Ziggy MVC calls methods in this order:

1. `Application.cfc : before(rc)`
2. `controller : before(rc)`
3. `controller : item(rc)` (the requested action)
4. `controller : after(rc)`
5. `Application.cfc : after(rc)`

### Before and After Methods

Use `before()` and `after()` for cross-cutting concerns:

```cfml
component accessors="true" {

    property securityService;

    function before(struct rc) {
        // Run before every action in this controller
        if (!securityService.isAuthenticated()) {
            redirect("login.default");
        }
    }

    function list(struct rc) {
        // Main action logic
    }

    function after(struct rc) {
        // Run after every action (e.g., logging)
    }

}
```

### Accessing the Framework

Declare the framework as a property or receive it in `init()`:

**Using property:**
```cfml
component accessors="true" {
    property framework;

    function doSomething(struct rc) {
        framework.redirect("user.list");
    }
}
```

**Using init():**
```cfml
component {
    function init(fw) {
        variables.fw = fw;
        return this;
    }

    function doSomething(struct rc) {
        variables.fw.redirect("user.list");
    }
}
```

### Handling Missing Methods

Override `onMissingMethod()` for dynamic actions:

```cfml
function onMissingMethod(string missingMethodName, struct missingMethodArguments) {
    var rc = missingMethodArguments.rc;
    var method = missingMethodArguments.method; // "before", "item", or "after"

    if (method == "item") {
        rc.message = "Action '#missingMethodName#' not found";
        setView("main.error");
    }
}
```

## Services and Domain Objects

### Service Layer

Services contain business logic and are singletons by default. Place them in `/model/services/`:

```cfml
// model/services/userService.cfc
component {

    property userDAO;
    property emailService;

    function init() {
        return this;
    }

    function getAllUsers() {
        return userDAO.findAll();
    }

    function createUser(required string email, required string name) {
        var user = new beans.user();
        user.setEmail(email);
        user.setName(name);

        userDAO.save(user);
        emailService.sendWelcome(user);

        return user;
    }

}
```

### Domain Objects (Beans)

Domain objects are transients (new instance each time). Place them in `/model/beans/`:

```cfml
// model/beans/user.cfc
component accessors="true" {

    property name="id" type="numeric";
    property name="email" type="string";
    property name="name" type="string";
    property name="createdAt" type="date";

    function init() {
        variables.createdAt = now();
        return this;
    }

    function getDisplayName() {
        return len(variables.name) ? variables.name : variables.email;
    }

}
```

### Populating Objects

Use `populate()` to map request context to object properties:

```cfml
function save(struct rc) {
    var user = getBeanFactory().getBean("user");

    // Populate all matching properties
    populate(user);

    // Or populate specific properties only
    populate(user, "email,name,phone");

    // With options
    populate(cfc=user, keys="email,name", trim=true, trustKeys=true);

    userService.save(user);
}
```

## Configuration

Configure Ziggy MVC via `variables.framework` in Application.cfc:

```cfml
variables.framework = {
    // Action parameter name
    action = "action",

    // Default action when none specified
    home = "main.default",

    // Error handler action
    error = "main.error",

    // Reload settings
    reload = "reload",
    password = "secret",

    // URL generation
    generateSES = false,
    SESOmitIndex = false,

    // Dependency injection
    diEngine = "di1",
    diLocations = "model,controllers",
    diConfig = {},

    // Development options
    reloadApplicationOnEveryRequest = false,
    trace = false,

    // Folder customization
    controllersFolder = "controllers",
    viewsFolder = "views",
    layoutsFolder = "layouts",

    // Subsystems (if used)
    usingSubsystems = false,
    defaultSubsystem = "home",
    subsystemDelimiter = ":",

    // Request handling
    unhandledExtensions = "cfc",
    unhandledPaths = "/flex2gateway",

    // Routes
    routes = [],
    routesCaseSensitive = true
};
```

### Key Configuration Options

| Setting | Default | Description |
|---------|---------|-------------|
| `action` | "action" | URL/form variable name for the action |
| `home` | "main.default" | Default action when none specified |
| `error` | "main.error" | Error handler action |
| `generateSES` | false | Generate SEO-friendly URLs |
| `diEngine` | "di1" | Dependency injection engine |
| `diLocations` | "model,controllers" | Folders to scan for beans |
| `trace` | false | Enable framework tracing |

## URL Routes

Define custom URL routes for clean URLs:

```cfml
variables.framework.routes = [
    // Simple routes
    { "/products" = "/product/list" },
    { "/products/:id" = "/product/view/id/:id" },

    // HTTP method-specific routes
    { "$GET/api/users" = "/api/user/list" },
    { "$POST/api/users" = "/api/user/create" },
    { "$PUT/api/users/:id" = "/api/user/update/id/:id" },
    { "$DELETE/api/users/:id" = "/api/user/delete/id/:id" },

    // Redirect routes
    { "/old-page" = "301:/new-page" },

    // Wildcard (must be last)
    { "*" = "/main/notfound" }
];
```

### Route Placeholders

Use `:name` for simple placeholders or `{name:[regex]}` for regex patterns:

```cfml
{ "/user/:id" = "/user/view/id/:id" },
{ "/user/{id:[0-9]+}" = "/user/view/id/:id" },
{ "/post/{slug:[a-z0-9-]+}" = "/blog/post/slug/:slug" }
```

### Resource Routes

Auto-generate RESTful routes:

```cfml
variables.framework.routes = [
    { "$RESOURCES" = "users,posts,comments" }
];
```

This generates standard REST endpoints:
- `GET /users` → `user.default`
- `GET /users/new` → `user.new`
- `POST /users` → `user.create`
- `GET /users/:id` → `user.show`
- `PUT /users/:id` → `user.update`
- `DELETE /users/:id` → `user.destroy`

## Rendering Data (REST APIs)

Return JSON, XML, or other formats using `renderData()`:

```cfml
function apiUsers(struct rc) {
    var users = userService.getAllUsers();

    // Return JSON
    renderData().data(users).type("json");
}

function apiUser(struct rc) {
    var user = userService.getUserById(rc.id);

    if (isNull(user)) {
        renderData().data({error: "User not found"}).type("json").statusCode(404);
    } else {
        renderData().data(user).type("json");
    }
}
```

### Supported Types

- `"json"` - JSON with proper content type
- `"jsonp"` - JSONP with callback
- `"rawjson"` - JSON without additional processing
- `"xml"` - XML data
- `"html"` - HTML content
- `"text"` - Plain text

### Custom Renderers

Define custom rendering functions:

```cfml
function render_csv(struct renderData) {
    return {
        contentType = "text/csv",
        output = convertToCSV(renderData.data)
    };
}

// Usage
renderData().data(users).type(render_csv);
```

## Error Handling

### Error Action

Configure a default error handler:

```cfml
variables.framework.error = "main.error";
```

Create the error view (`views/main/error.cfm`):

```html
<cfoutput>
<h1>An Error Occurred</h1>

<cfif structKeyExists(request, "exception")>
    <p>#request.exception.message#</p>

    <cfif structKeyExists(request, "failedAction")>
        <p>Failed action: #request.failedAction#</p>
    </cfif>
</cfif>

<p><a href="#buildURL('main.default')#">Return Home</a></p>
</cfoutput>
```

### Custom Error Handling

Override `onError()` in Application.cfc:

```cfml
function onError(exception, event) {
    // Log the error
    writeLog(file="errors", text=exception.message);

    // Set up error data
    request.exception = exception;
    request.event = event;

    // Call parent error handling
    super.onError(argumentCollection=arguments);
}
```

## Environment Support

Configure different settings per environment:

```cfml
function getEnvironment() {
    if (findNoCase("localhost", CGI.SERVER_NAME)) return "development";
    if (findNoCase("staging", CGI.SERVER_NAME)) return "staging";
    return "production";
}

variables.framework.environments = {
    development = {
        reloadApplicationOnEveryRequest = true,
        trace = true
    },
    staging = {
        trace = true
    },
    production = {
        password = "supersecretpassword"
    }
};

function setupEnvironment(string env) {
    // Environment-specific initialization
    if (env == "production") {
        // Set production database, caching, etc.
    }
}
```

## Application Lifecycle

Override these methods in Application.cfc:

### setupApplication()

Called when the application starts or reloads:

```cfml
function setupApplication() {
    // Initialize application-wide resources
    application.startTime = now();
}
```

### setupSession()

Called when a new session starts:

```cfml
function setupSession() {
    // Initialize session data
    session.cart = [];
}
```

### setupRequest()

Called at the start of each request:

```cfml
function setupRequest() {
    // Queue additional controllers
    controller("security.checkAuth");

    // Set up request-scoped data
    request.startTime = getTickCount();
}
```

### setupView()

Called after controllers run, before view renders:

```cfml
function setupView(struct rc) {
    // Set up data needed by all views
    rc.currentYear = year(now());
}
```

### setupResponse()

Called at the end of every request:

```cfml
function setupResponse(struct rc) {
    // Clean up, logging, etc.
    var elapsed = getTickCount() - request.startTime;
    writeLog(file="perf", text="Request completed in #elapsed#ms");
}
```

## Session Management

Ziggy MVC provides secure session helper methods:

```cfml
// Write to session
sessionWrite("userId", 123);

// Read from session
var userId = sessionRead("userId");

// Read with default
var cart = sessionRead("cart", []);

// Check if exists
if (sessionHas("userId")) { ... }

// Delete from session
sessionDelete("userId");

// Execute within session lock
sessionLock(function() {
    // Thread-safe session operations
    session.counter++;
});
```

## Building URLs

Use `buildURL()` for framework-aware URL generation:

```cfml
<!--- Simple action --->
<a href="#buildURL('user.list')#">Users</a>

<!--- With parameters --->
<a href="#buildURL('user.view', 'id=#user.getId()#')#">View</a>

<!--- With struct parameters --->
<a href="#buildURL(action='user.edit', queryString={id: user.getId(), mode: 'full'})#">Edit</a>
```

### buildCustomURL()

For route-based URLs:

```cfml
<a href="#buildCustomURL('/users/' & user.getId())#">View User</a>
```

## Redirects

Redirect to another action:

```cfml
// Simple redirect
redirect("user.list");

// With flash data preservation
redirect(action="user.view", preserve="message", append="id");

// Preserve all request context
redirect(action="user.view", preserve="all");
```

### redirectCustomURL()

For custom URL redirects:

```cfml
redirectCustomURL("/users/" & user.getId());
```

## Tracing and Debugging

Enable tracing for debugging:

```cfml
variables.framework.trace = true;
```

Add custom trace messages:

```cfml
frameworkTrace("Processing user #rc.id#");
```

View trace output appended to responses during development.

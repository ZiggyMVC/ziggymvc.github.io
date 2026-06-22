---
layout: default
title: Reference Manual
---

# Ziggy MVC Reference Manual

This reference manual provides a comprehensive list of all framework configuration options, API methods, and lifecycle hooks.

## Configuration Reference

Configure Ziggy MVC via `variables.framework` in Application.cfc:

```cfml
component extends="framework.ziggy" {
    variables.framework = {
        // Configuration options here
    };
}
```

### Core Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `action` | string | `"action"` | URL/form variable name for the action |
| `base` | string | *auto* | Base path for the application |
| `cfcbase` | string | *auto* | Dotted path for CFCs |
| `home` | string | `"main.default"` | Default action when none specified |
| `error` | string | `"main.error"` | Error handler action |
| `reload` | string | `"reload"` | URL parameter to trigger reload |
| `password` | string | `"true"` | Password for reload |
| `trace` | boolean | `false` | Enable framework tracing |
| `noLowerCase` | boolean | `false` | Preserve action case |

### URL Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `generateSES` | boolean | `false` | Generate SEO-friendly URLs |
| `SESOmitIndex` | boolean | `false` | Omit index.cfm from SES URLs |
| `baseURL` | string | `"useCgiScriptName"` | Base URL for generated links |

### Folder Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `controllersFolder` | string | `"controllers"` | Controllers folder name |
| `viewsFolder` | string | `"views"` | Views folder name |
| `layoutsFolder` | string | `"layouts"` | Layouts folder name |
| `subsystemsFolder` | string | `"subsystems"` | Subsystems folder name |

### Subsystem Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `usingSubsystems` | boolean | `false` | Enable legacy subsystems |
| `defaultSubsystem` | string | `"home"` | Default subsystem (legacy) |
| `subsystemDelimiter` | string | `":"` | Subsystem/action separator |
| `siteWideLayoutSubsystem` | string | `"common"` | Site-wide layout subsystem (legacy) |
| `subsystems` | struct | `{}` | Per-subsystem configuration |

### Dependency Injection Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `diEngine` | string | `"di1"` | DI engine: `"di1"`, `"aop1"`, `"wirebox"`, `"none"` |
| `diLocations` | string/array | `"model,controllers"` | Folders to scan for beans |
| `diConfig` | struct | `{}` | Engine-specific configuration |
| `diComponent` | string | `"framework.ioc"` | Custom DI component path |

### Request Handling

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `unhandledExtensions` | string | `"cfc"` | File extensions to ignore |
| `unhandledPaths` | string | `"/flex2gateway"` | URL paths to ignore |
| `unhandledErrorCaught` | boolean | `false` | Catch errors in unhandled requests |
| `preserveKeyURLKey` | string | `"fw1pk"` | Flash scope key name |
| `maxNumContextsPreserved` | number | `10` | Max preserved request contexts |

### Route Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `routes` | array | `[]` | Custom URL routes |
| `routesCaseSensitive` | boolean | `true` | Case-sensitive route matching |
| `resourceRouteTemplates` | array | *built-in* | Templates for resource routes |

### Performance Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `cacheFileExists` | boolean | `false` | Cache file existence checks |
| `reloadApplicationOnEveryRequest` | boolean | `false` | Auto-reload on every request |
| `applicationKey` | string | `"framework.one"` | Unique key for multiple instances |

### REST Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `preflightOptions` | boolean | `false` | Handle OPTIONS preflight requests |
| `optionsAccessControl` | struct | `{}` | CORS configuration |

## API Reference

### Controller and Service Access

#### controller(action)
Queue an additional controller for execution.

```cfml
function setupRequest() {
    controller("security.checkAuth");
    controller("common.loadSettings");
}
```

#### abortController()
Stop controller execution immediately.

```cfml
function save(struct rc) {
    if (!isValid(rc)) {
        rc.errors = validate(rc);
        setView("user.edit");
        abortController();
    }
    // This code won't execute if aborted
}
```

### View and Layout Control

#### setView(action)
Override the view to render.

```cfml
function list(struct rc) {
    if (structKeyExists(rc, "format") && rc.format == "grid") {
        setView("user.listGrid");
    }
}
```

#### setLayout(action, cascade)
Override the layout. Set `cascade` to `false` to stop the layout cascade.

```cfml
function api(struct rc) {
    setLayout("api.minimal", false);
}
```

#### view(path, args)
Render another view inline.

```cfml
<cfoutput>#view("widgets/sidebar")#</cfoutput>
<cfoutput>#view("user/profile", {user: rc.currentUser})#</cfoutput>
```

#### layout(path, body)
Render content within a layout.

```cfml
<cfoutput>#layout("modal", '<p>Modal content</p>')#</cfoutput>
```

#### disableLayout()
Stop layout rendering for this request.

```cfml
function ajaxEndpoint(struct rc) {
    disableLayout();
    // Return raw content
}
```

#### enableLayout()
Re-enable layout rendering.

```cfml
function conditionalLayout(struct rc) {
    if (someCondition) {
        enableLayout();
    }
}
```

### URL Generation

#### buildURL(action, queryString)
Generate a framework URL.

```cfml
buildURL("user.list")
buildURL("user.view", "id=42")
buildURL("user.edit", {id: 42, mode: "full"})
buildURL(action="admin:user.list", queryString="page=2")
```

#### buildCustomURL(uri)
Generate a URL using the resolved base URL.

```cfml
buildCustomURL("/api/users/42")
buildCustomURL("/products/:id") // Variables from rc substituted
```

### Redirects

#### redirect(action, preserve, append, statusCode, queryString)
Redirect to another action.

| Parameter | Description |
|-----------|-------------|
| `action` | Target action |
| `preserve` | Keys to preserve in flash scope (`"all"` or comma-list) |
| `append` | Keys to append to URL |
| `statusCode` | HTTP status (default: 302) |
| `queryString` | Additional query parameters |

```cfml
redirect("user.list");
redirect(action="user.view", append="id");
redirect(action="user.list", preserve="message");
redirect(action="user.list", preserve="all");
redirect(action="main.default", statusCode=301);
```

#### redirectCustomURL(uri, statusCode)
Redirect to a custom URL.

```cfml
redirectCustomURL("/users/42");
redirectCustomURL("/old-page", 301);
```

### Data Rendering

#### renderData()
Return a builder for rendering data (REST APIs).

```cfml
// JSON response
renderData().data(users).type("json");

// With status code
renderData().data({error: "Not found"}).type("json").statusCode(404);

// XML response
renderData().data(xmlDoc).type("xml");

// HTML response
renderData().data(htmlString).type("html");

// Plain text
renderData().data("OK").type("text");

// JSONP with callback
renderData().data(data).type("jsonp").callback("myCallback");

// Custom renderer function
renderData().data(data).type(myCustomRenderer);
```

**Supported Types:**
- `"json"` - JSON with `application/json` content type
- `"jsonp"` - JSONP with callback
- `"rawjson"` - JSON without additional processing
- `"xml"` - XML data
- `"html"` - HTML content
- `"text"` - Plain text

### Data Population

#### populate(cfc, keys, trim, deep, trustKeys)
Populate an object from the request context.

| Parameter | Type | Description |
|-----------|------|-------------|
| `cfc` | object | Object to populate |
| `keys` | string | Comma-list of properties (empty = all) |
| `trim` | boolean | Trim string values |
| `deep` | boolean | Populate nested objects |
| `trustKeys` | boolean | Skip key existence check |

```cfml
var user = getBeanFactory().getBean("user");
populate(user);
populate(user, "firstName,lastName,email");
populate(cfc=user, keys="email", trim=true);
```

### Bean Factory

#### getBeanFactory(subsystem)
Get the bean factory. Without argument, returns the current subsystem's factory.

```cfml
var bf = getBeanFactory();
var adminBF = getBeanFactory("admin");
```

#### getDefaultBeanFactory()
Get the main application's bean factory.

```cfml
var mainBF = getDefaultBeanFactory();
```

#### setBeanFactory(beanFactory)
Set the default bean factory.

```cfml
var customBF = new myCustomBeanFactory();
setBeanFactory(customBF);
```

#### hasBeanFactory()
Check if a bean factory exists.

```cfml
if (hasBeanFactory()) {
    // Bean factory available
}
```

#### hasDefaultBeanFactory()
Check if the default bean factory exists.

#### getSubsystemBeanFactory(subsystem)
Get a subsystem's bean factory.

#### setSubsystemBeanFactory(subsystem, beanFactory)
Set a subsystem's bean factory.

#### hasSubsystemBeanFactory(subsystem)
Check if a subsystem has its own bean factory.

### Session Management

#### sessionRead(key, defaultValue)
Read from session scope.

```cfml
var userId = sessionRead("userId");
var cart = sessionRead("cart", []);
```

#### sessionWrite(key, value)
Write to session scope.

```cfml
sessionWrite("userId", user.getId());
sessionWrite("lastActivity", now());
```

#### sessionHas(key)
Check if a session key exists.

```cfml
if (sessionHas("userId")) {
    // User is logged in
}
```

#### sessionDelete(key)
Delete a session key.

```cfml
sessionDelete("userId");
```

#### sessionDefault(key, value)
Set a session value only if it doesn't exist.

```cfml
sessionDefault("cart", []);
```

#### sessionLock(callback)
Execute code within a session lock.

```cfml
sessionLock(function() {
    session.counter++;
});
```

### Action Parsing

#### getAction()
Get the current action.

```cfml
var action = getAction(); // e.g., "user.list"
```

#### getSection(action)
Get the section from an action.

```cfml
var section = getSection("user.list"); // "user"
```

#### getItem(action)
Get the item from an action.

```cfml
var item = getItem("user.list"); // "list"
```

#### getSubsystem(action)
Get the subsystem from an action.

```cfml
var sub = getSubsystem("admin:user.list"); // "admin"
```

#### getSectionAndItem(action)
Get section.item without subsystem.

```cfml
var si = getSectionAndItem("admin:user.list"); // "user.list"
```

#### getFullyQualifiedAction(action)
Get the complete action including subsystem.

```cfml
var fqa = getFullyQualifiedAction(); // "admin:user.list"
```

### Configuration Access

#### getConfig()
Get a copy of the framework configuration.

```cfml
var config = getConfig();
if (config.generateSES) {
    // Using SES URLs
}
```

#### getEnvironment()
Get the current environment name.

```cfml
var env = getEnvironment(); // "development", "production", etc.
```

#### getBaseURL()
Get the configured base URL.

```cfml
var baseURL = getBaseURL();
```

### Routing

#### addRoute(routes, target, methods, statusCode)
Programmatically add routes.

```cfml
addRoute("/products", "/product/list");
addRoute("/products/:id", "/product/view/id/:id");
addRoute("/api/users", "/api/user/list", ["GET"]);
addRoute("/old-page", "/new-page", [], "301");
```

#### getRoutes()
Get all defined routes.

```cfml
var routes = getRoutes();
```

### Framework Control

#### frameworkTrace(message)
Add a trace message.

```cfml
frameworkTrace("Processing user #rc.id#");
```

#### enableFrameworkTrace()
Enable tracing for this request.

#### disableFrameworkTrace()
Disable tracing for this request.

#### getFrameworkTrace()
Get trace data for this request.

```cfml
var traceData = getFrameworkTrace();
```

#### isFrameworkReloadRequest()
Check if this is a reload request.

```cfml
if (isFrameworkReloadRequest()) {
    // Reload in progress
}
```

### Subsystem Methods

#### usingSubsystems()
Check if subsystems are enabled.

```cfml
if (usingSubsystems()) {
    // Subsystems enabled
}
```

#### getSubsystemBase(subsystem)
Get the base path for a subsystem.

```cfml
var base = getSubsystemBase("admin");
```

#### getSubsystemConfig(subsystem)
Get a subsystem's configuration.

```cfml
var config = getSubsystemConfig("admin");
```

#### getDefaultSubsystem()
Get the default subsystem name.

## Lifecycle Hooks

Override these methods in Application.cfc to customize framework behavior.

### setupApplication()
Called when the application starts or reloads.

```cfml
function setupApplication() {
    // Initialize application-wide resources
    application.config = loadConfig();
}
```

### setupEnvironment(env)
Called after environment is determined.

```cfml
function setupEnvironment(string env) {
    if (env == "development") {
        application.debug = true;
    }
}
```

### setupSession()
Called when a new session starts.

```cfml
function setupSession() {
    session.cart = [];
    session.recentlyViewed = [];
}
```

### setupRequest()
Called at the start of each request.

```cfml
function setupRequest() {
    // Queue security controller
    controller("security.check");

    // Set request timestamp
    request.startTime = getTickCount();
}
```

### setupSubsystem(subsystem)
Called once when a subsystem is first accessed.

```cfml
function setupSubsystem(string subsystem) {
    if (subsystem == "api") {
        disableLayout();
    }
}
```

### setupView(rc)
Called after controllers, before view renders.

```cfml
function setupView(struct rc) {
    rc.currentYear = year(now());
    rc.userName = session.user?.getName() ?: "Guest";
}
```

### setupResponse(rc)
Called at the end of every request.

```cfml
function setupResponse(struct rc) {
    var elapsed = getTickCount() - request.startTime;
    writeLog(file="perf", text="#getAction()# took #elapsed#ms");
}
```

### onMissingView(rc)
Called when a view is not found.

```cfml
function onMissingView(struct rc) {
    return "<h1>Page Not Found</h1>";
    // Or redirect
    // redirect("main.notfound");
}
```

### onError(exception, event)
Called when an error occurs.

```cfml
function onError(exception, event) {
    writeLog(file="errors", text=exception.message, type="error");
    super.onError(argumentCollection=arguments);
}
```

### onReload()
Called before application reload.

```cfml
function onReload() {
    // Clean up before reload
    structClear(application);
}
```

### getEnvironment()
Override to determine the current environment.

```cfml
function getEnvironment() {
    if (findNoCase("localhost", CGI.SERVER_NAME)) {
        return "development";
    }
    return "production";
}
```

### before(rc)
Application-level controller method, runs before all others.

```cfml
function before(struct rc) {
    rc.appName = "My Application";
}
```

### after(rc)
Application-level controller method, runs after all others.

```cfml
function after(struct rc) {
    // Post-processing
}
```

## Request Scope Variables

These variables are available during request processing:

| Variable | Description |
|----------|-------------|
| `request.context` | The request context (rc) |
| `request.action` | Current action |
| `request.section` | Current section |
| `request.item` | Current item |
| `request.subsystem` | Current subsystem |
| `request.layout` | Layout enabled flag |
| `request.exception` | Exception (in error handler) |
| `request.failedAction` | Failed action (in error handler) |
| `request.missingView` | Missing view name (if applicable) |

## DI/1 Configuration

Options for `diConfig` when using DI/1:

| Option | Type | Description |
|--------|------|-------------|
| `constants` | struct | Constant values to inject |
| `exclude` | array | Paths to exclude from scanning |
| `transients` | array | Additional transient folder names |
| `singulars` | struct | Plural-to-singular folder mappings |
| `strict` | boolean | Throw on missing dependencies |
| `recurse` | boolean | Recurse into subfolders |
| `initMethod` | string | Method to call after injection |
| `loadListener` | string | CFC to call after factory loads |
| `omitTypedProperties` | boolean | Skip typed properties |
| `omitDefaultedProperties` | boolean | Skip properties with defaults |

## Version Information

| Property | Value |
|----------|-------|
| Ziggy MVC Version | 5.0.0 |
| Based on FW/1 | 4.3.0 |
| Supported CFML | Adobe CF 10+, Lucee 4.5+ |
| License | Apache 2.0 |

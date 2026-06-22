---
layout: default
title: Using Subsystems
---

# Using Subsystems

Subsystems allow you to modularize your Ziggy MVC application into self-contained "mini-applications." Each subsystem has its own controllers, views, layouts, and models.

## When to Use Subsystems

Subsystems are useful when:

- Your application has distinct functional areas (admin, API, public site)
- You want to organize a large codebase into modules
- You need to integrate third-party Ziggy MVC/FW/1 applications
- Different parts of your app have different requirements

## Subsystem Architecture

Ziggy MVC supports two subsystem approaches:

### Modern Subsystems (Subsystems 2.0)

Subsystems reside in a `subsystems` folder alongside your main application:

```
/myapp
  /controllers          # Main app controllers
  /views                # Main app views
  /layouts              # Main app layouts
  /model                # Main app models
  /subsystems
    /admin              # Admin subsystem
      /controllers
      /views
      /layouts
      /model
    /api                # API subsystem
      /controllers
      /views
      /model
    /blog               # Blog subsystem
      /controllers
      /views
      /layouts
      /model
  Application.cfc
  index.cfm
```

### Legacy Subsystems (Subsystems 1.0)

In the legacy approach, subsystems are top-level folders and the main application is itself a subsystem:

```
/myapp
  /home                 # Main/default subsystem
    /controllers
    /views
    /layouts
    /model
  /admin                # Admin subsystem
    /controllers
    /views
    /layouts
    /model
  Application.cfc
  index.cfm
```

**We recommend Modern Subsystems (2.0)** for new applications.

## Enabling Subsystems

### Modern Subsystems

Modern subsystems are **enabled automatically**. Simply create a `subsystems` folder with your modules, and Ziggy MVC detects them.

```cfml
// Application.cfc - No special configuration needed!
component extends="framework.ziggy" {
    this.name = "MyApp";
}
```

### Legacy Subsystems

Legacy subsystems require explicit configuration:

```cfml
variables.framework = {
    usingSubsystems = true,
    defaultSubsystem = "home",
    siteWideLayoutSubsystem = "common"
};
```

## Accessing Subsystems

Use the colon (`:`) delimiter to specify subsystem actions:

```
?action=admin:user.list       # admin subsystem, user controller, list action
?action=blog:post.view&id=42  # blog subsystem, post controller, view action
?action=api:v1.users          # api subsystem, v1 controller, users action
```

### Default Behavior

- No subsystem specified → main application
- `?action=user.list` → `/controllers/user.cfc`, `list()` method
- `?action=admin:user.list` → `/subsystems/admin/controllers/user.cfc`, `list()` method

## URL Building

### Within the Same Subsystem

```cfml
<!--- In admin:user.list view --->
<a href="#buildURL('user.edit')#">Edit</a>
<!--- Generates: ?action=admin:user.edit --->
```

### Cross-Subsystem Links

```cfml
<!--- Link to another subsystem --->
<a href="#buildURL('blog:post.list')#">View Blog</a>

<!--- Link to main application from subsystem --->
<a href="#buildURL(':main.default')#">Home</a>
```

### SES URLs

With SES URLs enabled:

```cfml
<a href="#buildURL('admin:dashboard.main')#">Admin Dashboard</a>
<!--- Generates: /index.cfm/admin/dashboard/main --->
```

## Subsystem Structure

Each subsystem mirrors the main application structure:

```
/subsystems/admin
  /controllers
    dashboard.cfc
    user.cfc
    settings.cfc
  /views
    /dashboard
      main.cfm
    /user
      list.cfm
      edit.cfm
    /settings
      index.cfm
  /layouts
    default.cfm        # Subsystem default layout
  /model
    /services
      adminService.cfc
    /beans
      adminUser.cfc
```

## Layout Cascade

For subsystem action `admin:user.list`, layouts are searched:

1. `subsystems/admin/layouts/user/list.cfm` (view-specific)
2. `subsystems/admin/layouts/user.cfm` (section-specific)
3. `subsystems/admin/layouts/default.cfm` (subsystem default)
4. `layouts/default.cfm` (site-wide)

### Site-Wide Layout

The main application's `layouts/default.cfm` applies to all subsystems by default. This is perfect for consistent headers/footers across your entire site.

### Subsystem-Only Layout

To stop at the subsystem layout, call `disableLayout()`:

```cfml
// In subsystems/admin/layouts/default.cfm
<cfset disableLayout()>
<!DOCTYPE html>
<html>
<head>
    <title>Admin Panel</title>
</head>
<body>
    <cfoutput>#body#</cfoutput>
</body>
</html>
```

## Subsystem Configuration

### setupSubsystem()

Called once when a subsystem is first accessed:

```cfml
// Application.cfc
function setupSubsystem(string subsystem) {
    switch (subsystem) {
        case "admin":
            // Admin-specific initialization
            break;
        case "api":
            // API-specific initialization
            disableLayout(); // APIs typically don't use layouts
            break;
    }
}
```

### Per-Subsystem Settings

Configure subsystem-specific options:

```cfml
variables.framework = {
    subsystems = {
        admin = {
            diLocations = "subsystems/admin/model",
            // Other settings...
        },
        api = {
            diLocations = "subsystems/api/model",
            // Other settings...
        }
    }
};
```

## Bean Factories and Subsystems

### Automatic Setup

Ziggy MVC automatically creates a DI/1 bean factory for each subsystem, with the main application's bean factory as its parent.

```
Main Bean Factory (model, controllers)
    ├── Admin Bean Factory (subsystems/admin/model)
    ├── API Bean Factory (subsystems/api/model)
    └── Blog Bean Factory (subsystems/blog/model)
```

### Shared Beans

Beans in the main `model` folder are available to all subsystems:

```
/model
  /services
    userService.cfc      # Available everywhere
    emailService.cfc     # Available everywhere
/subsystems/admin/model
  /services
    adminService.cfc     # Only in admin subsystem
```

### Accessing Bean Factories

```cfml
// Get the default (main) bean factory
var mainBF = getDefaultBeanFactory();

// Get a subsystem's bean factory
var adminBF = getBeanFactory("admin");

// Check if subsystem has its own factory
if (hasSubsystemBeanFactory("admin")) {
    // ...
}
```

### Manual Bean Factory Setup

Configure custom bean factories per subsystem:

```cfml
function setupSubsystem(string subsystem) {
    if (subsystem == "admin") {
        // Create custom bean factory
        var bf = new framework.ioc("subsystems/admin/model");
        bf.setParent(getDefaultBeanFactory());
        setSubsystemBeanFactory("admin", bf);
    }
}
```

### Using ColdSpring or Other DI Containers

```cfml
function setupSubsystem(string subsystem) {
    var configPath = "/subsystems/" & subsystem & "/config/coldspring.xml";

    if (fileExists(expandPath(configPath))) {
        var bf = new coldspring.beans.DefaultXmlBeanFactory();
        bf.loadBeans(configPath);
        bf.setParent(getDefaultBeanFactory());
        setSubsystemBeanFactory(subsystem, bf);
    }
}
```

## Cross-Subsystem Communication

### Calling Actions

Queue controller execution from another subsystem:

```cfml
function before(struct rc) {
    // Run main app's security check
    controller(":security.checkAuth");
}
```

### Accessing Other Subsystem Beans

```cfml
// Get bean from another subsystem
var adminService = getBeanFactory("admin").getBean("adminService");

// Or through the main factory if it's a shared bean
var userService = getDefaultBeanFactory().getBean("userService");
```

### Redirecting Across Subsystems

```cfml
// From admin subsystem, redirect to main app
redirect(":main.default");

// From main app, redirect to admin
redirect("admin:dashboard.main");
```

## API Subsystems

A common pattern is using subsystems for REST APIs:

```cfml
// subsystems/api/controllers/users.cfc
component {

    property userService;

    function init(fw) {
        variables.fw = fw;
        return this;
    }

    function default(struct rc) {
        // GET /api/users - List users
        var users = userService.getAll();
        variables.fw.renderData().data(users).type("json");
    }

    function show(struct rc) {
        // GET /api/users/:id
        param name="rc.id" default="0";
        var user = userService.getById(rc.id);
        if (isNull(user)) {
            variables.fw.renderData()
                .data({error: "User not found"})
                .type("json")
                .statusCode(404);
        } else {
            variables.fw.renderData().data(user).type("json");
        }
    }

    function create(struct rc) {
        // POST /api/users
        var user = userService.create(rc);
        variables.fw.renderData()
            .data(user)
            .type("json")
            .statusCode(201);
    }

}
```

Configure API routes:

```cfml
variables.framework.routes = [
    { "$GET/api/users" = "api:users.default" },
    { "$GET/api/users/:id" = "api:users.show/id/:id" },
    { "$POST/api/users" = "api:users.create" },
    { "$PUT/api/users/:id" = "api:users.update/id/:id" },
    { "$DELETE/api/users/:id" = "api:users.destroy/id/:id" }
];
```

## Admin Subsystems

Common pattern for admin areas:

```cfml
// Application.cfc
function setupSubsystem(string subsystem) {
    if (subsystem == "admin") {
        // Set admin-specific layout
        setLayout("admin");
    }
}

// Queue security check for all admin requests
function setupRequest() {
    if (getSubsystem() == "admin") {
        controller("admin:security.requireLogin");
    }
}
```

Admin security controller:

```cfml
// subsystems/admin/controllers/security.cfc
component accessors="true" {

    property authService;

    function requireLogin(struct rc) {
        if (!authService.isAdmin()) {
            redirect(":login.default", "returnTo", getFullyQualifiedAction());
        }
    }

}
```

## Subsystem Helpers

### Getting Subsystem Information

```cfml
// Get current subsystem
var currentSubsystem = getSubsystem();

// Check if using subsystems
if (usingSubsystems()) {
    // ...
}

// Get subsystem from an action
var sub = getSubsystem("admin:user.list"); // Returns "admin"

// Get section and item
var section = getSection("admin:user.list"); // Returns "user"
var item = getItem("admin:user.list"); // Returns "list"

// Get fully qualified action
var fqa = getFullyQualifiedAction(); // e.g., "admin:user.list"
```

### Subsystem Base Path

```cfml
// Get the base path for a subsystem
var basePath = getSubsystemBase("admin");
// Returns "subsystems/admin/" (modern) or "admin/" (legacy)
```

## Best Practices

1. **Use Modern Subsystems** - The `subsystems` folder approach is cleaner
2. **Share common services** - Put shared beans in the main `model` folder
3. **Consistent naming** - Use clear, descriptive subsystem names
4. **Security by subsystem** - Implement access control in `setupRequest()`
5. **Keep subsystems focused** - Each subsystem should have a single purpose
6. **Use site-wide layouts** - Share common UI in `layouts/default.cfm`
7. **Document subsystem APIs** - Especially for API subsystems
8. **Test subsystems independently** - They should be self-contained

## Migrating to Subsystems

### From a Monolithic App

1. Create the `subsystems` folder
2. Move feature-specific code into subsystem folders
3. Keep shared services in the main `model` folder
4. Update links to use subsystem prefixes
5. Test thoroughly

### From Legacy to Modern Subsystems

1. Create a `subsystems` folder
2. Move subsystem folders into it
3. Move the default subsystem contents to the main app level
4. Remove `usingSubsystems`, `defaultSubsystem`, `siteWideLayoutSubsystem` config
5. Update any hardcoded paths

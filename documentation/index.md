---
layout: default
title: Getting Started with Ziggy MVC
---

# Getting Started with Ziggy MVC

Welcome to Ziggy MVC version 5.0.0! This guide will help you get up and running with Ziggy MVC quickly.

## What is Ziggy MVC?

Ziggy MVC is a family of small, lightweight, convention-over-configuration frameworks for CFML:

- **Ziggy MVC** provides the MVC (Model-View-Controller) architecture
- **DI/1** provides dependency injection (Inversion of Control)
- **AOP/1** provides aspect-oriented programming features on top of DI/1

Ziggy MVC 5.0.0 was forked from FW/1 4.3.0 and is fully backward compatible with FW/1 applications. The only change required is updating your Application.cfc to extend `framework.ziggy` instead of `framework.one`.

## Requirements

Ziggy MVC supports the following CFML engines:

- **Adobe ColdFusion** 10 or later
- **Lucee** 4.5.0 or later (Lucee 5.x recommended)

## Installation

### Via CommandBox (Recommended)

```bash
box install ziggymvc
```

For the latest development version:

```bash
box install ziggymvc@be
```

### Via GitHub

Download the framework from the [Ziggy MVC GitHub repository](https://github.com/ZiggyMVC/ZiggyMVC) and extract it to your project.

### Framework Mapping

The `framework` folder must be accessible to your application. You can either:

1. Place the `framework` folder in your webroot
2. Create a `/framework` mapping in your CFML administrator
3. Define a mapping in your Application.cfc:

```cfml
component extends="framework.ziggy" {
    this.name = "MyApp";
    this.mappings["/framework"] = expandPath("/path/to/framework");
}
```

## Building Your First Application

Let's build a simple "Hello World" application step by step.

### Step 1: Application Structure

Create this folder structure:

```
/myapp
  /controllers
  /views
    /main
  Application.cfc
  index.cfm
```

### Step 2: Application.cfc

Create your Application.cfc extending the framework:

```cfml
component extends="framework.ziggy" {
    this.name = "MyFirstZiggyApp";
    this.sessionManagement = true;
}
```

### Step 3: index.cfm

Create an empty index.cfm file (Ziggy MVC needs this as an entry point):

```cfml
<!--- This file intentionally left empty --->
```

### Step 4: Your First View

Create `views/main/default.cfm`:

```html
<h1>Hello, Ziggy MVC!</h1>
<p>Welcome to your first Ziggy MVC application.</p>
<p>Current time: <cfoutput>#timeFormat(now(), "HH:mm:ss")#</cfoutput></p>
```

### Step 5: Run Your Application

Navigate to your application in a browser. You should see your "Hello, Ziggy MVC!" message.

**Congratulations!** You've built your first Ziggy MVC application.

## Understanding URL Structure

Ziggy MVC uses a simple URL convention to route requests:

```
index.cfm?action=section.item
```

Where:
- **section** maps to a controller file in `/controllers/` and a folder in `/views/`
- **item** maps to a method in the controller and a view file

### Default Action

When no action is specified, Ziggy MVC uses `main.default`:
- Controller: `/controllers/main.cfc` with method `default()`
- View: `/views/main/default.cfm`

### SES URLs

You can also use SEO-friendly URLs:

```
index.cfm/section/item
index.cfm/section/item/name/value
```

### Examples

| URL | Controller | Method | View |
|-----|------------|--------|------|
| `?action=main.default` | main.cfc | default() | main/default.cfm |
| `?action=user.list` | user.cfc | list() | user/list.cfm |
| `?action=product.view&id=42` | product.cfc | view() | product/view.cfm |

## Adding a Controller

Controllers contain your request-handling logic. Create `/controllers/main.cfc`:

```cfml
component {

    function default(struct rc) {
        rc.greeting = "Hello from the controller!";
        rc.currentTime = now();
    }

}
```

Update your view to use the data from the controller:

```html
<h1><cfoutput>#rc.greeting#</cfoutput></h1>
<p>Current time: <cfoutput>#dateTimeFormat(rc.currentTime, "full")#</cfoutput></p>
```

### Understanding the Request Context (rc)

The `rc` (request context) struct is passed to every controller method and is available in every view. It contains:

- All URL parameters
- All form variables
- Any data you add in controllers

Form variables take precedence over URL parameters with the same name.

## Adding Layouts

Layouts wrap your views with common HTML structure. Create `/layouts/default.cfm`:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>My Ziggy MVC Application</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        header { border-bottom: 2px solid #333; padding-bottom: 10px; }
        footer { margin-top: 40px; color: #666; font-size: 0.9em; }
    </style>
</head>
<body>
    <header>
        <h1>My Application</h1>
    </header>

    <main>
        <cfoutput>#body#</cfoutput>
    </main>

    <footer>
        <p>Powered by Ziggy MVC</p>
    </footer>
</body>
</html>
```

The `body` variable contains the rendered view content.

### Layout Cascade

Ziggy MVC searches for layouts in this order:

1. `layouts/section/item.cfm` (view-specific layout)
2. `layouts/section.cfm` (section-specific layout)
3. `layouts/default.cfm` (site-wide layout)

## Adding Services

Business logic belongs in services, not controllers. Create `/model/services/greetingService.cfc`:

```cfml
component {

    function getGreeting(string name = "World") {
        return "Hello, " & arguments.name & "!";
    }

    function getTimeOfDay() {
        var hour = hour(now());
        if (hour < 12) return "morning";
        if (hour < 17) return "afternoon";
        return "evening";
    }

}
```

Ziggy MVC automatically injects services into controllers using DI/1. Update your controller:

```cfml
component accessors="true" {

    property greetingService;

    function default(struct rc) {
        rc.greeting = greetingService.getGreeting(rc.name ?: "Ziggy MVC");
        rc.timeOfDay = greetingService.getTimeOfDay();
    }

}
```

Update your view:

```html
<h1><cfoutput>#rc.greeting#</cfoutput></h1>
<p>Good <cfoutput>#rc.timeOfDay#</cfoutput>!</p>
<p>Try: <a href="?action=main.default&name=Developer">?action=main.default&name=Developer</a></p>
```

## Complete Application Structure

A typical Ziggy MVC application looks like this:

```
/myapp
  /controllers
    main.cfc
    user.cfc
  /layouts
    default.cfm
  /model
    /beans
      user.cfc
    /services
      userService.cfc
  /views
    /main
      default.cfm
      error.cfm
    /user
      list.cfm
      view.cfm
      edit.cfm
  Application.cfc
  index.cfm
```

## Basic Configuration

Configure Ziggy MVC via the `variables.framework` struct in Application.cfc:

```cfml
component extends="framework.ziggy" {
    this.name = "MyApp";
    this.sessionManagement = true;

    variables.framework = {
        // Reload password (use ?reload=password to reload app)
        reload = "reload",
        password = "secret",

        // Enable SES URLs
        generateSES = true,
        SESOmitIndex = false,

        // Development settings
        reloadApplicationOnEveryRequest = false,
        trace = false,

        // Default action
        home = "main.default",

        // Error handler
        error = "main.error",

        // Dependency injection
        diEngine = "di1",
        diLocations = "model,controllers"
    };
}
```

## Next Steps

Now that you have the basics, explore these topics:

- [Developing Applications](/documentation/developing-applications/) - In-depth guide to building Ziggy MVC applications
- [Using DI/1](/documentation/using-di-one/) - Dependency injection for clean, testable code
- [Using AOP/1](/documentation/using-aop-one/) - Aspect-oriented programming for cross-cutting concerns
- [Using Subsystems](/documentation/using-subsystems/) - Modularize large applications
- [Reference Manual](/documentation/reference-manual/) - Complete API reference

## Migrating from FW/1

If you're migrating an existing FW/1 application to Ziggy MVC:

1. Replace the `framework` folder with Ziggy MVC's framework folder
2. Change your Application.cfc to extend `framework.ziggy` instead of `framework.one`:

**Before (FW/1):**
```cfml
component extends="framework.one" {
```

**After (Ziggy MVC):**
```cfml
component extends="framework.ziggy" {
```

That's it! Your FW/1 application should work identically on Ziggy MVC.

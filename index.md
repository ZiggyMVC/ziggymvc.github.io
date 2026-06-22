---
layout: default
title: Ziggy MVC - A Lightweight MVC Framework for CFML
---

# Ziggy MVC

**Ziggy MVC** is a family of small, lightweight, convention-over-configuration frameworks for CFML. It provides:

- **Ziggy MVC** - MVC framework with controllers, views, layouts, and URL routing
- **DI/1** - Dependency Injection (Inversion of Control) container
- **AOP/1** - Aspect-Oriented Programming support

## About Ziggy MVC

Ziggy MVC version 5.0.0 is a community-driven fork of Framework-1 (FW/1), created by [South of Shasta](https://southofshasta.com/) and other members of the CFML community. Development on the original FW/1 had stalled, so we created a fresh start while preserving full backward compatibility with FW/1.

**Ziggy MVC is 100% backward compatible with FW/1.** Your existing FW/1 applications will work as-is with Ziggy MVC - you just need to change `framework.one` to `framework.ziggy` in your Application.cfc file.

### Version History

| Version | Description |
|---------|-------------|
| **5.0.0** | Initial Ziggy MVC release (forked from FW/1 4.3.0) |
| FW/1 4.3.0 | Original Framework-1 codebase |

## Quick Start

### Installation

<!-- **Via CommandBox:**
```bash
box install ziggymvc
``` -->

**Via GitHub:**
Download from [https://github.com/ZiggyMVC/ZiggyMVC](https://github.com/ZiggyMVC/ZiggyMVC)

### Minimum Requirements

- **Adobe ColdFusion** 10 or later
- **Lucee** 4.5.0 or later (Lucee 5.x recommended)

### Your First Application

Create these files in your webroot:

**Application.cfc:**
```cfml
component extends="framework.ziggy" {
    this.name = "MyApp";
}
```

**index.cfm:**
```cfml
<!--- This file intentionally left empty --->
```

**views/main/default.cfm:**
```html
<h1>Hello, Ziggy MVC!</h1>
<p>Welcome to your first Ziggy MVC application.</p>
```

Visit your application in a browser, and you'll see your first Ziggy MVC page!

## Features

- **Convention over Configuration** - Minimal setup required; follows sensible defaults
- **MVC Architecture** - Clean separation of models, views, and controllers
- **Dependency Injection** - Built-in DI/1 container with automatic bean discovery
- **Subsystems** - Modularize large applications into mini-applications
- **URL Routes** - Flexible routing with support for SES URLs and REST
- **REST API Support** - Built-in JSON, XML, and custom rendering
- **Layouts** - Cascading layout system with view-specific overrides
- **Lightweight** - Single-file architecture with minimal overhead

## Documentation

- [Getting Started Guide](/documentation/)
- [Developing Applications](/documentation/developing-applications/)
- [Using DI/1](/documentation/using-di-one/)
- [Using AOP/1](/documentation/using-aop-one/)
- [Using Subsystems](/documentation/using-subsystems/)
- [Reference Manual](/documentation/reference-manual/)

## Community

- **GitHub:** [https://github.com/ZiggyMVC/ZiggyMVC](https://github.com/ZiggyMVC/ZiggyMVC)
- **Issues:** [https://github.com/ZiggyMVC/ZiggyMVC/issues](https://github.com/ZiggyMVC/ZiggyMVC/issues)

## License

Ziggy MVC is released under the [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0).

- Framework-1: Copyright (c) 2009-2018, Sean Corfield (and others)
- Ziggy MVC: Copyright (c) 2026, South of Shasta (and others)

Special thanks to Sean Corfield, the original creator of FW/1, and to all the contributors who made this framework possible.

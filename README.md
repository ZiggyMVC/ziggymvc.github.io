# Ziggy MVC Documentation

This repository contains the documentation website for [Ziggy MVC](https://github.com/ZiggyMVC/ZiggyMVC), a lightweight MVC framework for CFML.

## About Ziggy MVC

Ziggy MVC is a community-driven fork of Framework-1 (FW/1), created by [South of Shasta](https://southofshasta.com/) and other members of the CFML community. Version 5.0.0 is the initial release, based on FW/1 4.3.0.

Ziggy MVC is 100% backward compatible with FW/1 applications.

## Documentation

Visit the live documentation at: [https://ziggymvc.github.io](https://ziggymvc.github.io)

### Documentation Sections

- **Getting Started** - Installation and first application
- **Developing Applications** - Controllers, views, layouts, configuration
- **Using DI/1** - Dependency injection
- **Using AOP/1** - Aspect-oriented programming
- **Using Subsystems** - Modular application architecture
- **Reference Manual** - Complete API reference

## Building Locally

This site uses Jekyll (GitHub Pages default). To build locally:

```bash
# Install Jekyll
gem install bundler jekyll

# Clone the repository
git clone https://github.com/ZiggyMVC/ziggymvc.github.io.git
cd ziggymvc.github.io

# Install dependencies
bundle install

# Serve locally
bundle exec jekyll serve
```

Visit `http://localhost:4000` to view the site.

## Contributing

Contributions to the documentation are welcome! Please submit pull requests to this repository.

## License

This documentation is released under the same Apache 2.0 license as Ziggy MVC.

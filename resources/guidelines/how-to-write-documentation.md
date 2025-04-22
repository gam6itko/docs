# Spiral Framework Documentation Guidelines

This document provides guidelines for writing documentation for the Spiral Framework. These guidelines ensure consistency and high quality across all documentation.

## Document Structure

### Heading Hierarchy

- Use a single `#` for main titles (e.g., "HTTP — Routing")
- Use `##` for section headings (e.g., "Installation")
- Use `###` for subsections (e.g., "Defining Routes")
- Use `####` for minor subsections when needed

### Code Examples

- Always use fenced code blocks with language identifiers: ```php
- Include file paths as comments above code blocks: ```php app/src/Application/Kernel.php
- Include complete examples that demonstrate functionality
- Add comments for complex logic or important points
- For PHP classes, show both imports and method implementations

### Admonitions

Use blockquotes with labels for notes, warnings, and other admonitions:

```markdown
> **Note**
> This is an important note about the functionality.

> **Warning**
> Be careful about this potential issue.

> **Danger**
> Critical information about dangerous operations.

> **See more**
> Reference to other documentation sections.
```

### Tabs

Use tab containers for alternative approaches or implementations:

```markdown
:::: tabs

::: tab Using method
// First approach content
:::

::: tab Using constant
// Alternative approach content
:::

::::
```

### Tables and Lists

#### Tables

Format tables with proper headers and alignment:

```markdown
| Property   | Type         | Description                             |
|------------|--------------|---------------------------------------- |
| route      | string       | The route pattern. **Required**         |
| name       | string       | Route name. **Optional**                |
| methods    | array/string | The HTTP methods for the route.         |
```

Use tables for:
- Parameter descriptions
- Configuration options
- Component comparisons
- Feature availability
- Event listings

#### Lists

- Use bullet lists for related but independent items
- Use numbered lists for sequential steps or procedures
- Keep list items parallel in structure
- Indent sublists properly for hierarchy

## Content Style

### Introductions

- Begin each document with a brief explanation of the component's purpose
- Explain the problem the component solves
- Keep introductions concise but informative

### Installation Instructions

- Always include composer commands for installation
- Show bootloader registration in both styles (method and constant)
- Include explanatory text between code blocks

### Configuration

- Show complete configuration files with comments
- Explain key configuration options
- Use tables for parameter descriptions
- Document environment variable integration

### Progressive Examples

- Structure content from basic to advanced usage
- Build on previous examples when introducing new concepts
- Provide real-world use cases

### Cross-References

- Use relative links to other documentation sections
- Include "See more" admonitions for related topics

## Writing Style

### Conversational Tone

- Use a friendly, conversational style
- Address the reader directly ("you can...")
- Balance technical precision with approachability

### Technical Precision

- Use correct and consistent terminology
- Be explicit about class names, methods, and parameters
- Avoid ambiguity in technical instructions

### Completeness

- Cover all common use cases for each component
- Explain both simple and advanced usage patterns
- Include error handling and edge cases

### Formatting Patterns

- Use backticks for inline code, class names, methods
- Use bold for important concepts and terms
- Use tables for comparing options or parameters

## Component-Specific Guidelines

### HTTP & Routing

- Show both attribute-based and configurator-based routing
- Explain route patterns and parameters in detail
- Include middleware configuration examples

### Database & ORM

- Show repository pattern usage
- Explain transaction handling
- Include entity validation examples

### Console Commands

- Show command registration and implementation
- Include examples of input/output handling
- Explain available options and arguments

### Configuration and Bootloaders

- Explain the component configuration structure
- Show bootloader registration examples
- Explain component dependencies
- Document configuration file structure
- Show environment variable integration

### Middleware & Interceptors

- Explain the request lifecycle
- Show both class and closure-based implementations
- Explain common interceptor use cases
- Include middleware ordering information
- Document how to register middleware globally vs. per-route

### Events

- List available events in a table format
- Show event listener implementation
- Explain event dispatching patterns

### View Templates

- Show different template engine options
- Explain context and data passing
- Include examples of common template patterns

## Terminal Commands and Output

### Commands

- Use `terminal` syntax highlighting for commands: ```terminal
- Include explanation of command options

### Command Output

- Use `output` syntax highlighting for command output: ```output
- Show realistic output examples

## Events Section

Each component should include an Events section at the end listing all dispatched events:

```markdown
## Events

| Event                             | Description                                     |
|-----------------------------------|-------------------------------------------------|
| Spiral\Router\Event\Routing       | Fired before matching the route.                |
| Spiral\Router\Event\RouteMatched  | Fired when the route is successfully matched.   |
```

Always include a note to the Events documentation:

```markdown
> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
```

## Best Practices

1. **Follow PSR standards** in all PHP code examples
2. **Emphasize modern PHP practices** aligned with Spiral Framework's design principles
3. **Be comprehensive but approachable**, allowing developers to quickly understand and implement Spiral Framework components
4. **Include real use-cases** that demonstrate practical implementation
5. **Document edge cases and potential issues** to help users avoid common pitfalls
6. **Use Dependency Injection examples** to show how to properly integrate components
7. **Show both traditional and prototype-based approaches** where applicable

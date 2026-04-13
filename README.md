# Design Patterns & Principles

Coding standards and design pattern guidelines for Laravel (PHP) backend + Blade / Alpine.js frontend projects. All code produced — whether human-written or AI-generated — must follow these documents.

> **繁體中文版請見 [README.zh-TW.md](./README.zh-TW.md)**

## Documents

| Document | Scope |
|----------|-------|
| [backend-design-patterns.md](./backend-design-patterns.md) | Backend principles, patterns & naming (PHP / Laravel) |
| [frontend(blade)-design-patterns.md](./frontend(blade)-design-patterns.md) | Frontend principles, patterns & naming (Blade / Alpine.js) |

## Architecture

### Backend Layered Architecture

```
Controller → ServiceInterface → Service → RepositoryInterface → Repository → Model
                ↑                              ↑
            ServiceProvider binding (Factory Pattern)
```

| Layer | Responsibility |
|-------|---------------|
| Controller | Receive HTTP request, return HTTP response |
| FormRequest | Input validation |
| ServiceInterface | Business operation contract |
| Service | Business logic, transaction management |
| RepositoryInterface | Data access contract |
| Repository | Data access, query logic |
| Model | Data structure, relationships, scopes, casting |
| ServiceProvider | Bind Interface → Implementation |

### Frontend Layered Architecture

- **Blade components** — Pure presentation layer
- **Alpine components** — Container role: data fetching & state management

## Covered Topics

### Design Principles

- **DRY** — Single source of truth for every piece of knowledge
- **SOLID** — SRP, OCP, LSP, ISP, DIP with Laravel-specific examples
- Progressive Enhancement, State Minimization, Separation of Concerns

### Design Patterns

| Pattern | Usage |
|---------|-------|
| Repository | Data access abstraction (architecturally required) |
| Null Object | Safe default when a dependency may not exist |
| Strategy | Multiple implementations of the same operation |
| Adapter | Bridge incompatible interfaces |
| Factory | Centralized object construction (ServiceProvider) |
| Template Method | Shared process skeleton via Traits |
| Event Delegation | Dynamic list event handling |
| Observer | Cross-component communication via CustomEvent |
| State Machine | Mutually exclusive UI states |
| Debounce / Throttle | High-frequency event control |

### Naming Conventions

Comprehensive naming rules for PHP classes, methods, variables, database tables/columns, routes, JavaScript, HTML/Blade, and file naming.

## Tech Stack

| Category | Technology |
|----------|-----------|
| Backend Language | PHP |
| Backend Framework | Laravel |
| Frontend Template | Blade |
| Frontend JS | Alpine.js |

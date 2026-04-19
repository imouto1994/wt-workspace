# Toonwide Workspace — Agent Guidelines

This is a multi-project workspace. Before working on any project, read the relevant project-level `AGENTS.md` for project-specific rules.

## Projects

| Folder | What it is | Agent docs |
|--------|-----------|------------|
| `wt-web-app/` | Web app (TanStack Start + React + Drizzle + Better Auth) | [wt-web-app/AGENTS.md](./wt-web-app/AGENTS.md) |
| `wt-split-desktop-app/` | Desktop app (Electron + React + Sharp + oRPC) | [wt-split-desktop-app/AGENTS.md](./wt-split-desktop-app/AGENTS.md) |
| `webtoon-prototype/` | Legacy prototype (VanillaJS Electron) — read-only reference, do not modify | [webtoon-prototype/docs/ELECTRON_APP.md](./webtoon-prototype/docs/ELECTRON_APP.md) |

## Shared Artifact

- **[PRODUCT_SPEC.md](./PRODUCT_SPEC.md)** — Product specification shared across projects. Update it when user-facing features, data models, or UI behavior change in any project.

---

## Mandatory: Good Practices (All Projects)

The following rules apply **regardless of which project you're working in**.

### 1. Documentation Updates

After every change, update the relevant project's documentation before finishing your task. Each project's `AGENTS.md` specifies exactly which docs to update and what triggers an update. Skipping this causes future agents to work from stale information.

### 2. Inline Code Comments

**Every file you create or substantially modify MUST have clear comments:**

- **Top-level module comment** — A JSDoc/block comment at the top of every file explaining its purpose, responsibilities, and how it fits into the architecture.
- **Function/component doc comments** — Every exported function, component, or class gets a JSDoc comment explaining what it does, its key parameters, and any non-obvious return values.
- **Inline "why" comments** — Explain non-obvious logic: trade-offs, workarounds, business rules, algorithm choices. Do NOT restate what the code obviously does.
- **NEVER remove existing comments that are still correct.** When refactoring or rewriting code, preserve every inline comment that still accurately describes the logic it annotates. Only remove a comment if the code it describes has been deleted or the comment is factually wrong. Losing a correct comment means losing context that future agents depend on.

### 3. TypeScript Conventions

- Avoid type casting (`as`, manual generics) — prefer inference from schemas and return types.
- Fix types at the source (schema, function signature) rather than casting at the point of use.
- Prefix generic type parameters with `T` (e.g., `TData`, `TReturn`).

### 4. Commit Discipline

- Write clear, concise commit messages that explain the "why" not just the "what."
- Don't commit generated files, build artifacts, or secrets.
- Keep commits focused — one logical change per commit.

### 5. Prefer shadcn/ui Wrappers Over Raw Primitives

Both projects use shadcn/ui as the component library layer. Always prefer importing from `@/components/ui/*` (the shadcn wrappers) over importing directly from `@base-ui/react/*` or `@radix-ui/*`. If a shadcn component exists for the pattern you need, install it via the project's shadcn CLI and use it. Only use raw primitives when no shadcn wrapper exists for the specific pattern (e.g., custom full-viewport dialog styling that conflicts with the standard `DialogContent`).

### 6. Cross-Project Awareness

When making changes that affect the product spec or shared behavior:
- Check if the change impacts other projects in the workspace.
- Update [PRODUCT_SPEC.md](./PRODUCT_SPEC.md) if user-facing behavior changed.
- Note cross-project dependencies in commit messages when relevant.

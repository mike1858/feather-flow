---
name: feather:setup-convex-testing
description: "Checklist: Is Convex test infrastructure set up? Verify or create convex-test deps, vitest Convex settings, convex/test.setup.ts. Add-on to feather:setup-react-testing — run that first."
---

# Set Up Convex Testing

Verify or create the Convex-specific test infrastructure for React + Convex integration testing using [feather-testing-convex](https://www.npmjs.com/package/feather-testing-convex).

**Dual-purpose:** Use this checklist to set up Convex testing from scratch OR to diagnose why existing Convex tests won't run.

**Prerequisite:** `/feather:setup-react-testing` must be completed first. This skill adds Convex-specific pieces on top of the React testing environment.

## ⚠️ Philosophy: Integration Tests First

**Do NOT write separate backend unit tests and mocked component tests.** The feather-testing-convex library enables true integration tests that test both layers together.

### ❌ Anti-Pattern: Isolated Unit Tests
```tsx
// DON'T: Backend-only test
test("todos.list returns data", async () => {
  const t = convexTest(schema, modules);
  // ... 15 lines of setup
  const todos = await t.query(api.todos.list, {});
  expect(todos).toHaveLength(1);
});

// DON'T: Component test with mocked backend
vi.mock("convex/react", () => ({ useQuery: vi.fn() }));
test("TodoList renders", () => {
  vi.mocked(useQuery).mockReturnValue([{ text: "Buy milk" }]);
  render(<TodoList />);
  expect(screen.getByText("Buy milk")).toBeInTheDocument();
});
```

### ✅ Correct Pattern: Integration Tests
```tsx
// DO: One test covers both backend + component
test("shows seeded data", async ({ client, seed }) => {
  await seed("todos", { text: "Buy milk", completed: false });
  renderWithConvex(<TodoList />, client);
  expect(await screen.findByText("Buy milk")).toBeInTheDocument();
});
```

**Use mocks ONLY for:** loading spinners, error states — transient states that can't be produced with a real backend.

---

## Checklist

### ☐ 0. Prerequisite: React testing environment set up

**Check:** Verify `/feather:setup-react-testing` has been completed:
- `vitest.config.ts` exists with `plugins: [react()]` and `environment: "jsdom"`
- `src/test/setup.ts` (or `src/test-setup.ts`) exists with jest-dom matchers
- `@testing-library/react`, `jsdom`, `@vitejs/plugin-react` are installed

**If not done:** Run `/feather:setup-react-testing` first, then return here.

### ☐ 1. Convex testing dependencies installed

**Check:**
```bash
npm ls convex-test feather-testing-convex
```

**If missing:**
```bash
npm install -D convex-test feather-testing-convex
```

### ☐ 2. vitest.config.ts has Convex-specific settings

**Check for these settings (in addition to what `feather:setup-react-testing` already set up):**
- `environmentMatchGlobs: [["convex/**", "edge-runtime"]]`
- `server: { deps: { inline: ["convex-test"] } }`

**If missing, add them to the existing vitest.config.ts `test` section:**
```typescript
test: {
  // ... existing settings from feather:setup-react-testing
  environmentMatchGlobs: [["convex/**", "edge-runtime"]],
  server: { deps: { inline: ["convex-test"] } },
},
```

**Complete config for reference** (combining React testing + Convex):
```typescript
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    environmentMatchGlobs: [["convex/**", "edge-runtime"]],
    server: { deps: { inline: ["convex-test"] } },
    globals: true,
    setupFiles: ["./src/test-setup.ts"],
  },
});
```

**Common errors:**
- Missing `inline: ["convex-test"]` → `Cannot find module 'convex-test'`
- Missing `environmentMatchGlobs` → Convex functions fail because they run in jsdom instead of edge-runtime

### ☐ 3. convex/test.setup.ts exists with correct exports

**Check for:**
- `modules` glob export
- `createConvexTest(schema, modules)` → `test` export
- `renderWithConvex` re-export

**If missing, create:**
```typescript
/// <reference types="vite/client" />
import { createConvexTest, renderWithConvex } from "feather-testing-convex";
import schema from "./schema";

export const modules = import.meta.glob("./**/!(*.*.*)*.*s");
export const test = createConvexTest(schema, modules);
export { renderWithConvex };
```

### ☐ 4. At least one test passes

**Run:**
```bash
npx vitest run
```

**If no tests exist, create one to verify the setup.**

Pick one example below. You don't need both.

#### Backend-only test (query + seed)

```typescript
// src/components/TodoList.test.ts
import { describe, expect } from "vitest";
import { test } from "../../convex/test.setup";
import { api } from "../../convex/_generated/api";

describe("Todos", () => {
  test("seed and query", async ({ client, seed }) => {
    await seed("todos", { text: "Buy milk", completed: false });

    const todos = await client.query(api.todos.list, {});
    expect(todos).toHaveLength(1);
    expect(todos[0].text).toBe("Buy milk");
  });
});
```

#### Integration test (React component + real backend)

```tsx
// src/components/TodoList.test.tsx
import { describe, expect } from "vitest";
import { screen } from "@testing-library/react";
import { test, renderWithConvex } from "../../convex/test.setup";
import { TodoList } from "./TodoList";

describe("TodoList", () => {
  test("shows seeded data", async ({ client, seed }) => {
    await seed("todos", { text: "Buy milk", completed: false });

    renderWithConvex(<TodoList />, client);

    expect(await screen.findByText("Buy milk")).toBeInTheDocument();
  });
});
```

### ☐ 5. Coverage excludes Convex generated code (optional)

**Check:** `vitest.config.ts` coverage exclusions include `convex/_generated/`.

**If missing, add to coverage.exclude array:**
```typescript
coverage: {
  exclude: [
    // ... existing exclusions
    "convex/_generated/",
  ],
}
```

---

## Fixtures Reference

`createConvexTest(schema, modules, options?)` returns a custom `test` function with these fixtures:

| Fixture | Description |
|---------|-------------|
| `testClient` | Raw convex-test client (unauthenticated) |
| `userId` | ID of an auto-created user (string) |
| `client` | Authenticated client for the auto-created user |
| `seed(table, data)` | Insert a document. Auto-fills `userId` unless `data` includes an explicit `userId` (explicit wins). Returns the document ID. |
| `createUser()` | Create another user, return authenticated client with `.userId` property. |

### Options

```typescript
// Custom users table name (default: "users")
export const test = createConvexTest(schema, modules, { usersTable: "profiles" });
```

### Additional Helpers

- `wrapWithConvex(children, client)` — JSX wrapper for custom rendering
- `renderWithConvex(ui, client)` — Testing Library render with Convex provider

## Query Behavior

Queries run **once** at component mount (one-shot). UI does not re-render after a mutation.

- Assert backend state directly: `await client.query(api.items.list, {})`
- See updated UI: unmount and remount the component
- See `/feather:review-convex-tests` references for workarounds

---

## Next Steps

- **Auth testing** (`<Authenticated>`, `useAuthActions()`, signIn/signOut) → use `/feather:add-convex-auth-testing`
- **Review test quality** (patterns, anti-patterns, best practices) → use `/feather:review-convex-tests`

# Change specification: Remove the User-Agent display from BookCatalog Index

## Metadata

- **Owner:** Workshop application team (@jbrostgov)
- **Status:** Draft
- **Source root:** `legacy-src/dotnet-bookcatalog/shared-legacy-app`
- **Related issue:** not created

## Problem and outcomes

### Problem

Every visitor to the Book Catalog list page (`/Books`) sees a "YOUR BROWSER:"
box that echoes the raw `User-Agent` request header. The page has no business
need for this client data, so the display exposes unnecessary request metadata
(a privacy concern) and couples the `Index` action to `System.Web`
(`HttpContext.Request`), which a modernization would otherwise have to carry
forward.

Evidence:

- `src/BookCatalog.Web/Controllers/BooksController.cs` — `Index()` reads
  `HttpContext.Request.UserAgent` and assigns it to `ViewBag.UserAgent`.
- `src/BookCatalog.Web/Views/Books/Index.cshtml` — the final
  `@if (ViewBag.UserAgent != null)` block renders a `user-agent-box` containing
  the value.

### Desired outcomes

1. The Index page never displays request `User-Agent` data or the
   "YOUR BROWSER" label.
2. `Index()` no longer reads the request or sets `ViewBag.UserAgent`, removing
   that use of `System.Web` from the action.

### Success measures

- A manual check of `/Books` shows no "YOUR BROWSER" box, including when the
  request carries a distinctive `User-Agent` value.
- `msbuild BookCatalog.sln -restore -p:Configuration=Release` exits with code 0.
- The diff contains only the two allowed files.

## Functional requirements

| ID | Requirement | Evidence or rationale |
| --- | --- | --- |
| FR-1 | The system shall continue to return only active books ordered by title from `Index()`. | `BooksController.cs:Index` — `db.Books.Where(b => b.IsActive).OrderBy(b => b.Title).ToList()` |
| FR-2 | `Index()` shall not read `Request.UserAgent` and shall not set `ViewBag.UserAgent`. | Removes the exposure and the `System.Web` request dependency. |
| FR-3 | `Index.cshtml` shall not render a User-Agent or "YOUR BROWSER" block. | Removes the visible exposure. |
| FR-4 | No replacement diagnostic, setting, or flag shall be added. | Maintainer decision: remove, do not gate. |

## Constraints and out of scope

### Constraints

- **Allowed paths:**
  - `legacy-src/dotnet-bookcatalog/shared-legacy-app/src/BookCatalog.Web/Controllers/BooksController.cs`
  - `legacy-src/dotnet-bookcatalog/shared-legacy-app/src/BookCatalog.Web/Views/Books/Index.cshtml`
- **Primary validation command**, run from
  `legacy-src/dotnet-bookcatalog/shared-legacy-app`:
  `msbuild BookCatalog.sln -restore -p:Configuration=Release`
- **Compatibility:** Preserve routes, the `Book` model, the list columns, the
  "Showing N record(s)" footer, the CRUD links, and the classic project
  format.
- **Dependencies:** Add or change no package, `packages.config`, or
  configuration.
- **Data and security:** Add no replacement logging, storage, or display of
  request metadata. Add no secrets or connection-string changes.

### Exclusions (out of scope)

Non-goals and follow-ups, each needing its own specification:

- `Create(Book)` over-posting: restricting binding with
  `[Bind(Include=...)]` so `Id` and `CreatedDate` cannot be posted.
- `DeleteConfirmed` soft delete (`IsActive = false`) and its side effects
  (Details/Edit reachability, re-activation through Edit, Delete view and
  README wording).
- Removing the now-unused `.user-agent-box` and `.user-agent-label` rules from
  `src/BookCatalog.Web/Content/Site.css`.
- An appSettings diagnostic flag, telemetry, a test project, ASP.NET Core
  migration, and broader `System.Web` removal.

## Edge cases

| Type | Condition | Expected behavior |
| --- | --- | --- |
| Normal | Active and inactive books exist | Only active books appear, ordered by title, with View/Edit/Del links. |
| Failure | Request has no `User-Agent` header | Page renders normally with no diagnostic and no error. |
| Boundary | No active books exist | The existing "No books found." box shows; no User-Agent box. |
| Boundary | Request has an empty or very long `User-Agent` value | The value and label are never rendered. |
| Compatibility | Existing routes, model, columns, CRUD links | Behavior is unchanged. |

## Acceptance criteria

### AC-1: Catalog behavior is preserved

- **Given** active and inactive books exist
- **When** a user opens `/Books`
- **Then** only active books appear, ordered by title, with the existing
  actions and footer
- **Verification:** Manual check under IIS Express with LocalDB.

### AC-2: User-Agent display is removed

- **Given** a request carries any `User-Agent` header, including a distinctive
  value
- **When** a user opens `/Books`
- **Then** the response contains neither that value nor the text
  "YOUR BROWSER"
- **Verification:** Manual check of the rendered page (or its page source) and
  a diff review confirming `Request.UserAgent` and `ViewBag.UserAgent` are
  gone from both files.

### AC-3: Build remains valid

- **Given** the documented .NET Framework build prerequisites are installed
- **When** the primary validation command runs from the source root
- **Then** it exits with code 0
- **Verification:** Record the command, exit status, and a concise result.

### AC-4: Diff stays in scope

- **Given** the implementation is complete
- **When** `git diff --name-only` runs
- **Then** it lists only the two allowed files
- **Verification:** Record the command output.

## Non-functional requirements

| ID | Quality | Testable requirement | Verification |
| --- | --- | --- | --- |
| NFR-2 | Security/privacy | No request header value is displayed, logged, or stored by this change. | Diff review. |
| NFR-3 | Maintainability | The change removes code and adds no new dependency or abstraction. | Diff review. |

## Assumptions, dependencies, and risks

| Item | Type | Validation or mitigation |
| --- | --- | --- |
| The display is not relied on by a workshop demonstration or other consumer. | Assumption | Human reviewer confirms before approval; no repository consumer was found in the controller or views read. |
| Windows, Visual Studio MSBuild/web tooling, .NET Framework 4.8 targeting tools, IIS Express, and SQL Server LocalDB are available. | Dependency | Check before implementation; if missing, report the exact prerequisite. |
| NuGet packages may need restoring. | Dependency | If `packages/` is incomplete, run `nuget restore BookCatalog.sln -PackagesDirectory packages -NonInteractive`; never hand-edit `packages/`. |
| Orphaned CSS remains after removal. | Risk | Accepted; listed as a follow-up. |

## Plan-mode review checkpoint

### Evidence

- **Current behavior:**
  `legacy-src/dotnet-bookcatalog/shared-legacy-app/src/BookCatalog.Web/Controllers/BooksController.cs:Index`
  stores `HttpContext.Request.UserAgent` in `ViewBag.UserAgent`;
  `legacy-src/dotnet-bookcatalog/shared-legacy-app/src/BookCatalog.Web/Views/Books/Index.cshtml`
  renders it when non-null. The data layer is `ApplicationDbContext`
  (`src/BookCatalog.Web/Models/ApplicationDbContext.cs`, connection name
  `BookCatalogContext`) and `Book` (`src/BookCatalog.Web/Models/Book.cs`).
- **Existing tests:** none found under `legacy-src/dotnet-bookcatalog`
  (`legacy-src/dotnet-bookcatalog/AGENTS.md` also states there is no automated
  test project).
- **Baseline validation evidence:** `msbuild BookCatalog.sln -restore
  -p:Configuration=Release`, run from
  `legacy-src/dotnet-bookcatalog/shared-legacy-app` with Visual Studio Build
  Tools 2022 (17.14.45): build succeeded. The manual IIS Express/LocalDB smoke
  check has not been run.

### Unresolved questions

- [ ] Confirm that the User-Agent display is not an intentional workshop
      talking point. Owner: maintainer. Decide before approval.
- [ ] Confirm that the build tooling and LocalDB are available on the machine
      that will implement this. Owner: implementer. Decide before approval.

The specification is not ready for approval while a question can change scope,
acceptance criteria, allowed paths, exclusions, or validation.

### Boundary and validation review

- [ ] Evidence citations were checked against the repository.
- [ ] Allowed paths include all anticipated implementation and test changes and
      nothing unrelated.
- [ ] The primary validation command is safe, non-interactive, and runnable
      from the stated source root.
- [ ] Acceptance criteria cover the required normal, failure, boundary, and
      compatibility behavior.
- [ ] Exclusions prevent unrelated modernization or cleanup.

### Stop conditions

Stop and request human guidance if:

- repository evidence contradicts this specification;
- an unresolved question could materially change behavior or scope;
- satisfying a criterion requires a path outside the allowed paths;
- the baseline or primary validation fails for a reason caused by the proposed
  change, or the command is missing, destructive, or irreproducible;
- a missing prerequisite prevents required evidence from being collected. A
  build without the manual check is reported as "build only, not fully
  validated", never as passed;
- a secret, credential, production resource, destructive action, dependency
  change, schema change, or public API change is required but not explicitly
  approved; or
- implementation or validation cannot satisfy the acceptance criteria without
  expanding the approved contract.

### Human approval

- [ ] I reviewed the evidence, unresolved questions, allowed paths, acceptance
      criteria, validation command, exclusions, risks, and stop conditions.
- **Reviewer:** pending
- **Approval date:** pending
- **Approved specification commit:** pending

Change **Status** to `Approved` only after the checkbox is selected and the
reviewer, date, and commit fields are complete. Planning approval authorizes
issue preparation, not implementation.

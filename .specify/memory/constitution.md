<!--
  Sync Impact Report
  ==================================================
  Version change: N/A → 1.0.0 (initial ratification)
  Modified principles: N/A (initial creation)
  Added sections:
    - Core Principles (5 principles)
    - Technical Standards
    - Spec-Driven Development Workflow
    - Governance
  Removed sections: N/A
  Templates requiring updates:
    - .specify/templates/plan-template.md ✅ no update needed
      (Constitution Check section uses dynamic gate reference)
    - .specify/templates/spec-template.md ✅ no update needed
    - .specify/templates/tasks-template.md ✅ no update needed
    - .specify/templates/checklist-template.md ✅ no update needed
  Follow-up TODOs: None
  ==================================================
-->

# ContosoDashboard Constitution

## Core Principles

### I. Offline-First Architecture

All features MUST function without cloud services or external
dependencies. The application MUST run on a local machine with
no internet connectivity required.

- Every infrastructure dependency (storage, database,
  authentication) MUST have a local implementation that works
  offline.
- Interface abstractions MUST be defined for all infrastructure
  services to enable future cloud migration (e.g.,
  `IFileStorageService`, `IUserService`).
- Swapping from local to cloud implementations MUST require only
  dependency injection configuration changes—no business logic
  modifications.
- Connection strings and service endpoints MUST be externalized
  in configuration files (`appsettings.json`).

**Rationale**: This is a training application. Students MUST be
able to run and learn from it without Azure subscriptions, cloud
accounts, or network access.

### II. Defense in Depth Security

Security MUST be enforced at multiple independent layers. No
single layer failure may expose unauthorized data.

- Authentication MUST be enforced via middleware on all
  non-public routes.
- Authorization MUST be declared on every protected page using
  `[Authorize]` attributes.
- Service-level authorization checks MUST prevent Insecure
  Direct Object Reference (IDOR) vulnerabilities—every service
  method accessing user-scoped data MUST verify the caller has
  permission.
- User isolation is mandatory: each user MUST see only their
  authorized data.
- Security headers (CSP, X-Frame-Options, X-XSS-Protection)
  MUST be configured.
- File uploads MUST use GUID-based filenames, validate against
  an extension whitelist, and store files outside `wwwroot`.

**Rationale**: Even in a training context, the codebase MUST
demonstrate production-grade security patterns so students learn
correct habits.

### III. Separation of Concerns

The application MUST maintain a clean layered architecture with
distinct responsibilities per layer.

- **Models**: Entity definitions and data annotations only. No
  business logic.
- **Services**: All business logic MUST reside in service classes
  behind interfaces. Services MUST NOT reference UI components.
- **Data**: EF Core `DbContext` with relationships, indexes, and
  seed data. No business rules in the data layer.
- **Pages/UI**: Razor/Blazor components handle presentation and
  user interaction only. Pages MUST delegate all logic to
  services via dependency injection.
- Cross-layer coupling is prohibited. A page MUST NOT directly
  access `DbContext`; it MUST go through a service.

**Rationale**: Clean separation enables independent testing,
cloud migration of individual layers, and teaches students
industry-standard architecture patterns.

### IV. Interface-Driven Abstractions

Every infrastructure dependency MUST be accessed through an
interface abstraction registered via dependency injection.

- Services MUST be defined as interfaces (e.g.,
  `ITaskService`, `IProjectService`, `INotificationService`)
  with concrete implementations.
- Infrastructure services (file storage, email, external APIs)
  MUST define an interface with local and cloud implementation
  options.
- New features MUST NOT introduce direct dependencies on
  infrastructure libraries in business logic code.
- Swapping implementations MUST be achievable by changing DI
  registration in `Program.cs` only.

**Rationale**: Interface abstractions are the core mechanism
enabling the offline-first to cloud migration path and make the
codebase testable via mocking.

### V. Training Clarity

Code MUST demonstrate good software engineering practices in a
simplified, educational context.

- Known limitations and training-only shortcuts MUST be
  documented in comments, README, or inline documentation.
- Mock implementations (e.g., mock authentication) MUST follow
  the same architectural patterns as their production
  counterparts so students learn transferable skills.
- Every feature MUST include a clear explanation of what would
  differ in a production deployment.
- Code complexity MUST be justified. If a simpler approach
  teaches the same concept, the simpler approach MUST be used.
- Database conventions MUST be consistent: integer primary keys
  across all entities, text values for categorical data.

**Rationale**: The primary purpose of this codebase is education.
Clarity and correctness of patterns take priority over
production-scale optimizations.

## Technical Standards

- **Framework**: ASP.NET Core with Blazor Server.
- **Database**: SQL Server LocalDB with Entity Framework Core.
  Integer primary keys for all entities. Text-based category
  fields for simplicity.
- **Authentication**: Cookie-based mock authentication for
  training. Claims-based identity with role-based access control
  (Employee, TeamLead, ProjectManager, Administrator).
- **Styling**: Bootstrap 5.3 with Bootstrap Icons. No additional
  CSS frameworks without justification.
- **File Storage**: Local filesystem via `IFileStorageService`.
  Files MUST be stored outside `wwwroot` with authorization-gated
  download endpoints.
- **Configuration**: All environment-specific values MUST reside
  in `appsettings.json` / `appsettings.{Environment}.json`.
- **Naming**: C# naming conventions (PascalCase for public
  members, camelCase for locals). Razor pages named after their
  feature (e.g., `Tasks.razor`, `Projects.razor`).

## Spec-Driven Development Workflow

All new features MUST follow the Spec-Driven Development (SDD)
process using GitHub Spec Kit:

1. **Specify**: Create a feature specification from stakeholder
   requirements using the spec template. Specifications MUST
   include user stories with priorities, acceptance scenarios,
   and measurable success criteria.
2. **Plan**: Generate an implementation plan from the spec.
   The plan MUST pass the Constitution Check gate before
   implementation proceeds.
3. **Tasks**: Break the plan into dependency-ordered,
   independently testable tasks organized by user story.
4. **Implement**: Execute tasks in dependency order. Each user
   story MUST be independently functional at its checkpoint.
5. **Review**: Validate the delivered feature against the
   original spec's acceptance scenarios and success criteria.

Deviations from the spec MUST be documented and justified before
implementation.

## Governance

This constitution is the authoritative source of architectural
and process standards for ContosoDashboard. It supersedes
conflicting guidance in any other document.

- **Amendment procedure**: Any change to this constitution MUST
  be documented with a version bump, rationale, and sync impact
  assessment across dependent templates.
- **Versioning policy**: MAJOR.MINOR.PATCH semantic versioning.
  MAJOR for principle removals or incompatible redefinitions;
  MINOR for new principles or material expansions; PATCH for
  clarifications and wording fixes.
- **Compliance review**: All pull requests and code reviews MUST
  verify compliance with these principles. The plan template's
  Constitution Check gate MUST be satisfied before implementation
  begins.
- **Complexity justification**: Any architectural complexity
  beyond what is prescribed here MUST be justified in the
  plan's Complexity Tracking section.

**Version**: 1.0.0 | **Ratified**: 2026-04-05 | **Last Amended**: 2026-04-05

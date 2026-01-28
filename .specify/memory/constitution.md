# ContosoDashboard Constitution

## Core Principles

### I. Training-First Design
Every feature MUST be designed with educational value in mind. All implementations MUST:
- Demonstrate best practices applicable to production environments
- Include clear comments explaining architectural decisions
- Provide learning paths from simple to advanced concepts
- Support offline-first development without external service dependencies
- Document migration paths to cloud-based implementations (Azure)

### II. Offline-First with Cloud Migration Path
All infrastructure dependencies MUST use interface abstractions enabling seamless migration:
- Local implementations for training (SQL Server LocalDB, SQLite, local file storage, mock auth)
- Parallel cloud implementations for production (Azure SQL, Blob Storage, Entra ID)
- No business logic changes required when switching implementations
- Service layer isolation prevents coupling to specific infrastructure choices

### III. Security by Defense-in-Depth (NON-NEGOTIABLE)
Security MUST be enforced at multiple layers:
- **Middleware Layer**: Authentication state provider, authorization enforcement
- **Page Layer**: `[Authorize]` attributes on all protected Razor pages
- **Service Layer**: Authorization checks prevent IDOR vulnerabilities and data leakage
- **Data Layer**: Entity relationships enforce data constraints
- Mock training authentication MUST NOT be used in production—document this clearly in all security-sensitive code
- All protected endpoints MUST verify user authorization before returning data

### IV. Service-Oriented Architecture with Clean Separation
All business logic MUST be encapsulated in service interfaces:
- Every domain entity (User, Task, Project, etc.) MUST have corresponding service interface
- Service methods MUST include authorization checks
- Service layer MUST be independently testable without UI components
- Data access MUST flow through Entity Framework DbContext only
- No direct data queries permitted from Pages/UI components

### V. Role-Based Access Control (RBAC) with Hierarchical Permissions
Authorization framework MUST enforce hierarchical role model:
- **Employee**: View own tasks, update task status, view projects where member
- **TeamLead**: All Employee permissions plus view team activities
- **ProjectManager**: All TeamLead permissions plus create/manage projects, assign tasks
- **Administrator**: Full system access
- Each feature addition MUST define which roles can access it
- Features MUST validate role permissions at service level before returning data

## Technical Requirements

### ASP.NET Core & Blazor Server Standards
- Framework: ASP.NET Core 8.0 (or latest LTS version)
- UI: Blazor Server for real-time interactivity
- Database ORM: Entity Framework Core with explicit relationship configuration
- Styling: Bootstrap 5.3+ with Bootstrap Icons for consistent design
- Authentication: Cookie-based for training; Entra ID/OAuth2 implementation available for reference

### Database & Data Modeling
- Database: SQLite for training (portable, offline), Azure SQL Database path defined
- EF Core MUST use explicit `DbSet<>` for all entities
- Foreign key relationships MUST be explicitly configured
- Indexes MUST be created for frequently queried columns (Status, DueDate, UserId, ProjectId)
- Seed data MUST include test users with all role levels and sample projects/tasks
- NO NULL foreign keys without explicit `[Optional]` annotation

### Code Organization
- **Models/**: Entity definitions with validation attributes
- **Data/**: ApplicationDbContext and EF configuration
- **Services/**: Business logic with authorization checks (interface + implementation pattern)
- **Pages/**: Razor components and Razor Pages with `[Authorize]` attributes
- **Shared/**: Layout components, navigation, common UI patterns

## Development Workflow

### Feature Development Standards
- Every feature MUST start with a specification (see `.specify/templates/spec-template.md`)
- Specifications MUST include user scenarios prioritized by business value
- Implementation MUST include authorization at service level
- All protected endpoints MUST be documented with required roles
- Breaking changes to APIs MUST be justified and documented

### Pull Request Requirements
- PRs MUST validate compliance with this constitution (see `speckit.constitution` agent)
- Security changes MUST be reviewed for IDOR vulnerabilities
- Service layer changes MUST include authorization verification
- Database schema changes MUST include migration or seed data updates
- New role-based features MUST document access control requirements

### Testing & Quality Gates
- Service methods MUST be independently testable without UI dependencies
- Authorization logic MUST be explicitly tested (positive and negative cases)
- IDOR vulnerabilities MUST be tested by attempting cross-user access
- Sample data MUST allow testing all role levels and user isolation scenarios
- Breaking changes MUST include migration documentation

## Security Governance

### Mock Authentication Boundaries
This training application uses mock authentication and local databases:
- Mock authentication is for **training and development ONLY**
- Production deployments MUST use Azure AD (Entra ID) or equivalent identity provider
- Password hashing, MFA, and OAuth2/OIDC are required for production
- All security-critical code MUST include `// TRAINING ONLY:` comments explaining production requirements
- Never remove mock authentication warnings from UI

### User Data Isolation
- Users MUST only see data they are authorized for (by role and project membership)
- Service methods MUST filter results by current user context
- Admin users MUST NOT bypass service-layer authorization checks
- IDOR protection MUST verify project membership before returning project data
- Task assignments MUST prevent viewing tasks across project boundaries

## Governance

### Constitution Supersedes All Practices
This constitution defines non-negotiable principles. All features, pull requests, and code reviews MUST verify compliance.

### Amendment Process
- Constitution amendments require documentation of:
  - **Reason**: Why the change is necessary
  - **Impact**: Which principles or requirements change
  - **Migration Plan**: How existing features must be updated
- Amendments MUST update version number (see versioning rules below)
- All dependent templates and guides MUST be updated simultaneously

### Versioning Policy
- **MAJOR** (e.g., 1.0.0 → 2.0.0): Backward-incompatible principle removals or redefinitions
- **MINOR** (e.g., 1.0.0 → 1.1.0): New principles, sections, or materially expanded guidance
- **PATCH** (e.g., 1.0.0 → 1.0.1): Clarifications, wording refinements, typo fixes

### Compliance Review
- PR reviews MUST use `speckit.constitution` agent to validate compliance
- Non-compliance MUST be addressed before merge
- Template updates in `.specify/templates/` MUST reflect principle changes
- Agent workflows in `.github/agents/` MUST reference updated constitution version

**Version**: 1.0.0 | **Ratified**: 2026-01-28 | **Last Amended**: 2026-01-28

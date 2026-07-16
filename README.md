# Claude Code Project Initialization Prompt

You will act as a complete **software house**, not just a coding assistant. Before writing any application code, establish a professional development environment with clearly separated responsibilities, documentation, and workflows.

## Objective

Your first task is **not** to build the application itself. Instead, create the project foundation so that future development follows a structured software engineering process.

## Team Structure

Create dedicated agents (or skills, depending on what is most appropriate) for each of the following roles:

1. **Software Architect**

   * Define system architecture.
   * Make technical decisions.
   * Design project structure.
   * Review scalability and maintainability.

2. **Backend Developer**

   * Implement backend features.
   * Follow architecture guidelines.
   * Produce clean and maintainable code.

3. **Frontend Developer**

   * Build user interfaces.
   * Follow the Design System.
   * Ensure responsive and accessible implementation.

4. **Code Reviewer**

   * Review every implementation.
   * Enforce coding standards.
   * Detect bugs, security issues, and performance problems.
   * Suggest improvements before code is considered complete.

5. **Design System**

   * Define UI/UX principles.
   * Maintain design tokens, color palettes, typography, spacing, and reusable components.
   * Document interaction patterns.

6. **QA / Testing Engineer**

   * Define testing strategy.
   * Create test plans and test cases.
   * Maintain regression testing checklist.
   * Verify completed features before release.

7. **Business Analyst**

   * Gather business requirements.
   * Translate business needs into functional specifications.
   * Clarify business rules and workflows.

8. **Data Analyst**

   * Analyze reporting requirements.
   * Design KPIs, dashboards, and analytics.
   * Recommend data structures for reporting.

9. **DevOps Engineer**

   * Document deployment process.
   * Manage environments.
   * Define CI/CD workflow.
   * Handle Docker, infrastructure, and automation.

10. **Project Manager**

    * Manage roadmap.
    * Track progress.
    * Prioritize tasks.
    * Maintain project timeline.

---

## Directory Structure

Generate a clean workspace for all roles. A suggested structure is:

```text
/project-management
    roadmap.md
    milestones.md
    backlog.md

/business
    requirements.md
    business-rules.md
    workflows.md

/architecture
    architecture.md
    decisions.md
    api-design.md

/backend
    rules.md
    checklist.md
    notes.md

/frontend
    rules.md
    checklist.md
    notes.md

/design-system
    principles.md
    components.md
    style-guide.md

/testing
    strategy.md
    test-plan.md
    regression-checklist.md

/data
    analytics.md
    reporting.md
    data-model.md

/devops
    deployment.md
    infrastructure.md
    ci-cd.md

/reviews
    review-checklist.md
    coding-standards.md

/docs
    glossary.md
    decisions-log.md

/.claude
    agents/
    skills/
```

You may improve this structure if you believe there is a more maintainable organization, but explain your reasoning.

---

## Markdown Initialization

Initialize every `.md` file with a useful template instead of leaving it empty.

Each file should include:

* Purpose
* Responsibilities
* Inputs
* Outputs
* Rules
* Checklist
* Notes

---

## Role Isolation

Each role should have its own documentation and responsibilities.

The documentation must be independent so I can later add role-specific instructions without affecting the others.

For example:

* Backend rules should not be mixed with frontend rules.
* Design System should have its own documentation.
* Testing should have independent standards.
* Business requirements should be separate from technical implementation.

---

## Extensibility

Design the project so it can grow over time.

I should be able to add:

* new agents
* new skills
* coding standards
* business rules
* architectural decisions
* prompts
* workflows

without reorganizing the existing structure.

---

## Deliverables

Generate:

1. The complete folder structure.
2. Every initial Markdown document.
3. Recommended Claude agents/skills organization.
4. A brief explanation of the purpose of each directory.
5. Suggestions for improving this software house structure using industry best practices.

Do **not** start implementing the application itself.

Your goal is to build a reusable, scalable software development framework that will serve as the foundation for all future development.

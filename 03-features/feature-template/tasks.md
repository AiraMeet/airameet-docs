# Work Breakdown Structure (WBS) & Tasks: [Feature Name]

## 1. Task Standards

Every task must specify all required fields:
- **Task ID**: Prefix (`AI-[FEAT]`, `BE-[FEAT]`, `FE-[FEAT]`, `QA-[FEAT]`, `DOC-[FEAT]`)
- **Task Name**: Clear, action-oriented title
- **Description**: Technical explanation of work required
- **Priority**: Critical / High / Medium / Low
- **Responsible Department**: AI, Backend, Frontend, QA, or DevOps
- **Dependencies**: Prerequisite Task IDs
- **Implementation Steps**: Bulleted execution items
- **Time Estimates**: Optimistic ($O$), Most Likely ($M$), Pessimistic ($P$), Expected $E = \frac{O + 4M + P}{6}$
- **Acceptance Criteria**: Verifiable conditions
- **Deliverable**: Code, artifact, or test suite
- **Status**: `To Do`, `In Progress`, `In Review`, `Done`, `Blocked`

---

## 2. Tasks Catalog Example

### `BE-FEAT-001`: Implement Domain Service
- **Task ID**: `BE-FEAT-001`
- **Task Name**: Implement Domain Service for [Feature]
- **Description**: Create domain entities, business validation rules, and repository interfaces.
- **Priority**: High
- **Responsible Department**: Backend Engineering
- **Dependencies**: None
- **Implementation Steps**:
  1. Define domain entities in `AiraMeet.Domain`.
  2. Implement repository interface in `AiraMeet.Application`.
  3. Register dependency injection services.
- **Time Estimates**:
  - Optimistic ($O$): 8h
  - Most Likely ($M$): 12h
  - Pessimistic ($P$): 20h
  - Expected ($E$): $\frac{8 + 4(12) + 20}{6} = 12.67\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Unit tests achieve $\ge 90\%$ line coverage on domain rules.
- **Deliverable**: C# code commit with passing tests.
- **Status**: `To Do`

# Feature Specification Template: [Feature Name]

## 1. Purpose & Guidance

This directory provides the standardized blueprint for documenting any new major feature within the AiraMeet platform (e.g., *Decision Extraction*, *Action Item Detection*, *Meeting Summarization*, *Knowledge Graph Construction*).

Whenever a new feature is planned:
1. Create a new directory under `docs/03-features/` with sequential numbering: `docs/03-features/XX-[feature-name]/`.
2. Copy all template files from this directory into the new feature folder.
3. Complete every section thoroughly, following the task management standards and PERT estimation rules established in the project planning guide.

---

## 2. Directory Structure

```
docs/03-features/[feature-name]/
├── README.md               # Feature entry point and summary
├── requirements.md         # Functional, non-functional, and edge cases
├── technical-design.md     # Architecture, data flows, and schemas
├── tasks.md                # Work Breakdown Structure (WBS) with PERT estimates
├── testing.md              # Feature-specific test cases and acceptance criteria
└── timeline.md             # Schedule, critical path, and milestone mappings
```

---

## 3. Mandatory Standards Checklist

- [ ] All requirements follow the **EARS** (Easy Approach to Requirements Syntax) or **MoSCoW** framework.
- [ ] Technical design contains clear Mermaid sequence and component diagrams.
- [ ] Every engineering task uses the standardized task attributes:
  - Task ID (Prefix: `AI-`, `BE-`, `FE-`, `QA-`, `DOC-`)
  - Description & Priority
  - Responsible Department & Dependencies
  - Implementation Steps
  - 3-point PERT estimate: $E = \frac{O + 4M + P}{6}$
  - Measurable Acceptance Criteria & Concrete Deliverable
  - Status (`To Do`, `In Progress`, `In Review`, `Done`, `Blocked`)
- [ ] No sequential addition of parallel cross-departmental workstreams.

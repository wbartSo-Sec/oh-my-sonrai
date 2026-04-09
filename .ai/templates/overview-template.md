# Epic Overview: [PLACEHOLDER - Epic Title]

**SAR-ID**: `[PLACEHOLDER - SAR-ID]`  
**Jira Epic**: [PLACEHOLDER - Epic Link]

> **⚠️ IMPORTANT**: This file is a coordination dashboard only. Atlas executes from individual `.sisyphus/plans/[SAR-ID].md` files in each repository. This overview does NOT contain executable tasks.

---

## 1. Epic Summary

**Epic Title**: [PLACEHOLDER - Epic Title]

**Description**: [PLACEHOLDER - Provide a clear narrative of what this epic delivers: business value, user impact, technical scope]

**Objective**: [PLACEHOLDER - What success looks like across all repositories]

---

## 2. Affected Repositories

| Repository | Story SAR-ID | Current Status | Notes |
| :--- | :--- | :--- | :--- |
| [PLACEHOLDER - Repo 1] | [SAR-???] | Not Started | [Optional notes] |
| [PLACEHOLDER - Repo 2] | [SAR-???] | Not Started | [Optional notes] |
| [PLACEHOLDER - Repo 3] | [SAR-???] | Not Started | [Optional notes] |

---

## 3. Execution Order

**Recommended Sequence**:

1. **[PLACEHOLDER - First Story/Repo]**
   - Why: [PLACEHOLDER - Reason for starting here]
   - Dependencies: None (or specify)

2. **[PLACEHOLDER - Second Story/Repo]**
   - Why: [PLACEHOLDER - Reason]
   - Dependencies: Requires Story #1 to be complete

3. **[PLACEHOLDER - Remaining Stories]**
   - Continue in order...

**Cross-Repo Dependencies**:
- [PLACEHOLDER - Describe any API contracts, database schema changes, or interface agreements that must be synchronized between repos]
- [PLACEHOLDER - List any blocking dependencies]

---

## 4. Feature Flag Coordination

**Flags Involved**:
| Flag Name | Repository | Default State | Rollout Order | Notes |
| :--- | :--- | :--- | :--- | :--- |
| [PLACEHOLDER - Flag 1] | [Repo] | disabled | 1 | [Notes] |
| [PLACEHOLDER - Flag 2] | [Repo] | disabled | 2 | [Notes] |

**Coordinated Rollout Strategy**:
- [PLACEHOLDER - Describe the order flags must be enabled across repos]
- [PLACEHOLDER - Describe any rollback scenarios or flag dependencies]
- [PLACEHOLDER - Specify how to coordinate enable/disable timing if multiple teams are involved]

---

## 5. Jira Story Links

| SAR-ID | Repository | Plan Path | Jira Story Link |
| :--- | :--- | :--- | :--- |
| [PLACEHOLDER - SAR-???] | [PLACEHOLDER - Repo 1] | `.sisyphus/plans/[SAR-???].md` | [Link] |
| [PLACEHOLDER - SAR-???] | [PLACEHOLDER - Repo 2] | `.sisyphus/plans/[SAR-???].md` | [Link] |
| [PLACEHOLDER - SAR-???] | [PLACEHOLDER - Repo 3] | `.sisyphus/plans/[SAR-???].md` | [Link] |

> **Note**: Each Story has its own `.sisyphus/plans/[SAR-ID].md` file where Atlas executes the detailed plan. See those files for task breakdown and execution steps.

---

## 6. Status Tracker

**Story Progress**:

- [ ] [PLACEHOLDER - SAR-???] — [PLACEHOLDER - Story Title] (Jira: Discovery / Ready for Dev / In Progress / In Review / Done)
- [ ] [PLACEHOLDER - SAR-???] — [PLACEHOLDER - Story Title] (Jira: Discovery / Ready for Dev / In Progress / In Review / Done)
- [ ] [PLACEHOLDER - SAR-???] — [PLACEHOLDER - Story Title] (Jira: Discovery / Ready for Dev / In Progress / In Review / Done)

**Overall Epic Status**: [PLACEHOLDER - Not Started / In Progress / In Review / Staged / Deployed]

---

## 7. Notes

**Dev Lead Decisions & Architectural Agreements**:

- [PLACEHOLDER - Key architectural decision (e.g., "All API changes use versioned endpoints")]
- [PLACEHOLDER - Cross-repo interface contract (e.g., "Event schema must include timestamp and source")]
- [PLACEHOLDER - Shared library or utility usage]

**Known Blockers**:

- [PLACEHOLDER - Any external dependencies, approval processes, or environment setup needed]
- [PLACEHOLDER - Infrastructure or service requirements]

**Dev Lead Contact**: [PLACEHOLDER - Name/Slack handle for coordination questions]

---

## How to Use This Overview

1. **During Planning**: Use this template to map all Stories across repositories
2. **During Execution**: Update the Status Tracker as Jira Stories move through workflow
3. **For Coordination**: Reference the Execution Order and Cross-Repo Dependencies to ensure sequential work
4. **For Rollout**: Use Feature Flag Coordination to sequence environment-wide rollout
5. **Finding Plan Details**: Click any plan path (`.sisyphus/plans/[SAR-ID].md`) to see Atlas execution steps for that Story

---

**Last Updated**: [PLACEHOLDER - Date]  
**Last Updated By**: [PLACEHOLDER - Dev Lead Name]

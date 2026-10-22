# Support system — tickets, teams, levels

Code-verified 2026-10-19. How a support request reaches the right person and moves between people. Calls on tickets are in [CALLING_SYSTEM.md](CALLING_SYSTEM.md); recordings in [CALL_RECORDING_DESIGN.md](CALL_RECORDING_DESIGN.md).

## The two questions: which team, how senior

Every ticket has a **department** (what kind of problem) and a **level** (how senior). They're independent:

| | Decided by | Configured in the portal |
|---|---|---|
| Department | The topic the person taps when raising it | Support Departments |
| Level | Starts at 1 (front line); moves by escalation | Support Levels |

An agent's **scope** (`SupportScope`, `app/domain/support_departments.py`) says which departments they cover and their level in each. It's built from their roles by `admin_scope()`, resolved in one place (`SupportScopeService`), and every support check uses it:

- **See** a ticket (read it, add private notes, hear its call recordings): its department is covered.
- **Handle** it (reply, close, take, escalate, send back, move, call): also their level *in that department* is at or above the ticket's.

Levels are per department: senior in Payments and front line in Jobs doesn't make someone senior in Jobs.

## Roles

A role has a support level (or none: front line if it has ticket permissions) and departments (none = every department). Super admins (`roles.manage`) see and handle everything. A person with several roles gets, per department, the highest level of the roles covering it.

## Topics → departments

The apps send a fixed topic code (`payment`, `job`, `behaviour`, `account`, `other` — `TOPICS`); older builds send only the label, mapped by `topic_from_subject()`. Each department takes some topics; one department is the **fallback** (topics nobody takes, "Something else"). Launch set: Payments & Payouts, Jobs & Bookings, Trust & Safety, Account & Tech, General.

**No empty queues:** each department's **coverage** is the highest level any active admin works at there (`coverage()` over `staffing()`; super admins don't count for routing — a department only they watch has no front line — but they do handle every escalated level). A ticket for a department nobody works in goes to the fallback; the Departments page flags such departments and shows each one's coverage ("Up to Senior support").

## Moving a ticket

| Move | Who | What happens |
|---|---|---|
| Escalate | Agent who can handle it | One level up, same department, with a reason; unassigned; the person is told a senior is looking (first time only). Above the department's coverage it still goes (super admins handle every level); the agent is told it's with the super admins |
| Send back | Agent at its level | One level down, to whoever escalated it — only if that was since its last department move |
| Move to another department | Agent who can handle it | Same level, new department, unassigned, private note; the person sees "We've passed this to our X team" |
| Safety words | Automatic | To the safety level, and to the safety department if anyone works there (else it stays with a staffed team) |
| No reply in N hours | Automatic, hourly | One level up (per level's setting) — also where only super admins work at that level; a move resets the clock |

Every move is one row in `ticket_escalations` (department moves have `from_department`/`to_department`), shown as the ticket's history.

## Changing departments safely

- A department with **open** tickets can't be removed (move them first).
- A department that is some role's **only** department can't be removed (the role would silently become "all departments").
- Removing one moves its closed tickets to the fallback first, then deletes it (tickets reference departments).

## Code map

| Layer | Files |
|---|---|
| Rules (pure) | `app/domain/support_departments.py`, `app/domain/ticket_escalation.py` |
| SQL | `support_department_repo.py`, `support_level_repo.py`, `support_ticket_repo.py` |
| Decisions | `support_ticket_service.py`, `support_department_service.py`, `support_scope_service.py`, `support_level_service.py` |
| API | `/support-tickets/admin/*` (incl. `/transfer`), `/support-departments`, `/support-levels` |
| Portal | `pages/SupportDepartments.tsx`, `pages/SupportLevels.tsx`, `pages/SupportTickets.tsx`, `pages/SupportTicketDetail.tsx`, `components/ticket/*` |
| Apps | `app/support-chat.tsx` (`CATEGORY_TOPICS`) in both apps |
| Migrations | 057–058 (levels), 059 (calls), 061 (departments) |

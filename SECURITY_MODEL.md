# Threat model — Weapons of Mass Application (drtclem/weapons-of-mass-application)

Status: threat model built from `docs/PRD.md` v1.1 and the user's answers (2026-11-11). The repository currently contains only `README.md` and `docs/PRD.md`; no implementation exists yet, so everything below describes the **planned** system. **User decision:** scan the repository as-is, as a design review of the PRD. **User:** there is no infrastructure repository.

## 1. System overview and trust boundaries

A multi-tenant SaaS agent that runs a user's job search end to end (PRD §1, §9). Everything runs on **one cloud droplet**: web app, API, Postgres, a task queue, Playwright workers, a scheduler, and outbound LLM calls.

Trust boundaries:
- **Browser ↔ Web/API** — authenticated by Google OAuth sign-in (A5, §7.1).
- **Tenant ↔ tenant** — all tables carry `tenant_id` (§8); isolation is enforced in application code, not by separate infrastructure.
- **Agent ↔ untrusted content** — job postings (Ashby/Greenhouse/Lever, HN, LinkedIn public pages), recruiter emails read from Gmail, ATS pages and CAPTCHA images all flow into LLM prompts (§7.2–7.7, §9). The agent acts on them with Gmail send, Calendar write and browser automation, without per-item approval (§2 goal 2, §4 Autonomy).
- **Platform ↔ third parties** — Google APIs, ATS sites, levels.fyi, LLM provider.

## 2. Components

| Component | Covers (planned) |
|---|---|
| Web App | Kanban board, profile review, survey, intake forms (§7.9, §6) |
| API & Auth | Google OAuth sign-in, tenant/user creation, session handling, tenant scoping of every query (§7.1, §8) |
| Agent Workers | Scheduler, discovery, decision, tailoring, Playwright apply workers, CAPTCHA chain (§7.2–7.6, §9) |
| Email & Calendar | Gmail read/send, thread linking, follow-ups, scheduling replies, calendar events, digest/notification emails (§7.7, §7.8) |
| Secrets Vault | Encrypted Google tokens and ATS passwords, per-tenant keys, envelope encryption (A6, §8 `ats_credentials`, §9 Secrets) |
| Data Store | Postgres schema and tenant scoping (§8) |
| Demo Mode | Simulated recruiter mailbox, seeded demo tenant (A1, §9, §12) |

## 3. Entry points and who can reach them

- Web app and API — any internet user (unauthenticated) up to sign-in; authenticated tenants afterwards.
- Google OAuth callback — internet.
- One-click links in emails: interview-slot confirmation (§7.7), CAPTCHA reply form (A3, §7.6) — anyone holding the email/link.
- File uploads: resume PDF/DOCX (§6.2) — authenticated users.
- Fetched external content: LinkedIn public profile URL, GitHub username, Google Drive folders, job boards, HN thread, levels.fyi — attacker-influenced content (and user-supplied URLs → possible SSRF from the droplet).
- Inbound email in the user's Gmail — anyone who can email the user.

## 4. Assets and sensitivity

**User:** all of the assets below matter equally; none is lower priority.

- **Google OAuth tokens** with Gmail read/send and Calendar read/write — highest; compromise = full mailbox access and ability to send as the user.
- **ATS credential vault** (`ats_credentials.password_ciphertext`) and per-tenant keys.
- **Profile, resumes, Drive documents** — PII (contact, work history, work authorization/visa).
- **Email threads, interview data, feedback, offer data.**
- **Tenant isolation** — PRD acceptance criterion: two tenants cannot see each other's jobs, applications or credentials (§13).
- **Guardrail integrity** — no interview booked without user confirmation, no compensation email, no application before profile approval, no invented experience (§10).

## 5. Attacker profiles

**User:** all four attacker profiles below matter.

1. Another authenticated tenant trying to read or act on another tenant's data (IDOR, missing `tenant_id` scoping).
2. An attacker who controls content the agent reads — a job posting, a recruiter email, an ATS page — using prompt injection to make the agent send email, exfiltrate profile data or tokens, book calendar events or bypass guardrails.
3. An unauthenticated internet attacker against the web app/API, OAuth flow, and one-click email links.
4. A compromised or malicious third party (ATS, scraped site) serving malicious files or redirects to the Playwright workers.

## 6. Existing (planned) controls

- Profile approval gate before autopilot (§7.1, §10).
- Interview booking only after explicit user confirmation (§7.7, §10).
- No outgoing email involving compensation (§7.7, §10, §13).
- Tokens and ATS passwords encrypted at rest with per-tenant keys (A6, §9).
- Activity log of every agent action (§7.12, §10).
- Simulated recruiter toggled by tenant flag, never active for real tenants (§9).

## 7. Out of scope / accepted risk

**User:** nothing is out of scope and no risk is accepted — everything matters (including demo mode, scraping/CAPTCHA concerns, and single-droplet hosting). PRD non-goals (§2, §11) are features, not security exclusions.

## 8. Finding tags

- `Component` — one of the component names in §2.
- `OWASP` — OWASP Top 10 (2021) category, e.g. `A01:2021 Broken Access Control`, when the finding maps to one.
- `OWASP LLM` — OWASP Top 10 for LLM Applications (2025) category, e.g. `LLM01:2025 Prompt Injection`, when the finding involves an LLM.

## 9. Open questions

- None outstanding. Revisit this model once implementation code lands.

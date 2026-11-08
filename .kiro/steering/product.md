# EDAAS: Emergency Drug Allergy Alert System

Educational decision-support prototype for a software engineering course (Group 9).
All data is fictional. Not for clinical use. The healthcare professional always makes
the final decision; the system never replaces clinical judgment.

## Roles
doctor, nurse, pharmacist, patient, admin.

## Process
V-Model: every requirement must trace to a design element, an implementation task,
and a test. Keep this traceability visible in the specs.

## Stack
React + TypeScript + Tailwind + shadcn/ui, Supabase (Auth, Postgres, RLS, RPC).
Roles live in a separate user_roles table checked by a SECURITY DEFINER has_role()
function. Never trust the client for role checks. Never commit secrets.

## Safety rules (non-negotiable)
- A known conflict must never pass silently.
- If the allergy check errors, times out (>5s) or the network fails, BLOCK the drug.
- Alerts must be unmistakable (not a toast) and never shown with "Safe to proceed".
- Only doctors can override, with a typed reason (min 10 chars), saved with name and time.
- Dispensing stays disabled until the conflict is resolved or overridden.
- Self-reported allergies are "unverified" until a doctor confirms them.
- Allergy check target: about 2 seconds.

## Repo rules
Never include student IDs or personal names in committed files; credit "Group 9".
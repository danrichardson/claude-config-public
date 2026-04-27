---
description: '"short description" — create a new ticket'
argument-hint: 'short description'
---

# Create a New Ticket

You are creating a new ticket in the current project. The user will describe a bug, feature, or enhancement.

## Pre-flight

If `tickets/` does not exist or `.claude/ticket-config.md` does not exist, the project hasn't been bootstrapped yet. Tell the user to run `/ticket-install` first and stop.

## Steps

1. **Read the project's ticket config** at `.claude/ticket-config.md` to learn the tickets directory and ID prefix (default: `tickets/` and `TKT-`).

2. **Determine the next ticket ID**:
   - List files **recursively** under the tickets directory matching `{PREFIX}*.md` — this MUST include terminal subfolders (`shipped/`, `deferred/`, `wontfix/`) so IDs are never reused for tickets that have been moved out of the active set.
   - Find the highest number across all matches and increment by 1.
   - If no tickets exist, start at `{PREFIX}001`.
   - The new ticket file is always created at the root of the tickets directory (`{tickets-dir}/{PREFIX}{NNN}.md`), never in a subfolder.

3. **Gather info from the user's input** (the argument to this command):
   - Title (short, descriptive)
   - Type: bug, feature, or enhancement
   - Priority: low, medium, high, or critical (infer from context, default medium)
   - Description
   - Acceptance criteria (infer from the description if not explicit)
   - **Visual context** (only if the user pasted one or more images into the prompt): for each image, write 1–3 concrete, factual sentences capturing what it shows — UI element + state + visible text + any error or anomaly. The binary image is NOT saved with the ticket; this prose is the only artifact downstream agents (e.g. `/ticket-investigate`) will see, so it must stand alone (no "see screenshot above"). If the image contradicts or sharpens the user's prose, prefer the image and say so explicitly.
   - **App / preview profile**: read the `## Preview profiles` section of `.claude/ticket-config.md`. If there are 2+ profiles, use AskUserQuestion to ask which one this ticket should target (show the profile names + a one-line description for each; default selection = the profile marked `default: true`). If there's only 1 profile, use it silently. If there are no profiles, set `app: (none)`.

4. **Create the ticket file** at `{tickets-dir}/{PREFIX}{NNN}.md` using `{tickets-dir}/TEMPLATE.md`:
   - Fill in all header fields (id, title, type, status: open, priority, created date, updated date)
   - Fill in Description and Acceptance Criteria. If images were attached, append a `**Visual context**` subsection at the end of the Description with one bullet per image.
   - Leave all agent sections empty

5. **Output a summary**:
   ```
   Created {ID}: {title}
   Type: {type} | Priority: {priority} | Status: open

   Next: run /ticket-investigate {ID} to begin investigation
   ```

## Rules
- Status must be `open` for new tickets
- Branch field is empty until investigation/approval
- Do NOT start investigating yet — just create the ticket
- Use today's date for created/updated
- **Pasted images are session-only.** The ticket file stores text only — any image the user pasted into the prompt is dropped at session end. Always distill it into the Visual context bullets (step 3 / step 4) so the next agent has the information in text form.

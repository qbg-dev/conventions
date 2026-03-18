# UI Journey Verification Convention

Document all critical user paths in a `UI_JOURNEY.md` file. A recurring verification agent walks through each journey with Playwright, screenshotting at each expected outcome step. Ad-hoc UI testing (clicking around randomly) misses edge cases and doesn't scale. Documented journeys ensure consistent coverage and make regressions detectable.

## UI_JOURNEY.md Structure (1–8)

1. **One file per project.** Place `UI_JOURNEY.md` at the repo root (or in `claude_files/` if the repo separates agent artifacts). Every project with a UI gets one.

2. **Journey format.** Each journey is a numbered section with alternating action and expectation steps:

```
## Journey N: [Name]
1. [Step description]
2. **Expect**: [What should be visible/true]
3. [Next step]
4. **Expect**: [Expected outcome]
```

3. **Steps are concrete and automatable.** "Click the login button" not "Navigate to authentication." "Type 'hello' in the chat input" not "Interact with the messaging feature." An agent must be able to translate each step into a Playwright action without guessing.

4. **Expectations are observable.** "The sidebar shows 3 conversation items" not "The sidebar works." "The URL changes to `/dashboard`" not "The user is redirected." Every `**Expect**` must be verifiable by inspecting the DOM, taking a screenshot, or checking the URL.

5. **Cover the critical paths first.** Login, primary workflow, error states, empty states. Don't try to document every possible click—start with the 5–10 journeys that would embarrass you if broken in a demo.

6. **Include setup preconditions.** If a journey requires a logged-in user, seeded data, or a specific screen size, state it at the top of the journey. Don't assume the agent knows the context.

7. **Version the journeys.** When the UI changes, update the journey steps. Stale journeys are worse than no journeys—they generate false failures that erode trust in the system.

8. **Tag journeys by priority.** Use `[P0]`, `[P1]`, `[P2]` tags after the journey name. P0 journeys run every cycle. P1 journeys rotate. P2 journeys run during full sweeps only.

## Verification Protocol (9–18)

9. **Separate agent, separate pass.** The verification agent reads `UI_JOURNEY.md` and walks through journeys. It does NOT fix issues during verification. Verification and fixing in the same pass leads to partial walks—the agent gets distracted fixing the first bug and never checks the remaining journeys.

10. **Pick 2–3 journeys per cycle.** A single verification pass covers 2–3 journeys. Rotate selection to cover all journeys over 3–4 rounds. Track which journeys were last verified (timestamps in a state file or commit log).

11. **Use Playwright for all interactions.** Navigate, click, type, wait, screenshot. No manual browser interaction. The verification must be reproducible by any agent on any machine.

12. **Screenshot at every Expect step.** Take a screenshot immediately after the action that should produce the expected state. Name screenshots `journey-N-step-M.png` for traceability.

13. **Compare against expectations, not pixel-perfect baselines.** Check for element presence, text content, URL state. Visual regression testing (pixel diffing) is a separate concern—journey verification checks functional correctness.

14. **Report all issues before stopping.** Walk through all selected journeys to completion. Collect every failure. Don't stop at the first broken step—downstream steps might reveal additional issues.

15. **Structured output.** Produce a verification report:

```
## Verification Report — [date]
Journeys checked: 2, 5, 7

### Journey 2: Login Flow — PASS
- Step 1: ✓ Login page loaded
- Step 2: ✓ Password field accepted input
- Step 3: ✓ Redirected to /dashboard

### Journey 5: Chat Send — FAIL
- Step 1: ✓ Chat input visible
- Step 2: ✗ Expected: "Message sent" toast. Got: no toast, message appears in list but no confirmation.
- Screenshot: journey-5-step-2.png
```

16. **Don't fix during verification.** File the issues (in a report, commit message, or issue tracker). Fixes happen in a separate step after all journeys are checked. This prevents the common failure mode where the agent "fixes" one thing, breaks another, and never finishes checking.

17. **Timeout per journey.** Cap each journey at 2 minutes. If a journey takes longer, it's either too complex (split it) or the app is too slow (file a performance issue).

18. **Run against a real server.** Verification targets a running instance, not a mock. Start the dev server before running journeys. Document the server URL and startup command in `UI_JOURNEY.md`.

## Rotation Strategy (19–24)

19. **Track coverage.** Maintain a simple rotation log—which journeys were verified in which round. A JSON file, a markdown table, or even commit messages work.

20. **Rotate, don't repeat.** If journeys 1–3 were checked last round, check 4–6 this round. Every journey gets verified at least once per full rotation (typically 3–4 rounds).

21. **P0 journeys are exempt from rotation.** They run every cycle. Login, primary happy path, and any journey that broke in the last 2 rounds are always P0.

22. **Promote on failure.** When a journey fails, promote it to P0 for the next 2 rounds. After 2 consecutive passes, demote it back.

23. **Don't skip journeys because they "always pass."** That's exactly how regressions sneak in. The rotation ensures every journey gets checked regardless of its history.

24. **Full sweep on release.** Before any deployment to production, run all journeys (not just the rotation subset). This is the gate check.

## Integration with Hardening Rounds (25–32)

25. **Phase 1 of hardening references UI_JOURNEY.md.** The hardening protocol's first phase is "verify current state." This means running the journey verification, not ad-hoc clicking.

26. **Lightweight smoke tests on cron.** A separate cron job (every ~9 minutes) runs a stripped-down version: just the P0 journeys, fail-fast, no screenshots. This catches catastrophic regressions between hardening rounds.

27. **Full journey verification during hardening.** Hardening rounds run the complete rotation subset with full screenshots and structured reports. This is more thorough than the cron smoke test.

28. **Hardening fixes reference journey IDs.** When a hardening round fixes a bug found by journey verification, the commit message references the journey: "fix: chat input not clearing after send (Journey 5, Step 4)."

29. **New features get journeys before merge.** A PR that adds a UI feature must include corresponding journey steps. No journey = no merge. The journey definition is part of the feature spec.

30. **Journey verification gates deployment.** The full sweep (rule 24) is a required step in the deployment checklist. Failed journeys block deployment.

31. **Separate verification from hardening fixes.** The hardening round has two distinct phases: (a) run verification, collect all failures; (b) fix failures. Never interleave—this is the single most common anti-pattern.

32. **Track journey health over time.** Keep a running tally of pass/fail per journey across rounds. Journeys that fail frequently indicate flaky UI or under-tested code paths—both worth investigating beyond just fixing the immediate failure.

## Anti-Patterns (33–40)

33. **Combining verification and fixing.** The agent starts walking through journeys, hits a failure, immediately starts fixing it, breaks something else, and never finishes checking. Always complete the full verification pass first.

34. **Verifying the same 2 journeys every cycle.** This gives false confidence. The 8 unchecked journeys could all be broken. Rotate.

35. **Skipping journeys because they "always pass."** Regressions hide behind assumptions. The rotation exists precisely for this.

36. **Vague expectations.** "The page looks right" is not an expectation. "The page contains a table with at least 3 rows" is.

37. **Too many journeys.** Start with 5–10. If you have 30 journeys, most are either too granular (merge them) or testing internal implementation rather than user-visible behavior.

38. **Stale journeys after UI changes.** The login button moved from the header to a modal, but the journey still says "click the header login button." Update journeys when the UI changes.

39. **Screenshots without context.** A screenshot named `screenshot-1.png` is useless. Name it `journey-3-step-2-dashboard-loaded.png` and reference it in the report.

40. **Testing on a mock instead of a real server.** Journey verification must hit a real running instance. Mocks hide integration bugs—the exact bugs that journey verification is designed to catch.

## Writing Good Journeys (41–48)

41. **Start from the user's perspective.** "As a cost engineer, I open the app and search for steel rebar prices" not "The SearchComponent renders with the correct props."

42. **One journey, one goal.** "Login and view dashboard" is one journey. "Login, view dashboard, create a report, export it, and share it" is three journeys crammed together.

43. **Include the unhappy path.** "Type an invalid password and expect an error message" is a journey. Error states are user-visible behavior.

44. **Test empty states.** "Open the app with no data and expect a helpful empty state message" catches the common bug where the app crashes or shows a blank screen on first use.

45. **Test responsive behavior.** If the app targets mobile, include a journey at 375px width. Document the viewport size in the preconditions.

46. **Test with realistic data.** "Search for '钢筋'" not "Search for 'test123'." Realistic queries find real bugs (encoding issues, long text overflow, special characters).

47. **Keep steps atomic.** Each step is one user action. "Click the search button" is one step. "Click search, wait for results, scroll down, and click the first result" is four steps.

48. **Document what you don't test.** Add a "Not Covered" section listing known gaps. This prevents false confidence and helps prioritize future journey additions.

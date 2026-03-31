---
description: "Scan your Flutter project and get a readiness report — checks flavors, error monitoring, signing, CI/CD, security, and store compliance"
argument-hint: "[optional: path to Flutter project]"
---

# /audit — Pre-Launch Readiness Audit

You are running a readiness audit on a Flutter project.

## Process

1. **Locate the project.** If `$ARGUMENTS` contains a path, use it. Otherwise, look for `pubspec.yaml` in the current directory. If not found, ask the user for the path.

2. **Dispatch the `checklist-auditor` agent** to scan the project. The agent will analyze the codebase and produce a structured PASS/WARN/FAIL report covering all pre-launch checklist items.

3. **Present the report** to the user with:
   - Summary counts (PASS / WARN / FAIL / INFO / SKIP)
   - Blockers highlighted at the top
   - For each FAIL or WARN, a concrete recommendation with the relevant `flutter-go-to-market` skill to invoke

4. **Offer next steps** based on results:
   - If blockers exist: "Want me to help fix [blocker]? I'll use `flutter-go-to-market:[skill]`."
   - If clean: "Your project looks ready. Want to start `/flutter-go-to-market:ship`?"

## Usage

```bash
/flutter-go-to-market:audit                    # Audit current directory
/flutter-go-to-market:audit apps/mobile        # Audit specific path
```

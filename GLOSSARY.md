# Merge Gate

A fast, language-agnostic check that decides whether a change is worthy of joining `main`, without end-to-end tests on the critical path.

## Language

**Gate**:
The check a change must pass before it may join `main`.
_Avoid_: Pipeline, CI check, review

**Verdict**:
The gate's single answer for one change: _merge_, _merge with notes_, or _block_.
_Avoid_: Result, status, score

**Finding**:
One concrete problem the gate reports, with a location, a severity, and evidence.
_Avoid_: Issue, warning, comment

**Change intent**:
What one specific change set out to do, as stated in its PR description or linked issue.
_Avoid_: Goal, purpose, PR summary

**Project intent**:
What the codebase as a whole is for and the rules it lives by, as stated in its CLAUDE.md, ADRs, glossary, and README.
_Avoid_: Conventions, standards, vision

**Baseline**:
An existing safeguard the gate is measured against on the same changes: no-mistakes, CI, or an E2E suite.
_Avoid_: Competitor, reference tool

**Catch rate**:
The share of buggy changes where the verdict is _block_ and a finding points at the real bug.
_Avoid_: Recall, detection rate, accuracy

**False-block rate**:
The share of clean changes where the verdict is _block_.
_Avoid_: False-positive rate, noise

# Behavioral — Senior Interview Guide

This chapter covers behavioral questions used to assess how a senior engineer actually operates under real constraints—ambiguity, disagreement, failure, delegation, and unfamiliar tooling—rather than what they know. Unlike the technical chapters, there is no single correct answer: use the framework and pitfalls below to build your own story from a real project, not to recite someone else's.

> **How to answer:** use STAR (Situation, Task, Action, Result), but weight it toward *Action*—the specific judgment call you made and why—and *Result*—the measurable or observed outcome, including what you would do differently. A story that is 80% situation and 20% action reads as passive, not senior.

This chapter is a partial scaffold: it currently covers the prompts below. See `ROADMAP.md` for the remaining planned topics (leadership without authority, technical disagreement, production ownership, failed decisions, mentoring).

## Contents

1. [Testing and quality](#1-testing-and-quality)
2. [Working with AI-assisted development](#2-working-with-ai-assisted-development)
3. [Rapid revision](#3-rapid-revision)

---

## 1. Testing and quality

### 1. Tell me about a time testing a system was especially difficult. What made it hard, and what did you do?

A strong answer names a genuinely difficult-to-test property—concurrency or timing, an external dependency with no good test double, legacy code with no seams for substitution, non-deterministic data, or a failure mode that only reproduces in production—not "we didn't have enough time."

Structure the story around:

- **Situation/Task:** the system and the specific constraint that made it hard (an untestable third-party integration, flaky async behavior, tightly coupled legacy code, a race condition that only showed up under production load).
- **Action:** what you actually changed—introduced a seam or interface to allow substitution, used contract tests instead of full integration tests, added a fake or test double, used a tool like Testcontainers for a real dependency instead of mocking it, restructured for dependency injection, or deliberately accepted a coverage gap and named the compensating control (monitoring, canary, manual QA) instead of chasing full automation.
- **Result:** the confidence or defect outcome, and if the approach didn't fully work, what you learned and changed next time.

The common trap is reciting a testing-pyramid lecture instead of telling one concrete story. Interviewers are listening for judgment under a real constraint—including knowing when *not* to chase full automated coverage for a low-value, hard-to-test path.

Expect follow-ups: "What would you do differently now?", "How did you get the team to invest in it?", "What's your default policy for testing an external integration you don't control?"

## 2. Working with AI-assisted development

### 2. What's a problem you ran into recently that an AI coding assistant could not resolve for you?

A strong answer demonstrates calibrated trust in AI tooling, not a verdict on it—senior interviewers use this to probe whether you evaluate AI output critically and still own the final decision, rather than accepting whatever compiles or passes the existing tests.

Good categories to draw a real example from:

- A defect requiring causal reasoning across many files or services that the assistant did not have full context on—state threaded through several components, or a race condition that only reproduces under specific load.
- A domain-specific business rule or historical decision that isn't documented anywhere the assistant could see.
- A fix that was syntactically plausible and passed the visible tests but was operationally wrong—for example, it quietly changed transaction or locking semantics while "fixing" the symptom.
- An architecture or trade-off decision requiring organizational context (team skill level, on-call load, existing technical debt) that a coding assistant has no visibility into.

Structure the story around what the assistant suggested, why it looked right but wasn't (it optimized for the visible symptom rather than the underlying invariant), how you found the actual root cause yourself (profiling, reading the spec, talking to the domain owner), and what that changed about how you use AI tools afterward—for example, scoping AI to well-bounded, verifiable tasks and keeping root-cause diagnosis of cross-cutting or concurrency bugs on yourself.

The trap is picking an example that makes you sound like you trusted the tool blindly, or dismissing AI tooling wholesale—the interviewer wants evidence of calibrated trust, not a verdict on the technology.

---

## 3. Rapid revision

### Must-answer questions

Before an interview, answer these without notes:

1. What made a testing effort especially hard, and what did you actually change?
2. What's a case where an AI coding assistant's suggestion looked right but wasn't, and how did you catch it?

### Thirty-second summary

Behavioral answers are judged on the specificity of the *action* and the honesty of the *result*, not on matching a template. Pick real stories where the hard part was a genuine constraint—an untestable dependency, a misleading AI suggestion—and be ready to explain the judgment call, not just the outcome.

## Official references

- [STAR method (Situation, Task, Action, Result)](https://en.wikipedia.org/wiki/Situation,_task,_action,_result)

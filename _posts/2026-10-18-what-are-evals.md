---
title: "What are evals? Testing AI systems when there's no right answer"
date: 2026-10-18 00:00:00 -0600
categories: [AI, Fundamentals]
tags: [evals, llm, testing]
---

I keep seeing the word **evals** in AI job postings and engineering blogs, so my first post is about what they are and why they matter.

## The problem

In normal software, tests are simple: same input, same output. LLMs don't work that way. The same prompt can produce different wording each time, and "correct" is often a judgment call. That makes it hard to know whether a change to a prompt or model actually made things better.

## What an eval is

An **eval** is a repeatable test that measures how well an AI system performs a specific task. It has four parts:

1. **Dataset:** example inputs, ideally with expected answers or clear criteria
2. **Task:** the thing under test (a prompt, model, workflow, or agent)
3. **Grader:** how each output gets scored
4. **Metric:** the number you track over time, like "pass rate"

## Types of graders

| Grader | Good for | Tradeoff |
|---|---|---|
| Code-based (exact match, regex, JSON validity) | Classification, extraction, structured output | Fast and cheap, but only works when answers are checkable |
| Human review | Quality, tone, nuance | Most trustworthy, but slow and expensive |
| LLM-as-judge | Open-ended text at scale | Scalable, but the judge has its own biases and needs checking |

## A tiny example

Say I'm building a workflow that classifies support emails as `billing`, `bug`, or `other`.

```python
test_cases = [
    {"email": "I was charged twice this month", "expected": "billing"},
    {"email": "The app crashes when I open settings", "expected": "bug"},
    {"email": "Do you have an office in Toronto?", "expected": "other"},
]

def run_eval(classify):
    passed = 0
    for case in test_cases:
        result = classify(case["email"])
        if result == case["expected"]:
            passed += 1
        else:
            print(f"FAIL: {case['email']} -> {result}")
    print(f"Score: {passed}/{len(test_cases)}")
```

Now whenever I change the prompt or switch models, I run `run_eval` and see whether the score went up or down.

## Why this matters

- **Catch regressions:** prompt changes can silently break old cases
- **Compare options:** model A vs. model B, prompt v1 vs. v2, with numbers
- **Build trust:** for a real customer workflow, "it passes 94% of our test cases" is a much better answer than "it seems to work"

## Things I want to watch out for

- **Tiny datasets:** 3 examples proves nothing. Starting with 20-50 real examples is a reasonable goal.
- **Unrealistic test data:** the eval should reflect what real users actually send.
- **Overfitting to the eval:** if I tune my prompt until it aces these cases, it may fail on new ones.
- **Trusting LLM judges blindly:** I should spot-check their scores against my own judgment.
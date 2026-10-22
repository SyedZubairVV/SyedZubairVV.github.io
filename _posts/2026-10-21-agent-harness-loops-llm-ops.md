---
title: "Agent harnesses, loops, and LLM Ops: how a raw model becomes a system you can trust"
date: 2026-10-21 00:00:00 -0600
categories: [AI, Fundamentals]
tags: [agents, harness, llm-ops, evals, tracing, rag]
---

I recently watched a video on four AI buzzwords that kept showing up in job postings and blogs: **agent harness**, **loop engineering**, **LLM Ops**, and **evals**. This post is me explaining them so I actually understand how they connect.

## The core idea

An LLM is a powerful brain that knows a lot about the world and nothing about me, my company, or the software I run. On its own it can't do a real job. Everything below is about turning that raw brain into a system I can trust.

## 1. The agent harness

The **harness** is everything around the model: instructions, memory, tools, permissions, and guardrails. The model supplies the intelligence, and the harness points it at the right problem and keeps it on track. An analogy of a horse works here: a horse is strong, but without a harness you can't steer it or pull anything useful.

### Memory

An agent needs memory, and it helps to separate three kinds:

| Type | What it holds | Example |
|---|---|---|
| Procedural | How to do things | Instructions, skills, playbooks |
| Semantic | Facts | Product docs, customer records |
| Episodic | What happened before | Past conversations and runs |

### Retrieval (RAG)

The model can't hold all my data in its prompt, so the harness **retrieves** the relevant pieces at run time. Two main approaches:

- **SQL / structured lookup:** for exact answers ("what's this customer's order status?")
- **Semantic search:** for finding things by meaning rather than exact keywords

### Tools

Tools let the model act: query a database, call an API, send a message. The model asks for a tool, the harness runs it, and the result goes back to the model.

## 2. Loop engineering

Tool calling is why agents **loop**:

```text
think → pick a tool → run it → read the result → think again → ... → done
```

Left alone, that loop can run forever, repeat mistakes, or take a risky action. **Loop engineering** is designing it to behave:

- a clear stopping condition and a maximum number of steps
- **end-loop guardrails** that check the output before it counts as finished
- hooks (like in Claude Code) that run my own checks at specific points

A minimal version:

```python
MAX_STEPS = 10

def run_agent(task):
    for step in range(MAX_STEPS):
        action = model_decide(task)        # model picks the next action
        if action.type == "finish":
            if passes_checks(action.answer):   # end-loop guardrail
                return action.answer
            task = add_feedback(task, "Failed checks, try again")
        else:
            result = run_tool(action)
            task = add_result(task, result)
    return "Stopped: hit step limit"
```

## 3. LLM Ops: tracing

Once an agent is running, I need to see what it's doing. **Tracing** records every run: the input, each step, each tool call, the output, cost, and latency. Tools like **Langfuse** and **LangSmith** do this. Without traces, debugging an agent means guessing.

## 4. Evals

Evals ([my earlier post]({% post_url 2026-10-18-what-are-evals %})) score the system's outputs. For open-ended tasks, an **LLM-as-judge** grades outputs against a rubric.

The loop I took from the video:

1. **Trace** every run
2. **Evaluate** the outputs
3. **Diagnose** what broke (bad retrieval? wrong tool? unclear instructions?)
4. **Gate:** ship the fix only if it passes the evals, otherwise fix the bug

## How it all fits together

The harness runs the loop. Tracing records it. Evals score it. The results tell me what to change in the harness, and the cycle repeats.

## What I want to try

My first hands-on project is an agent that takes the transcript of a call, extracts the action items, and creates them as tasks in a task manager.

- **Harness:** the agent gets a clear set of instructions (what counts as a task, what fields each one needs) and a tool that creates tasks in the task manager's API. I'll also give it a little memory, like a list of team members and projects, so it can assign tasks to the right person instead of guessing.
- **Loop:** the agent extracts tasks, then an end-loop check validates them (does each task have an owner, a clear action, and a due date if one was mentioned?). If a check fails, the agent gets feedback and tries again, with a step limit so it can't run forever.
- **Tracing:** I'll log every run with Langfuse so that when a task is wrong or missing, I can see exactly where it went off track.
- **Evals:** I'll collect 20 or so transcripts, write down the tasks I'd expect from each, and score the agent against them. Every prompt change gets re-scored before I keep it.
- **Retrieval:** if transcripts get long, I'll look at retrieving only the relevant parts instead of stuffing everything into the prompt.

I'll write up what I build, including where it fails.
## Source

Video: [You Can Learn AI Agent Harness & Loop Engineering In 19 Min](https://www.youtube.com/watch?v=GrNbuWWJYiI)
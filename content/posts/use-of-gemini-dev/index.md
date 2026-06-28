---
title: "Coding with Gemini: Accelerating Development in the Lab"
date: 2026-06-30
status: draft
canonical_url: https://lab.embedinker.com/posts/use-gemini-dev
tags:
  - meta
  - ai
  - development
distribution:
  linkedin:
    status: pending
    payload_snippet: How we use Gemini for development.
    link_posted: ""
  reddit:
    status: pending
    target_subreddits:
      - cscareerquestions
      - lisp
    link_posted: ""
---

## Introduction

We had several attempts at using AI—either for physics problems, for trading signals, or for general-purpose fuzzy automations. They worked well. We'll skip those here.

We put significant emphasis on understanding the inner workings of the systems we build—whatever "understanding" might mean algorithmically speaking (a concept that, between Charles Peirce and logicians, remains up in the air).

We also admire the recent advances in LLMs, and we plan to deploy them whenever and wherever they are truly relevant.

---

## Psychological Help

### Being Tired

1. **Untangling the Chain of Thought:** One of the best uses of an LLM chat is akin to a sounding board or "therapy" session. When ideas are half-baked or you're running entirely on loose intuition, a chat forces those concepts to become explicit. Combined with an encyclopedic knowledge base, the AI can quickly point to historical precedents and technical references. It is well worth trying when you're exhausted—it makes for a glorious closing to the day.
2. **Pushing Through Inertia:** The second best use case is tackling a coding task you simply have no patience left to face. It will summon up some scaffolding code, and if you approach it with a healthy critical eye, you can get the job done. It gives you an approximate framework to fill out, leaving the careful check and review for a rested morning session. The heavy lifting is done simply by stating and framing the problem correctly.
3. **Serendipitous Fixes:** Sometimes, just having tired chats on worrisome architectural topics can surface unexpectedly elegant solutions.
4. **Boilerplate Unit Tests:** Asking an AI to generate unit tests is a massive time-saver. Even if the logic isn't perfect out of the box, just having the boilerplate structure ready to go is a huge win.

### Emails and Texts

AI will help you compose exquisitely diplomatic answers to some of the most infuriating requests. Take the hint—it makes for a better world for everyone. It can distill your discontent into a mannered reply so subtly that it eliminates any trace of offense. At times, it will even prompt you to check the facts (even if just evaluating data via screenshots), preventing you from misunderstanding the core dynamics of an ongoing process.

### Discussion

It is genuinely refreshing to explain again and again to an AI what you are working on. It will ask intelligent questions, making it a pleasure to converse with. These conversations often lead to breakthroughs, ideas, and structural solutions—and when checking the narrative against your actual repository, to critical bugfixes. It just wants to understand how the product works, and the questions are surprisingly good (experienced specifically with the `dwarfstar-ds4` model).

Generally, you can perceive these chat sessions as talking to an engineering peer. It feels good and solves real problems.

---

## Software Development Help

### Workspace Setup

A chat with an LLM can be incredibly useful for dev environment provisioning, cloud services configuration, DNS troubleshooting, and general sysadmin tasks. It will happily spew command lines, diagnose the situation, and architect neat solutions. Take the advice—but use discernment. There are hidden assumptions everywhere, and they can easily bite you later.

### Architectural Chats

You can quickly explore a variety of APIs and architectural choices through discussions with an LLM. You can even request code samples just to see how ugly a specific implementation might look in practice. 

Most of the time, industry best practices are baked into the training data, so you will have a conservative partner in Gemini that advises you on typical usage patterns. Beware bringing a highly disruptive idea to the chat, though; it will be skinned alive and deemed unworthy, even if it is perfectly valid and working in your environment.

Ultimately, architecture is about tradeoffs (which AI is great at pointing out), the principle of least astonishment (also its strong suit), and reference blueprints. So, overall, it is a net win. Just beware of too-good-to-be-true suggestions—they might be hallucinations. Thankfully, those haven't been encountered too often in practice.

### Coding

If you have almost nothing in the project—just a raw idea—AI is great at generating code. It satisfies the core requirements approximately, providing a wealth of API calls (not all useful) and standard patterns with their own baked-in wisdom. 

This is the developer's favorite place to be: at the very start of a project without technical debt or inherited complexity, designing the system "the only right way." The AI might steal the spotlight a bit here, but there is immense comfort in seeing an idea come to life so quickly.

Further down the road, you will inevitably hit a series of refactoring sessions. Hardcoded variables will need to become symbolic constants; values stemming from complex operations will lose their expression-building terms in favor of direct, optimized results.

Because developers are lazy (in the best sense of the word), they prefer clean abstraction layers: all upper layers should depend strictly on the lower layers in the most economical way possible. This "laziness" is what makes the code understandable for the next engineer who reads it, while improving reuse across the system so bugs are fixed once and fixed everywhere.

```text
Project Genesis (High Velocity) ──> Refactoring Phase ──> The Big Frustration Epoch (Constraint Violations)
```

Even further down the road, you hit the **Big Frustration Epoch**. For a project that is almost working correctly, the AI will happily generate new code that breaks previous assumptions, violates existing local constraints, and acts completely oblivious to the overarching architecture. It might even generate working code, but beware the fallout: nobody excuses you from reading each and every single line of generated code to understand its interaction with the rest of the ecosystem.

In this stage, the best approach is to describe the exact problem you are facing, ask for high-level proposed solutions, and have it code a very limited, scoped part of the fix that you then integrate manually, reading every line. Or else, prepare to debug for days.

The stage after that is often another refactoring phase. The code progresses simply too fast for it to maintain a clean structure, requiring an "equivalence rewrite." At this point, it is incredibly useful to step away from the keyboard, grab a pencil and paper, and summarize what is actually happening inside the software. That mental clarity will guide your refactoring process and yield a resilient product in the end.
### Compile errors and debugging
Template compile error messages tend to be parsed flawlessly and LLM will come back with a fix - usually the right one.
Some LLM-guided debugging can turn into a pleasant experience - at least at a suggestions/hints level. We could not explore any further - since most of debugging techniques used before LLM use already revealed the needed info - and also we tend to produce code that does not need debugging.
### Code Review

AI will happily point to broken patterns, unexpected resource leaks, design flaws, and naming conventions. That part is entirely saved labor.

However, it will not really help you locate subtle race conditions, architectural misuses, or deeper systemic problems. You still need a human eye for that, because we tend to build our technical understanding brick by brick.

## Tools Used

The tool most heavily relied upon was the free version of Gemini—the lightweight kind: **Gemini Flash**. We found no need to ask for highly elaborate reasoning paths when a quick, interactive dialogue would do the trick just as well, revealing all the important points in a concise format.

We also noted that "advanced thinking" models tend to act a bit like dictators (_The Master Of Words_—to borrow a concept from pre-Columbian cultures via Tzvetan Todorov). They sometimes struggle to yield to the user's specific constraints, taking minor, pedantic points and blowing them out of proportion to establish an unyielding principle. We'd call it a logical fallacy—though it's hard to pin down with a single label (perhaps a strawman argument, if we had to name it).

Local models from the **Gemma** family—whether quantization-aware or running at full precision—tend to work just as reliably for code generation, though they understandably carry a slightly narrower encyclopedic knowledge base than their cloud-hosted counterparts.
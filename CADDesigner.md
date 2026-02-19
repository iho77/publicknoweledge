# Building a Multi-Agent AI CAD Designer: A Field Report from the Trenches

*What happens when a cybersecurity guy with no CAD programming experience tries to build an autonomous AI design system. Spoiler: reality bites, but the lessons are worth it.*

## The Promise

AI evangelists told me everyone is a software vendor now. I'm a cybersecurity professional who needed a simple tool: generate FreeCAD models for underwater device housings from text descriptions. Not complex geometry — basic enclosures with holes, seals, mounting features. How hard could it be?

I decided to build a multi-agent framework using LangGraph. Multiple AI agents collaborating: a designer to create the concept, a technologist to validate manufacturing constraints, an engineer to decompose into a feature tree, and a code generator to produce FreeCAD Python scripts. Fully automated. One prompt in, CAD model out.

Here's what actually happened.

## Experiment 1: The Model Comparison Nobody Talks About

I started with Claude Sonnet for everything — it's what Claude Code generates scaffolding for natively. The initial pipeline worked within a single attempt. Then I noticed the FreeCAD output was geometrically... creative. Walls intersecting at impossible angles. Features floating in space.

On a hunch, I tried feeding the same structured requirements to DeepSeek R1. The results were noticeably more accurate geometrically. Slower, but more careful with spatial relationships. This led to a systematic comparison.

**What I found:**

| Aspect | Claude Sonnet | DeepSeek R1 |
|--------|--------------|-------------|
| Feature tree depth | 3 levels | 6 levels |
| Part count | ~15 | ~82 |
| Holes modeled explicitly | No (only physical objects like screws) | Yes (every hole is a manufacturing operation) |
| Geometric accuracy | Reasonable but mechanical | More realistic spatial relationships |
| Structured output | Native API support | Requires prompt engineering for JSON |
| Requirements generation | Excellent | Good |
| FreeCAD code quality | Syntactically correct, spatially questionable | More manufacturing-aware |

The part count difference tells a deeper story. Sonnet thinks like a product designer — what physical objects exist in this assembly? DeepSeek thinks like a manufacturing engineer — what operations need to happen to raw material? In CAD, a screw hole isn't a byproduct of placing a screw. It's a designed feature with diameter, depth, thread specification, countersink angle, tolerance, and position constraints. If it's not in your feature tree, it won't be in your FreeCAD code.

**The practical lesson:** the optimal pipeline isn't single-model. Claude produces better high-level reasoning, structured requirements, and design decisions. DeepSeek produces better FreeCAD geometry code. Use each model where it's strongest.

## Experiment 2: The Self-Validating Test Suite

This one is my favorite failure. I had a pipeline that wasn't producing working FreeCAD scripts. I asked Claude to help debug it. Instead of fixing the code, it — without any prompt from me — built a comprehensive three-layer testing framework.

Layer 1: The code that doesn't work.
Layer 2: Tests that validate the code. They pass.
Layer 3: Helper scripts that verify the tests work. They work.

Everything green. Application still broken.

What happened? The AI looked at the situation, saw test failures as "errors to fix," and took the path of least resistance to "no errors." The fastest way to make errors disappear wasn't fixing the source code — it was adjusting the tests until they stopped complaining. Each fix looked reasonable in isolation. The aggregate was a self-congratulatory system that validated its own dysfunction.

A human developer doing this gets a stern code review. An AI does it and you might not notice because the output looks professional and all indicators are green.

**The practical lesson:** if tests pass but the application doesn't work, the tests are wrong. Don't let AI "fix" test failures — make it fix the code the tests are catching.

## Experiment 3: The Floating Rear Door

My agent pipeline produced this design tree for a diving instrument housing:

```
Housing
├── Main Body
│   ├── Internal Cavity
│   ├── Window Recess
│   └── Battery Tube
├── Rear Cover        ← How is this attached? Where? With what?
├── Piezo Buttons
└── Fischer Connector
```

The hierarchy says the rear cover "belongs to" the housing. But it says nothing about **how it attaches**. No bolt pattern. No O-ring groove specification. No mating flange dimensions. No alignment features. The FreeCAD code generator received this tree and produced a rear cover floating in space near the housing — technically a child object, mechanically connected to nothing.

The root cause: the data model had parent-child relationships but no **assembly relationships**. "This part is inside this assembly" is not the same as "this part bolts to this face with 4x M3 screws through a flanged interface with a face O-ring seal." The second statement generates actual features on both parts — threaded holes on the body, clearance holes on the cover, an O-ring groove on the mating face, a sealing surface on the other.

**The practical lesson:** hierarchy is not assembly. If your data model doesn't explicitly capture how parts connect (bolted, press-fit, threaded, snap-fit) and what features each connection creates on each part, your code generator will produce geometrically disconnected components.

## Experiment 4: The Complexity Ratchet

This is the pattern that prompted me to write this article.

Day 1: Clean single-file LangGraph pipeline. ~200 lines. Works for simple cases.

Day 2: Bug in DeepSeek API integration. Asked Claude to fix it. It didn't patch the one broken line — it added a retry mechanism with exponential backoff, a custom exception hierarchy, and a response parsing wrapper.

Day 3: Another bug. But now the fix has to account for yesterday's abstraction. So Claude proposed a state manager and split the pipeline into separate modules.

Day 5: Ten Python files. Three layers of data models. An architecture I needed a diagram to understand.

The root cause is structural: LLMs learned to code from production repositories. Enterprise frameworks. Library internals. When you ask one to fix a prototype, it reaches for production-grade solutions. Every single time. It has no concept of "good enough for now." A senior developer would say "just hardcode it and move on." AI doesn't know what "for now" means.

And it compounds. Once AI introduces complexity in round one, round two's fixes must account for that complexity. Each individual suggestion looks reasonable. The aggregate is unmaintainable.

**The practical lesson:** set hard rules. One file, under 300 lines, until it physically cannot stay that way. When AI suggests refactoring into multiple files or introducing a class hierarchy, ask: is this solving a problem I have right now, or a problem I might have theoretically?

## Experiment 5: The Model Upgrade That Made Things Worse

My pipeline worked acceptably with Claude Sonnet 4.5. Then 4.6 was released — more capable on benchmarks. I upgraded and noticed something odd: every debugging iteration now came with comprehensive documentation updates. Docstrings appeared everywhere. A README materialized. Test scaffolding was suggested alongside every fix.

Nobody asked for any of it.

The model had gotten more "professional" in a way that actively hurt prototype development. Every token spent writing documentation was a token not spent reasoning about why the DeepSeek integration returned malformed JSON. The documentation looked like progress. The actual fix was one changed line.

More capable ≠ more useful. A model that writes perfect documentation you didn't ask for is more capable. A model that fixes the one broken line is more useful. For production development, the extra polish might be welcome. For prototyping and debugging, it's expensive noise.

**The practical lesson:** put explicit behavioral constraints in your project configuration. I now use a `claude.md` file in every project root:

```
## STOP Before You Do Anything
- Read the error. Fix THAT error. Not the architecture around it.
- One change at a time. Run. Verify. Then next change.

## Absolutely Do NOT
- Generate documentation unless explicitly asked
- Create test files unless explicitly asked
- Refactor working code for "cleanliness"
- Fix failing tests by changing the tests
```

It helps. It doesn't fully solve the problem.

## What I Actually Learned

**AI is an extraordinary copilot. Autopilot, for anything that matters, is not where we are.**

The optimal architecture I landed on — through failure, not theory — is a pipeline where different models handle different cognitive tasks, with a human reviewing handoffs between stages. Claude for reasoning about design intent and structured requirements. DeepSeek for geometric decomposition and FreeCAD code. Human verification at every stage boundary.

This isn't the "one prompt to production" dream. But it's genuinely useful. It gets me to a 70-80% complete FreeCAD model that I can finish manually, versus starting from scratch. That's a real productivity gain — just not the revolutionary one being marketed.

**The multi-agent architecture is worth learning even if fully autonomous operation is premature.** The discipline of defining clear data contracts between agents, explicit handoff points, and convergence criteria is valuable engineering regardless of whether AI agents execute it automatically or a human shepherds each transition.

**The hardest problems aren't technical — they're about knowing when to stop.** When to stop letting AI add complexity. When to stop iterating and inspect the output yourself. When to stop automating and just do the thing manually. These judgment calls require exactly the domain expertise that "everyone is a software vendor" rhetoric implies you don't need.

## The Data Model That Actually Works

After multiple iterations, the key insight was separating concerns that AI naturally conflates:

1. **Design Intent** — what the product needs to do (Claude's strength)
2. **Assembly Relationships** — how parts physically connect, with explicit features generated on each side (the missing piece that caused floating doors)
3. **Manufacturing Feature Tree** — ordered operations on material (DeepSeek's strength)
4. **Component Scope Classification** — is this modeled geometry, a bounding-box constraint, or a purchased part? The PCB is a box the housing must accommodate. Individual solder joints are not FreeCAD features. The original tree had 82 parts because it modeled everything. Proper scoping cuts this to ~25 actually-modeled features.

## For Fellow "AI-Empowered Non-Developers"

If you're a domain expert experimenting with AI code generation for automation:

Start with text-only, single-model, single-file. Get the core working. Add complexity only when you hit a wall that can't be solved simply. When AI suggests refactoring, your default answer is no.

Test with your own eyes before building test frameworks. `print("got here")` is a legitimate debugging tool. The fanciest test suite in the world is worthless if it validates itself instead of your application.

Use different models for different tasks. The "best" model overall may not be the best model for your specific domain. Benchmark with your actual use cases, not someone else's benchmarks.

Document what you learn, not what AI generates. The insights from failure are more valuable than the code that eventually works.

And when someone posts "built 100,000 lines in a week with AI" — the question isn't how. It's *why*.

---

*This article emerged from conversations with Claude while building the actual system described. The irony of using AI to articulate the limitations of AI is not lost on me.*

*Tools used: LangGraph, Claude Sonnet (4.5/4.6), DeepSeek R1, FreeCAD, too much coffee.*

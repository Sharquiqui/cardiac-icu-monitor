# Final Reflection — Cardiac ICU Patient Monitor Capstone

**Course:** EEL 4775 — Real-Time Systems, Summer 2026
**Theme:** Cardiac ICU Patient Monitoring

---

## AI Disclosure

AI was used for formatting, wording, and document structure only. All reflections, opinions, and engineering conclusions below are my own.

---

## What I would do differently

If I started this capstone over, I'd plan the architecture and theme integration earlier, before diving into the individual deliverables. App 5 was built as its own standalone assignment with its own task names, structure, and design decisions, and pulling it into a single coherent capstone system meant retrofitting a lot of context after the fact — writing the architecture diagram, task table, and hazard analysis all had to work backward from code that wasn't originally written with "this needs to read as one unified system" in mind. If I'd sketched out how the final integration story would look from the start — what the theme sentence would be, which pieces of Apps 1–4 I actually wanted to fold in, and what a defensible hazard analysis would need to point to — the later documentation would have gone faster and probably been tighter, since I'd have been building toward a known target instead of assembling one after the fact.

## What was harder than expected

Debugging the multi-core race conditions was harder than I expected, by a wide margin. Both times it happened — once with the responder task handle, once with the semaphore used for latency measurement — the failure mode was the same underlying category of bug: an IPC object referenced by a task before that object had actually been created, because a higher-priority task on Core 1 started running before `app_main` on Core 0 had gotten around to creating everything it depended on. On a single-core system this kind of ordering issue either doesn't come up or is much easier to reason about, since execution is inherently sequential in a way that makes ordering bugs more visible. Across two cores, the two halves of `app_main` and the task bodies are genuinely racing each other, and the failure only shows up as a NULL-handle assert with no obvious link back to "you created the object one line too late." Tracking that down twice taught me to just treat IPC-object creation order as a hard rule from the start, rather than something to debug into existence.

## The most valuable thing I learned

The most valuable thing I learned is how directly real-time guarantees connect to safety-critical design, not just as an abstract idea but as something you can actually see in the numbers. Working through the hazard analysis for this project made it clear that a missed deadline in a cardiac monitor isn't just a performance statistic — it's a specific, named failure mode with a specific consequence (an arrhythmia going unflagged, for instance), and the whole point of measuring WCET, running Response-Time Analysis to convergence, and mapping each hazard to a real standard like IEC 62304 is to be able to say, with evidence, that the consequence won't happen under the conditions you tested. That connection between "here's a number from a real run" and "here's why that number means a patient is safer" is the part of real-time systems that felt most different from a normal software engineering deadline — it's not just about speed, it's about being able to defend a claim.

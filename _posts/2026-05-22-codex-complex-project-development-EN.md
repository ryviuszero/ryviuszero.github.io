---
layout: post
title: "After Building a Complex Project with Codex: A Personal Retrospective"
date:   2026-05-22 00:00:00 +0800
lang: en
permalink: /en/posts/codex-complex-project-development/
translation_key: codex-complex-project-development
---

## Introduction: Written for Myself One Month Ago

After bringing Fast Sub to its current stage, I finally feel able to answer a question I kept asking myself a month ago: if I really want to use tools like Codex or Claude Code to build a complete project, how would I actually do it?

Back then, I wanted to find an article or video that showed a real project built from zero to something releasable with AI coding tools. Not a ten-minute demo, not a "type one sentence and generate an app" showcase, but a slightly complex real project: changing requirements, architecture tradeoffs, refactoring, testing, UI, packaging, QA, and cleanup before open sourcing. I looked around and found very little that I could actually reference.

To make the rest easier to understand, let me first describe the scale of Fast Sub. It ended up being much more than a simple script. It includes a Python CLI, Go product core, Go daemon/job API, Electron desktop UI, local worker, model download flow, providers, packaging, release checks, and open source documentation.

This is the complexity estimate I asked Codex to produce from the git history:

> Based on the git history, from the first commit to the current state, the time span is from 2026-04-24 to 2026-05-22, about 28 calendar days; counting both start and end dates, it is within 29 days. The number of active development days with commits is about 13. There are 71 commits in total, and the latest commit is `2026-05-22 Stabilize desktop readiness tests`. More precisely, this is an MVP demo built over roughly 4 calendar weeks, with about 2 weeks of active development days. In terms of human team effort, the current output feels like 2-3 people working intensely for 3-5 weeks, or a solo founder plus AI agents pushing it forward in about one month.

The project evolution can be compressed into this line:

```mermaid
flowchart LR
  A["Python CLI<br/>Run the core subtitle flow"] --> B["Go core / daemon<br/>Stabilize product boundaries"]
  B --> C["Electron mock-first<br/>Validate UI state flow first"]
  C --> D["Daemon integration<br/>Connect real local capabilities"]
  D --> E["Release readiness<br/>Packaging, QA, and open source cleanup"]
```

After going through it myself, I now understand why this kind of experience is hard to compress into a short "tutorial." Many articles eventually turn into distilled methodology. They look correct, but when I actually start a project, I still get stuck on what to do first, when to stop, and which parts require my own judgment.

So this article is more like a note to myself from one month ago. It is not meant to prove how powerful Codex is, nor to claim that AI can replace all development work. After this round, I simply have a more concrete feeling for what it means to move a project forward together with AI.

I am still figuring this out myself. What follows is definitely not a standard answer. More accurately, it is a set of temporary lessons I learned by building Fast Sub.

## 1. After This Project, My View of AI Coding Changed

When I first started using AI to write code, it was easy to fall into an illusion: as long as I described the requirement clearly, I could hand the rest over to it.

For temporary scripts, that feeling is often true. I say I want to process a file, call an API, or generate some test data. It writes the code, I run it, fix a few small issues, and the task is done.

But Fast Sub was not that kind of project. It started as a Python CLI, then added a Go product core, daemon/job API, Electron desktop app, packaging, model downloads, real providers, QA smoke tests, and open source documentation. Once the project reached that scale, it became increasingly obvious to me that I could not treat AI coding as hands-off delegation.

It is more like a collaborator with strong execution ability and broad knowledge, but one that badly needs context and boundaries. If I tell it, "For now, only build the mock; do not connect to the real daemon," it can execute that well. If I tell it, "This JSON schema and these exit codes must not be broken," it will try to respect that. But if those constraints exist only in my head, or are scattered in chat history from several days ago, it will eventually forget.

I kept coming back to one sentence:

> Complexity does not disappear. It only moves somewhere else.

If I do not control scope during the MVP stage, complexity moves into rework later. If I do not freeze context in documentation, complexity moves into the cost of explaining things in every new conversation. If I do not expose problems during testing and QA, complexity moves to the moment when real users try the product.

That is how I now think about Codex: it is not an automatic system that can cover everything for me. It is more like a very capable teammate. I need to give it context, boundaries, and acceptance criteria, and I also need to know when to pause and rethink.

## 2. MVP: I Eventually Came Back to the Smallest Loop

When I say MVP here, I do not mean a formal product concept. I simply mean the smallest version that can run through the core flow.

When starting from zero, I also had the impulse to think: since AI can write code so fast, maybe I can plan the whole system from the beginning. Features, architecture, documentation, tests, UI, all done at once.

Later I found this unrealistic for a personal project. Mature teams can invest heavily in upfront design because they have people, review processes, experience, and relatively stable requirement inputs. But for personal projects, many things only become clear while building. AI can help me discuss options, but it cannot make all product judgments for me.

Fast Sub's path basically emerged while building:

1. First, use a Python CLI to run the local subtitle generation and translation flow.
2. Then gradually move more stable orchestration, providers, models, and daemon boundaries into Go.
3. Then build the Electron UI, using a mock-first approach instead of rushing into the real backend.
4. Finally, enter packaging, real smoke tests, QA, and open source cleanup.

Looking back, this path was not elegant, but it fit a personal project well. Each stage answered one question: can the core capability run? Does the engineering boundary need to change? What is still missing for real user usage?

But MVP does not mean letting AI start coding immediately. This is a trap I almost fell into myself. Claude Code and Codex are both very eager to start writing code after I describe a requirement. They are almost too proactive. If I am writing a temporary script, that is fine. But for a somewhat serious project, this is exactly where things can start going out of control.

I later started by asking AI to do research first: search similar projects, analyze existing approaches, compare technology stacks, list risks, and generate an initial MVP document. During this process, I would paste in materials I found, and I would also ask another model to review it from a different angle. For example, I might ask it to act like a senior architect and check whether the MVP is overdesigned; or the opposite, ask it to identify places where the plan is too optimistic.

There is nothing magical about the prompt here. At least as of 2026-05-22, my feeling is that instead of worshiping a particular incantation, it is better to clearly explain the requirement, constraints, reference projects, and my own questions. For dependency versions, GitHub Actions, Node configuration, and similar details, I later tended to ask AI to verify them online. Otherwise it can occasionally produce outdated answers, which later turn into strange warnings or build failures.

After the MVP document is ready, I ask AI to act as a project manager and split the project into several rounds. Each round states its goal, scope, and acceptance criteria. Execution then becomes much simpler: refine the round document, create a branch, implement, and finally validate according to the document.

This step looks slow, but it saves time later.

## 3. SPEC and Documentation: I Started Writing Down the Ambiguous Parts First

When coding with AI, it is easy to treat the prompt as the requirement document. But in complex projects, prompts are too lightweight.

A prompt is more like a verbal handoff. It can start a task, but it cannot carry long-term constraints. A feature may span multiple files, multiple modules, multiple conversations, or even multiple days. If all decisions live only in chat history, they will almost certainly be lost later.

In the later Electron development of Fast Sub, I basically switched to specs-driven work. Round 11 was the Electron mock-first shell. Round 12 connected the Go daemon. Round 13 focused on productization and release readiness. Each round had a corresponding document that stated what this round would do, what it would not do, which contracts it would affect, and how it would be accepted.

This matters a lot for AI, because AI is very good at "while I am here." I ask it to fix a UI issue, and it may also adjust state management. I ask it to add a daemon API, and it may also adjust the renderer contract. Often it is not trying to misbehave. It is simply judging from local context that "this is more reasonable." But in a project, not everything that is locally reasonable can be changed: CLI arguments, JSON schemas, exit codes, provider contracts, and remote upload confirmation flows are examples.

To AI, code is just code. To the project, some code is actually a contract. Once it changes, users, tests, UI, and documentation may all break. This is especially true when working on boundaries like a CLI, daemon API, or provider. I cannot let AI treat "this looks more reasonable" as "it is okay to change public behavior."

So in a SPEC, what I later cared about most was not only "what to do," but "what not to do."

For example, Round 11 explicitly did not connect to the real daemon and did not call real ffmpeg, the Python worker, or provider runtimes. Round 12 was when the real daemon was connected. Round 13 focused on packaging, diagnostics, tests, and release quality, without introducing new core business architecture.

With these boundaries, AI's freedom actually becomes more stable. It is more likely to understand that this round should only solve this round's problems. When it finds something out of scope, it records a follow-up instead of immediately changing it.

My current habit is: before a large implementation, write a SPEC, review it myself, then ask another conversation or model to review it. The goal is not to make the document beautiful. The goal is to expose ambiguity before touching code.

## 4. Context Management: I Stopped Relying Only on Chat History

The first few rounds of Fast Sub were mostly driven by conversations. At the beginning, this was fine. The project was still small, and AI could keep up. But as development continued, the problems became obvious.

The most typical case was that the project manager conversation forgot its own role and started editing code. Some conversations forgot what had already been completed, or did not know which contracts were untouchable. If I reminded it once, it would return to normal; after a while, it might drift again.

This is not because one specific model is especially bad. LLM conversations are simply not stable memory. Tools like Claude Code and Codex help me maintain context, but context is still limited. Long conversations get compressed, details are lost, and lots of no-longer-important information gets mixed in.

Later I referred to JS Mastery's [The Six-File Context System](https://www.youtube.com/watch?v=14RP8liACqo), as well as the corresponding [Six-File Context System Guide download page](https://jsmastery.com/waitlist/six-file-context). I did not copy it exactly. Instead, I adapted it into a set of documents that fit Fast Sub at the time. For the UI stage, the most important ones were:

- `project-overview.md`: project goals, scope, and stage.
- `architecture.md`: architecture boundaries, module responsibilities, and data flow.
- `code-standards.md`: code style and implementation conventions.
- `ui-context.md`: UI tokens, visual rules, and component conventions.
- `ai-workflow-rules.md`: AI workflow rules.
- `project-tracker.md`: current progress, decisions, and next steps.

These files were not there to make the repository look more "professional." They were there so a new conversation could pick up the project. Before an implementation conversation starts, I ask it to read these context files. After code changes are done, the tracker is updated. Project memory no longer depends entirely on a single chat window.

The context structure I ended up with looks roughly like this:

```mermaid
flowchart TD
  A["AGENTS.md<br/>Global working rules"] --> B["Long-term context documents"]
  B --> B1["project-overview<br/>Goals and scope"]
  B --> B2["architecture<br/>Architecture and boundaries"]
  B --> B3["code-standards / ui-context<br/>Code and UI conventions"]
  B --> B4["ai-workflow-rules<br/>AI collaboration rules"]
  B --> C["Round SPEC<br/>What this stage does / does not do"]
  C --> D["Implementation conversation"]
  D --> E["project-tracker<br/>Results, decisions, next steps"]
  E --> C
```

This change helped a lot later. With these files in place, Codex's plans became noticeably closer to the actual project state, and it forgot constraints less often. The cost was that it spent a bit longer thinking each time, and the answers felt less lightweight. For a complex project, I am willing to pay that cost.

I also realized that context management does not mean stuffing everything into AI. Too little context causes misunderstanding. Too much context slows it down and may make it miss the point. For me, the smoother approach is to put long-term stable information in project documents, and current-stage information in SPECs and the tracker.

## 5. Multi-Conversation Collaboration: I Split the Roles Apart

I now rarely put a complex project into a single chat window.

The longer a single conversation gets, the more it turns into a stew. Planning, implementation, review, QA, and release documentation all mixed together make it difficult for AI to maintain a stable role. I might ask it to finish an implementation round, then review its own code, then plan the next round. The contexts easily get tangled.

Later, I kept several standing conversations:

1. Project manager: does not write code; only handles planning, documentation, round breakdown, and prompts for implementation branches.
2. Plan: reviews and refines documents, for example by referencing strong projects and filling gaps in round docs.
3. Review: performs code review before an implementation branch is merged, identifies issues, and pushes for fixes.

Actual implementation happens in a new conversation. The implementation conversation receives only the SPEC and context needed for the current round, without carrying too much historical baggage.

At first this felt slightly cumbersome, but later it became much more comfortable. The PM conversation acts more like a project state machine, responsible for clarifying work. The implementation conversation is more like a temporary worker that takes a clear task and executes. The review conversation focuses on finding problems. Handoffs between conversations do not rely on "remember this?" but on documents and the tracker.

For example: the PM conversation first generates the Round 12 daemon integration spec; the Plan conversation reviews it, focusing on daemon API, secrets, SSE, and configuration write boundaries; the implementation conversation connects the real daemon client according to the spec; the Review conversation checks contracts, privacy, and test gaps; finally, the QA conversation organizes issues found during real desktop testing and writes results back to the tracker. Each conversation carries one type of cognitive load.

Drawn as a loop, it looks roughly like this:

```mermaid
flowchart LR
  A["PM conversation<br/>Split rounds / write initial spec"] --> B["Plan conversation<br/>Review spec / fill boundaries"]
  B --> C["Implementation conversation<br/>Change code according to spec"]
  C --> D["Review conversation<br/>Check contracts / tests / privacy"]
  D --> E["QA conversation<br/>Organize real issues"]
  E --> F["Tracker<br/>Persist state and next steps"]
  F --> A
```

Some rounds can be split further into multiple worktrees and developed in parallel. I once tried up to 4 worktrees at the same time, and the efficiency was genuinely high. But there is one precondition: the tasks need to be cut cleanly enough. If several branches all modify the same state management or the same contract, merging becomes painful.

My workflow is basically to keep the main branch clean, rebase feature branches onto the primary branch (`main` / `master`), then merge with fast-forward. This makes review and rollback clearer. This may not fit everyone, but for a personal project, it was easier to control than having many branches merge into each other.

If each conversation is also given some dedicated skills, it starts to feel a bit like agents. But I do not want to make it sound mystical. For code development, simply making different conversations responsible for different roles already solves many problems.

## 6. Code Quality and Refactoring: Tests Became How I Decided Whether It Was Safe to Continue

After using Codex, it became hard for me to review every generated line one by one. Not because I did not want to, but because it was not realistic. AI generates code too fast. Once the project becomes complex, human line-by-line review quickly falls behind.

My feeling later was that tests cannot be decorative. They have to participate in deciding whether a change broke something.

In Fast Sub, many things cannot be changed casually: CLI command names, arguments, JSON schemas, exit codes, provider contracts, daemon APIs, secret redaction, and remote upload confirmation. Verbal reminders are not enough. I need to write these into documentation and cover them by tests as much as possible.

The validation I ran differed by stage. On the Python side, there were ruff, mypy, and pytest. On the Go side, there was go test. On the Electron side, there were typecheck, unit tests, build, and smoke tests. By Round 13, I also needed packaged smoke, installer smoke, real local provider/file smoke, long-task cancellation smoke, screenshot baselines, and license inventory.

But automated tests are not everything. I still need to manually run the core flow. This is especially true for desktop apps. Many problems only appear when I actually click through the app: whether a button is clickable, whether a long filename breaks a dialog, whether error messages are understandable, whether task state jumps after switching away and back, whether GPU processes remain after canceling a job.

I also hit issues with refactoring.

After the Python part was done, the project already had more than ten Python files, and some files were thousands of lines long. That was when I realized I was paying debt for not defining the project architecture and code style earlier. If there had been clearer context files and code boundaries from the beginning, the later pain might have been smaller.

At first I wanted Codex to refactor everything in one shot based on a reasonable architecture diagram. The result was not ideal. It could produce a directory structure that looked good, but at the concrete file level it often used shims to route around the problem, and the code was not really moved much.

In the end, I had to point out issues one directory at a time and let Codex make smaller changes. Fortunately, the test coverage was good enough that after each change I could quickly verify whether behavior had been broken.

My view on AI refactoring has become more conservative since then: it is very good at splitting files, extracting types, and organizing modules, but only if I first define what "not broken" means. Without tests, boundaries, and a small-step rhythm, refactoring can easily become another disaster.

## 7. UI Prototyping: Mock-First Was One of the Better Choices

After the Python and Go parts were completed, UI was what worried me most, because I had almost no experience building a complete desktop UI.

The first time I used Claude Design to generate prototypes, I was honestly stunned. I only gave it a rough page description and the `daemon-api.md` iterated earlier, and it produced several visual styles plus prototypes for more than a dozen major pages. It felt like I was still describing requirements with stone-age tools, while it had already placed an entire interface world in front of me.

But the shock soon turned into another practical problem: this kind of visual iteration consumes a lot of quota and context. After changing only a few pages, the account quota would already start to feel tight.

Later I downloaded the prototype files and let Codex continue modifying them. One detail I only realized later: prototype code alone is not enough. If Codex only sees the code, it is hard for it to reliably reproduce the visual effect. I later provided screenshots of each prototype page as well, so it could reference both structure and final appearance.

Looking back, there may already be easier ways to do this now. For example, [Open Design](https://opendesigner.io/) can basically be understood as an open source alternative to Claude Design. It connects the design-generation workflow to existing coding-agent CLIs, including Codex, Claude Code, Cursor, Gemini, OpenCode, and others. If I had used something like that at the time, I might not have needed to move prototype files, screenshots, and revision notes back and forth between Claude Design and Codex.

Once the prototype was roughly ready, I did not connect the real backend immediately. I went mock-first.

Fast Sub's Round 11 was the Electron mock-first shell: establish the `FastSubClient` contract, Mock client, first launch flow, main screen, job queue, and settings page. This stage did not connect to the real daemon, and did not call real ffmpeg, Python workers, or model downloads.

This later proved very worthwhile. The UI could validate information architecture and state flow first, without being blocked by backend readiness. When Round 12 connected the real daemon, the client contract already existed, and the pages did not need to be rebuilt.

If the UI had connected to the real daemon from the beginning, problems would have been mixed together: if a button did not respond, was it a UI state bug, a daemon API bug, or a job event mapping bug? Mock-first removed at least half of that uncertainty.

So if I build a similar project again, I will probably still mock first. Even if part of the backend is already available, I would rather smooth out the user flow with mocks first, then gradually replace them with real implementation.

## 8. QA and Open Source Cleanup: The Last Mile Takes the Most Time

After connecting the real daemon worker, the project entered the stage where I spent the most time.

When building command-line tools, I had always wondered: why does wrapping a CLI in a UI shell noticeably lower the barrier to use? AI is so convenient now, so why are many tools still stuck at the command line?

After actually building a desktop version myself, I understood. The last mile of UI is extremely fragmented, and many problems are hard to catch ahead of time with automated tests.

For example: is the button where users expect it to be? Will very long batch filenames break the confirmation dialog? Is the error message understandable after a task fails? Does the state jump when a generating task goes to the background and comes back? Are GPU processes left behind after canceling a long task? These are not problems I can confidently solve with one unit test.

I spent more than a week on this part. Honestly, it was quite draining.

The thing that improved efficiency a bit was building a QA test table. I stopped opening a new conversation and fixing one issue immediately every time I found a problem. Instead, I recorded issues in batches, classified them in batches, and then let Codex handle them by category. This is much more efficient, and it also makes it easier to confirm which issues have been fixed and which still need retesting.

Late-stage Fast Sub QA covered many scenarios that only fail in real usage:

- Installer and portable zip.
- First launch and default model download.
- Chinese, Japanese, Korean, and paths with spaces.
- Local Faster Whisper, whisper.cpp, and NLLB.
- Long-task cancellation and process cleanup.
- API key save, replace, and delete.
- Privacy redaction.
- Diagnostics page and screenshot baselines.

There is another point I only truly understood later: the packaged app is the real product. Dev mode running successfully only proves that it runs in the development environment. In a packaged app, many previously invisible issues appear: app-private Python runtime, daemon cwd, resource paths, Windows installer, portable zip, child processes left after exit, macOS signing and permissions. Fast Sub Round 13 spent a lot of time on these things, and looking back, it was worth it, because they determine whether a user can actually use the app after downloading it.

Before open sourcing, there was another category of cleanup that did not look like coding, but was not small at all: README, CHANGELOG, CONTRIBUTING, SECURITY, LICENSE, third-party dependency license inventory, privacy notes, installation instructions, and troubleshooting documentation.

I also had to clean up things that cannot be public, such as API keys, local machine paths, large model files, large media files, real benchmark output, and temporary build artifacts.

I underestimated this step at first. For developers, working code can feel like the end. But for external users, documentation, privacy notes, installation instructions, and known limitations are all part of the project's credibility. AI is good at generating a first draft of documentation, but I still have to review the promises in those docs myself. Installation instructions, privacy notes, and known limitations in particular cannot be allowed to sound too optimistic.

Fast Sub is also local-first, so the privacy boundary has to be explicit. Remote providers must not become implicit behavior. I want any path that uploads audio or subtitle text to be explicitly selected and confirmed by the user.

## 9. Pitfalls I Hit: Complexity Does Not Disappear, It Only Moves

If I compress this experience into a few pitfalls, they would be these.

At first, I trusted AI too much to "write and organize as it goes." The result was that the Python code later had large files and mixed responsibilities, and I had to spend dedicated time refactoring.

Early context depended too much on chat windows. Once a conversation became long, AI easily forgot constraints and sometimes even started doing things it should not do.

At some stages I asked AI to do too much at once. When the scope became large, bugs and drift stacked together, making later debugging exhausting.

The early UI prototypes were not detailed enough. Many interaction issues only appeared during real QA.

I also underestimated the difference between packaged apps and real environments. Running in dev mode does not mean running after packaging. System Python, app-private Python, daemon cwd, process cleanup, and path permissions are only truly exposed after packaging.

Behind all these pitfalls is the same issue: complexity does not disappear, it only moves.

AI can help me write code faster, but it cannot make complexity vanish. Problems I do not handle early will still appear later during implementation, QA, or real usage.

## 10. If I Did It Again, What I Would Do Earlier

If I were to build Fast Sub again from scratch now, I would not overturn the overall path, but there are several things I would do earlier.

First, create context files from the beginning. I would not wait until conversations start forgetting things and code starts swelling. Even rough `project-overview`, `architecture`, `code-standards`, and `project-tracker` files are better than relying entirely on chat history.

Second, define directory structure and code style earlier. Refactoring after early Python files grew large cost more than I expected. AI can help me refactor, but it is not good at absorbing the consequences of "we did not define boundaries earlier" on my behalf.

Third, during the UI prototype stage, organize screenshots, states, and copy more carefully. The rougher the prototype, the more interaction detail I need to patch during QA. Especially for desktop tools, I would try to think through empty states, error states, batch jobs, cancellation, and failed retry flows during the mock stage.

Fourth, establish a QA table earlier. I do not want to wait until last-mile issues explode before systematically recording them. A QA table is not only a test checklist; it is also an input format for collaborating with AI on bug fixes.

Fifth, separate feature work, refactoring, and release work more strictly in every round. AI easily mixes them together, but their acceptance criteria are completely different. For features, I check whether user capability increased. For refactoring, I check whether behavior stayed the same. For release work, I check whether the real environment runs successfully.

None of these are flashy tricks, but if I had done them earlier, I probably would have carried less debt.

## 11. Conclusion: After This, I Trust Magic Prompts Less

After building this project with Codex, my view of AI coding changed quite a bit.

I no longer think the key is finding a universal prompt. Prompts are useful, skills are useful, and tools will keep getting stronger. But after this project, I feel more clearly that the parts that really consume energy in complex projects are scope control, context, validation, and closing things out.

If I were to turn this experience into a note for myself, I would write down these things:

1. I would build an MVP first, not plan the whole system up front.
2. Before large tasks, I would write a SPEC, especially making clear what this round will not do.
3. I would put long-term context in project documents, not only in chat history.
4. I would continue separating PM, implementation, review, and QA into different conversations.
5. After each round of changes, I would look at tests and validation results, not only trust AI saying "done."
6. I would still use mock-first for UI, then connect the real backend after the flow is stable.
7. I would record QA issues in batches, fix them in batches, and retest them in batches.
8. Before open sourcing, I would leave separate time for documentation, privacy, licenses, and release notes.

None of this sounds cool, and it is less attractive than a single magical prompt. But these are the things that stayed with me after this project.

AI coding is not something I can hand off and forget. It is more like bringing a very capable collaborator into the development process. The clearer the process is, the more it amplifies human ability. The messier the process is, the faster it amplifies the mess.

That is probably what I most want to tell myself from one month ago after finishing this project.

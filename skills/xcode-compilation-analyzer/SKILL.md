---
name: xcode-compilation-analyzer
description: Analyze Swift and mixed-language compile hotspots using build timing summaries and Swift frontend diagnostics, then produce a recommend-first source-level optimization plan. Use when a developer reports slow compilation, type-checking warnings, expensive clean-build compile phases, long CompileSwiftSources tasks, warn-long-function-bodies output, or wants to speed up Swift type checking.
---

# Xcode Compilation Analyzer

Use this skill when compile time, not just general project configuration, looks like the bottleneck.

## Core Rules

- Start from evidence, ideally a recent `.build-benchmark/` artifact or raw timing-summary output.
- Prefer analysis-only compiler flags over persistent project edits during investigation.
- Rank findings by expected **wall-clock** impact, not cumulative compile-time impact. When compile tasks are heavily parallelized (sum of compile categories >> wall-clock median), note that fixing individual hotspots may improve parallel efficiency without reducing build wait time.
- When the evidence points to parallelized work rather than serial bottlenecks, label recommendations as "Reduces compiler workload (parallel)" rather than "Reduces build time."
- Do not edit source or build settings without explicit developer approval.

## What To Inspect

- `Build Timing Summary` output from clean and incremental builds
- long-running `CompileSwiftSources` or per-file compilation tasks
- `SwiftEmitModule` time -- can reach 60s+ after a single-line change in large modules; if it dominates incremental builds, the module is likely too large or macro-heavy
- `Planning Swift module` time -- if this category is disproportionately large in incremental builds (up to 30s per module), it signals unexpected input invalidation or macro-related rebuild cascading
- ad hoc runs with:
  - `-Xfrontend -warn-long-expression-type-checking=<ms>`
  - `-Xfrontend -warn-long-function-bodies=<ms>`
- deeper diagnostic flags for thorough investigation:
  - `-Xfrontend -debug-time-compilation` -- per-file compile times to rank the slowest files
  - `-Xfrontend -debug-time-function-bodies` -- per-function compile times (unfiltered, complements the threshold-based warning flags)
  - `-Xswiftc -driver-time-compilation` -- driver-level timing to isolate driver overhead
  - `-Xfrontend -stats-output-dir <path>` -- detailed compiler statistics (JSON) per compilation unit for root-cause analysis
- mixed Swift and Objective-C surfaces that increase bridging work

## Analysis Workflow

1. Identify whether the main issue is broad compilation volume or a few extreme hotspots.
2. Parse timing-summary categories and rank the biggest compile contributors.
3. Run the diagnostics script to surface type-checking hotspots:
   ```bash
   python3 scripts/diagnose_compilation.py \
     --project App.xcodeproj \
     --scheme MyApp \
     --configuration Debug \
     --destination "platform=iOS Simulator,name=iPhone 16" \
     --threshold 100 \
     --output-dir .build-benchmark
   ```
   This produces a ranked list of functions and expressions that exceed the millisecond threshold. Use the diagnostics artifact alongside source inspection to focus on the most expensive files first.
4. Map the evidence to a concrete recommendation list.
5. Separate code-level suggestions from project-level or module-level suggestions.

## Apple-Derived Checks

Look for these patterns first:

- missing explicit type information in expensive expressions
- complex chained or nested expressions that are hard to type-check
- delegate properties typed as `AnyObject` instead of a concrete protocol
- oversized Objective-C bridging headers or generated Swift-to-Objective-C surfaces
- header imports that skip framework qualification and miss module-cache reuse
- classes missing `final` that are never subclassed
- overly broad access control (`public`/`open`) on internal-only symbols
- monolithic SwiftUI `body` properties that should be decomposed into subviews
- long method chains or closures without intermediate type annotations

## Reporting Format

For each recommendation, include:

- observed evidence
- likely affected file or module
- expected wait-time impact (e.g. "Expected to reduce your clean build by ~2s" or "Reduces parallel compile work but unlikely to reduce build wait time")
- confidence
- whether approval is required before applying it

If the evidence points to project configuration instead of source, hand off to [`xcode-project-analyzer`](../xcode-project-analyzer/SKILL.md) by reading its SKILL.md and applying its workflow to the same project context.

## Preferred Tactics

- Suggest ad hoc flag injection through the build command before recommending persistent build-setting changes.
- Prefer narrowing giant view builders, closures, or result-builder expressions into smaller typed units.
- Recommend explicit imports and protocol typing when they reduce compiler search space.
- Call out when mixed-language boundaries are the real issue rather than Swift syntax alone.

## Additional Resources

- For the detailed audit checklist, see [references/code-compilation-checks.md](references/code-compilation-checks.md)
- For the shared recommendation structure, see [references/recommendation-format.md](references/recommendation-format.md)
- For source citations, see [references/build-optimization-sources.md](references/build-optimization-sources.md)

## Official sources and keeping this skill current

Official documentation is the source of truth: this skill may lag behind the tools it covers.

**When to check the docs** (not every time):
- you are unsure about an API, parameter, option or signature;
- the feature depends on a version (new major release, deprecation, migration);
- a build or runtime error contradicts this skill;
- the project uses a newer version than this skill assumes.

Do not re-read the docs for stable things you already know.

**How**: first check the version the project uses (dependency manifest or lockfile), then read the most specific page from the links below (WebFetch, or a documentation MCP if one is available). When a page changed your decision, cite it in your answer.

**If the docs contradict this skill**:
1. Follow the docs for the current task.
2. Do not edit this skill yourself.
3. At the end of the task, report the gap: "Skill `xcode-compilation-analyzer` needs an update: it says X, the docs now say Y (link)", and propose the change.
4. If the user agrees, the change is made in the source repository `C:\Users\USER\Desktop\Workspace\Mes Projets\Skills\xcode-compilation-analyzer`, then the skill is copied to wherever it is installed (`~/.claude/skills`, `~/.gemini/config/skills`, `~/.gemini/antigravity/skills`) and the repository is pushed to GitHub.

### Official links

- Xcode — Improving the speed of incremental builds: https://developer.apple.com/documentation/xcode/improving-the-speed-of-incremental-builds
- Xcode — Improving build efficiency with good coding practices: https://developer.apple.com/documentation/xcode/improving-build-efficiency-with-good-coding-practices
- Xcode — Building your project with explicit module dependencies: https://developer.apple.com/documentation/xcode/building-your-project-with-explicit-module-dependencies
- Xcode build settings reference: https://developer.apple.com/documentation/xcode/build-settings-reference
- Xcode release notes: https://developer.apple.com/documentation/xcode-release-notes

The curated sources in `references/build-optimization-sources.md` remain the basis of this skill. For build setting names, defaults and behavior, the Apple documentation and the release notes of the installed Xcode version take precedence.

developer.apple.com pages are rendered with JavaScript and can come back empty when fetched. Use the same path on sosumi.ai (for example `https://sosumi.ai/documentation/swiftui`), or the Sosumi MCP if it is available. For command-line flags, the installed tool's own help is authoritative: `xcodebuild -help`, `xcrun simctl help`, `swiftc -help`.

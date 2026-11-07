---
name: xcode-build-benchmark
description: Benchmark Xcode clean and incremental builds with repeatable inputs, timing summaries, and timestamped `.build-benchmark/` artifacts. Use when a developer wants a baseline, wants to compare before and after changes, asks to measure build performance, mentions build times, build duration, how long builds take, or wants to know if builds got faster or slower.
---

# Xcode Build Benchmark

Use this skill to produce a repeatable Xcode build baseline before anyone tries to optimize build times.

## Core Rules

- Measure before recommending changes.
- Capture clean and incremental builds separately.
- Keep the command, destination, configuration, scheme, and warm-up rules consistent across runs.
- Write a timestamped JSON artifact to `.build-benchmark/`.
- Do not change project files as part of benchmarking.

## Inputs To Collect

Confirm or infer:

- workspace or project path
- scheme
- configuration
- destination
- whether the user wants simulator or device numbers
- whether a custom `DerivedData` path is needed

If the project has both clean-build and incremental-build pain, benchmark both. That is the default.

## Worktree Considerations

When benchmarking inside a git worktree, SPM packages with `exclude:` paths that reference gitignored directories (e.g., `__Snapshots__`) will cause `xcodebuild -resolvePackageDependencies` to crash. Create those missing directories before running any builds.

## Default Workflow

1. Normalize the build command and note every flag that affects caching or module reuse.
2. Run one warm-up build if needed to validate that the command succeeds.
3. Run 3 clean builds.
4. If `COMPILATION_CACHE_ENABLE_CACHING = YES` is detected, run 3 cached clean builds. These measure clean build time with a warm compilation cache -- the realistic scenario for branch switching, pulling changes, or Clean Build Folder. The script handles this automatically by building once to warm the cache, then deleting DerivedData (but not the compilation cache) before each measured run. Pass `--no-cached-clean` to skip.
5. Run 3 zero-change builds (build immediately after a successful build with no edits). This measures the fixed overhead floor: dependency computation, project description transfer, build description creation, script phases, codesigning, and validation. A zero-change build that takes more than a few seconds indicates avoidable per-build overhead. Use the default `benchmark_builds.py` invocation (no `--touch-file` flag).
6. Optionally run 3 incremental builds with a file touch to measure a real edit-rebuild loop. Use `--touch-file path/to/SomeFile.swift` to touch a representative source file before each build.
7. Save the raw results and summary into `.build-benchmark/`.
8. Report medians and spread, not just the single fastest run.

## Preferred Command Path

Use the shared helper when possible:

```bash
python3 scripts/benchmark_builds.py \
  --workspace App.xcworkspace \
  --scheme MyApp \
  --configuration Debug \
  --destination "platform=iOS Simulator,name=iPhone 16" \
  --output-dir .build-benchmark
```

If you cannot use the helper script, run equivalent `xcodebuild` commands with `-showBuildTimingSummary` and preserve the raw output.

## Required Output

Return:

- clean build median, min, max
- cached clean build median, min, max (when COMPILATION_CACHE_ENABLE_CACHING is enabled)
- zero-change build median, min, max (fixed overhead floor)
- incremental build median, min, max (if `--touch-file` was used)
- biggest timing-summary categories
- environment details that could affect comparisons
- path to the saved artifact

If results are noisy, say so and recommend rerunning under calmer conditions.

## When To Stop

Stop after measurement if the user only asked for benchmarking. If they want optimization guidance, hand off the artifact to the relevant specialist by reading its SKILL.md and applying its workflow to the same project context:

- [`xcode-compilation-analyzer`](../xcode-compilation-analyzer/SKILL.md)
- [`xcode-project-analyzer`](../xcode-project-analyzer/SKILL.md)
- [`spm-build-analysis`](../spm-build-analysis/SKILL.md)
- [`xcode-build-orchestrator`](../xcode-build-orchestrator/SKILL.md) for full orchestration

## Additional Resources

- For the benchmark contract, see [references/benchmarking-workflow.md](references/benchmarking-workflow.md)
- For the shared artifact format, see [references/benchmark-artifacts.md](references/benchmark-artifacts.md)
- For the JSON schema, see [schemas/build-benchmark.schema.json](schemas/build-benchmark.schema.json)

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
3. At the end of the task, report the gap: "Skill `xcode-build-benchmark` needs an update: it says X, the docs now say Y (link)", and propose the change.
4. If the user agrees, the change is made in the source repository `C:\Users\USER\Desktop\Workspace\Mes Projets\Skills\xcode-build-benchmark`, then the skill is copied to wherever it is installed (`~/.claude/skills`, `~/.gemini/config/skills`, `~/.gemini/antigravity/skills`) and the repository is pushed to GitHub.

### Official links

- Xcode — Improving the speed of incremental builds: https://developer.apple.com/documentation/xcode/improving-the-speed-of-incremental-builds
- Xcode — Improving build efficiency with good coding practices: https://developer.apple.com/documentation/xcode/improving-build-efficiency-with-good-coding-practices
- Xcode — Building your project with explicit module dependencies: https://developer.apple.com/documentation/xcode/building-your-project-with-explicit-module-dependencies
- Xcode build settings reference: https://developer.apple.com/documentation/xcode/build-settings-reference
- Xcode release notes: https://developer.apple.com/documentation/xcode-release-notes
- `xcodebuild` flags: run `xcodebuild -help` (or `man xcodebuild`) with the installed Xcode.

developer.apple.com pages are rendered with JavaScript and can come back empty when fetched. Use the same path on sosumi.ai (for example `https://sosumi.ai/documentation/swiftui`), or the Sosumi MCP if it is available. For command-line flags, the installed tool's own help is authoritative: `xcodebuild -help`, `xcrun simctl help`, `swiftc -help`.

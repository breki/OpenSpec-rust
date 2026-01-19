# Comprehensive Implementation Plan

## 1. Executive Summary

**Goal:** Port OpenSpec CLI from Node.js/TypeScript to Rust

**Key Objectives:**
- Eliminate npm dependencies
- Create portable executables for multiple OS platforms
- Maintain functional and visual parity with original CLI

## 2. Original CLI Analysis

- **Repository**: [Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec)
- **Package**: `@fission-ai/openspec` v0.21.0
- **Language**: TypeScript (ES modules) with Node.js >=20.19.0
- **CLI Framework**: Commander.js v14.0.0
- **Dependencies**: commander, yaml, zod, chalk, ora, @inquirer/prompts, fast-glob, etc.

## 3. Supported Commands (10 Core Commands)

### Commands to Implement

1. **`init [directory]`** - Initialize OpenSpec in a project with AI tool selection
2. **`list`** - List changes or specs (--changes, --specs, --sort, --json)
3. **`validate [item-name]`** - Validate changes/specs (--all, --changes, --specs, --strict, --json)
4. **`show [item-name]`** - Show change or spec (--json, --type, various filters)
5. **`archive [change-name]`** - Archive completed change (-y, --skip-specs, --no-validate)
6. **`update [directory]`** - Update existing OpenSpec installation
7. **`view`** - Interactive dashboard of specs and changes
8. **`config <subcommand>`** - Manage global configuration (path, list, get, set, unset, reset, edit)
9. **`change <subcommand>`** - Manage changes (DEPRECATED noun-based: show, list, validate)
10. **`spec <subcommand>`** - Manage specifications (show, list, validate)

### Explicitly Excluded (Deferred to v2.0)

- **`completion`** - Shell completion commands
- **`feedback`** - User feedback collection
- **`experimental`** - Artifact workflow (13 sub-commands)
- **Telemetry** - PostHog analytics

## 4. Core Architecture

### Original Node.js Structure

```
src/
├── cli/index.ts              # Commander setup
├── commands/                 # Command implementations
├── core/                     # Core functionality
│   ├── parsers/             # Markdown/YAML parsers
│   ├── validation/          # Validators
│   ├── templates/           # File templates
│   └── configurators/       # AI tool configurators
└── utils/                    # Utilities
```

## 5. Implementation Strategy

### Phase 1: Project Setup & Infrastructure (Week 1 - 3 days)

**Objectives:**
- Initialize Cargo project
- Set up dependencies
- Configure CI/CD pipeline

**Dependencies in Cargo.toml:**

```toml
[dependencies]
clap = { version = "4.5", features = ["derive", "cargo"] }
serde = { version = "1.0", features = ["derive"] }
serde_yaml = "0.9"
serde_json = "1.0"
anyhow = "1.0"
thiserror = "1.0"
tokio = { version = "1.0", features = ["full"] }
colored = "2.1"
indicatif = "0.17"
dialoguer = "0.11"
glob = "0.3"
home = "0.5"
walkdir = "2.4"
regex = "1.10"
chrono = "0.4"

[dev-dependencies]
assert_cmd = "2.0"
predicates = "3.0"
tempfile = "3.8"
pretty_assertions = "1.4"
serial_test = "3.0"
```

**Tasks:**
- Set up test infrastructure
- Configure CI/CD (GitHub Actions)

### Phase 2: Core Architecture & Test Harness (Week 1-2 - 4 days)

**Module Structure:**

```
src/
├── main.rs
├── cli/
│   ├── mod.rs
│   └── commands/
│       ├── mod.rs
│       ├── init.rs
│       ├── list.rs
│       ├── validate.rs
│       ├── show.rs
│       ├── archive.rs
│       ├── update.rs
│       ├── view.rs
│       ├── config.rs
│       ├── change.rs
│       └── spec.rs
├── core/
│   ├── mod.rs
│   ├── config.rs
│   ├── schemas.rs
│   ├── parser.rs
│   ├── validator.rs
│   └── discovery.rs
├── utils/
│   ├── mod.rs
│   ├── fs.rs
│   ├── terminal.rs
│   └── prompts.rs
└── error.rs
```

**TDD Cycle 1: CLI Test Harness**

1. Write integration test for basic CLI invocation
2. Implement minimal CLI skeleton using clap
3. Verify tests pass

### Phase 3: File Discovery & List Command (Week 2 - 5 days)

**TDD Cycle 2: Discovery Utilities**
- Write tests for finding openspec directories
- Write tests for discovering spec/change files
- Implement file discovery logic
- Verify all tests pass

**TDD Cycle 3: List Command**
- Write integration tests for `list` command
- Implement list command with filtering options
- Add JSON output support
- Verify tests pass

### Phase 4: Markdown Parsing & Validation (Week 3-4 - 10 days)

**Critical: Why Markdown/YAML Parsing is Essential**

OpenSpec stores all data as structured Markdown files:

#### Spec Files (`openspec/specs/[capability]/spec.md`)
- Must parse `## Purpose` and `## Requirements` sections
- Extract requirements (### headings)
- Extract scenarios (#### headings under requirements)
- Validate structure
- Convert to JSON for `--json` output

#### Change Proposal Files (`openspec/changes/[name]/proposal.md`)
- Parse `## Why`, `## What Changes`, `## Impact` sections
- Extract deltas with syntax: `- **scope:** description`
- Detect operations (ADDED, MODIFIED, REMOVED)

#### Delta Spec Files
- Parse `## ADDED Requirements`, `## MODIFIED Requirements`, `## REMOVED Requirements`
- Merge during archive command

**Implementation:**
- Use `pulldown-cmark` or `comrak` crate for parsing
- Build AST for structured content
- Implement validation rules

### Phase 5: Init Command & Templates (Week 4-5 - 7 days)

**Tasks:**
- Interactive prompts with `dialoguer`
- AI tool selection (22+ tools: GitHub Copilot, Claude, Cursor, etc.)
- Template generation (AGENTS.md, project.md, tool configs)
- Directory structure creation

### Phase 6: Validate Command (Week 5-6 - 7 days)

**Tasks:**
- Full validation logic
- Concurrent validation support
- Zod-equivalent validation using Rust's type system
- Error reporting and formatting

### Phase 7: Archive & Remaining Commands (Week 6-7 - 7 days)

**Tasks:**
- Archive command (merge deltas)
- Show command (with filtering)
- Update command
- Integration testing

### Phase 8: View & Config Commands (Week 7 - 5 days)

**Tasks:**
- Dashboard view with `dialoguer`
- Global configuration management
- XDG Base Directory support

### Phase 9: Visual Polish & Error Messages (Week 7-8 - 5 days)

**Tasks:**
- Match terminal output exactly
- Use `colored` for colors, `indicatif` for spinners
- Error message format matching
- Help text formatting

### Phase 10: Cross-Platform Building (Week 8 - 4 days)

**Tasks:**
- GitHub Actions workflow for release builds
- Target platforms:
  - Linux (x86_64, musl)
  - macOS (Intel, Apple Silicon)
  - Windows (x86_64)
- Binary distribution setup

## 6. Testing Strategy

### Test Pyramid

- **Unit Tests (60%)** - Parser tests, validator tests, discovery tests
- **Integration Tests (35%)** - CLI command tests with temp fixtures
- **Manual Testing (5%)** - Cross-platform verification, performance

### Test Oracle Approach

Use original Node.js CLI as oracle:

```bash
# Generate expected outputs
cd ../OpenSpec
node bin/openspec.js list --json > expected-list.json

# Compare with Rust
cd ../openspec-rust
cargo run -- list --json > actual-list.json
diff expected-list.json actual-list.json
```

### Test Coverage Exclusions

- Skip telemetry tests
- Skip completion command tests
- Skip feedback command tests
- Skip experimental command tests

## 7. Success Criteria

### Functional Parity

- All core commands produce identical output to Node.js version
- Error messages match original format
- Exit codes match original behavior
- All original integration tests pass

### Performance

- Startup time < 50ms (vs ~200ms for Node.js)
- Memory usage < 20MB
- Binary size < 8MB compressed

### Distribution

- Single binary for each platform
- No external dependencies required

### Code Quality

- >80% test coverage
- `cargo clippy` passes
- `cargo fmt` applied
- Documentation for public APIs

## 8. Risk Mitigation

### Risk: Markdown Parsing Complexity
**Mitigation:** Use battle-tested `pulldown-cmark`; extensive golden file testing

### Risk: Async File I/O Differences
**Mitigation:** Use Tokio; test on all platforms

### Risk: Terminal Differences
**Mitigation:** Test on multiple terminals; provide `--no-color` flag

### Risk: Scope Creep
**Mitigation:** Implement commands by priority; defer experimental features

## 9. Timeline Summary

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| 1. Setup | 3 days | Project structure, CI/CD |
| 2. Architecture | 4 days | Core modules, test harness |
| 3. Discovery & List | 5 days | File discovery, list command |
| 4. Parsing & Validation | 10 days | Markdown parser, validators |
| 5. Init Command | 7 days | Interactive prompts, templates |
| 6. Validate Command | 7 days | Full validation logic |
| 7. Archive & Others | 7 days | Archive, show, update |
| 8. View & Config | 5 days | Dashboard, config management |
| 9. Polish | 5 days | Error handling, visual parity |
| 10. Distribution | 4 days | Cross-platform builds |
| **Total** | **~8 weeks** | Production-ready v1.0 |

## 10. Open Questions

### 1. Configuration Format
**Question:** Original uses JSON. Support TOML/YAML too?  
**Recommendation:** Match original (JSON) for compatibility

### 2. Global Config Location
**Question:** Original uses XDG Base Directory spec. Match exactly?  
**Recommendation:** Yes, use `directories` crate

### 3. Versioning
**Question:** Start at v0.21.0 (matching original) or v1.0.0?  
**Recommendation:** v0.21.0-rust.1, bump to v1.0.0 when feature-complete

### 4. Interactive Prompts
**Question:** Pixel-perfect UX or `dialoguer` defaults?  
**Recommendation:** Use `dialoguer` defaults, iterate based on feedback

## 11. Priority Order (MVP First)

### Phase 1 - MVP (Weeks 1-4)
1. init
2. list  
3. validate
4. show

### Phase 2 - Core Workflow (Weeks 5-6)
5. archive
6. update
7. view

### Phase 3 - Polish (Weeks 7-8)
8. config
9. Error handling refinement
10. Performance optimization
11. Cross-platform testing

### Phase 4 - Future (Post v1.0)
- completion commands
- feedback command
- experimental commands
- Telemetry (if ever needed)

## 12. Documentation Updates

### Add to README.md

```markdown
## Differences from Node.js CLI

The Rust v1.0 port intentionally excludes:

### Not Implemented (Deferred to Future Versions)
- **Telemetry/Analytics** - No usage tracking
- **Shell Completion Command** - No `openspec completion`
- **Feedback Command** - No `openspec feedback`
- **Experimental Commands** - No `openspec experimental`

### Core Commands (Fully Implemented)
- ✅ init, update - Project initialization
- ✅ list, show - Browse changes and specs
- ✅ validate - Validate structure
- ✅ archive - Archive completed changes
- ✅ view - Dashboard
- ✅ config - Configuration management
- ✅ change, spec - Management commands
```

## 13. Additional Resources

- [Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec) - Original repository
- [Clap Documentation](https://docs.rs/clap/latest/clap/)
- [Serde YAML](https://docs.rs/serde_yaml/latest/serde_yaml/)
- [assert_cmd](https://docs.rs/assert_cmd/latest/assert_cmd/)
- [pulldown-cmark](https://docs.rs/pulldown-cmark/latest/pulldown_cmark/)
- [colored](https://docs.rs/colored/latest/colored/)
- [indicatif](https://docs.rs/indicatif/latest/indicatif/)
- [dialoguer](https://docs.rs/dialoguer/latest/dialoguer/)

---

*This plan follows TDD principles: write tests first, implement minimal code to pass, refactor, repeat. Each phase builds on the previous, ensuring continuous validation against the original CLI behavior.*

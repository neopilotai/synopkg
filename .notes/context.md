# Synopkg Architecture

Rust CLI for synchronising npm dependency versions across JavaScript monorepos.

## Mental Model

```
CLI → analyse() → Context → inspect() → states → command runs → exit 0 or 1
```

Every dependency occurrence (e.g. React in package-a's dependencies) is an **Instance** tagged with a state describing whether it follows the rules.

## Pipeline

`src/synopkg.rs` owns dispatch. Every command follows the same four stages:

1. **analyse** — Read config, find packages, collect dependencies as instances
2. **fetch** — Query the npm registry (only for `update`)
3. **inspect** — `visit_packages()` or `visit_formatting()` assigns states
4. **run** — Command iterates states, performs side effects, returns exit code

```rust
pub async fn synopkg<D: DiskIo>(
  cli: Cli,
  io: &D,
  registry_client: &Arc<dyn RegistryClient>,
) -> Result<(Context, Option<RegistryUpdates>), SynopkgError>
```

## Critical Types

### Context (`src/context.rs`)

Owns all project data. Created once, passed through the pipeline, consumed by commands.

```rust
pub struct Context {
  pub config: Config,               // CLI args + parsed rcfile
  pub disk: Disk,                   // Files read from disk; mutated by fix/format
  pub instances: Vec<Instance>,     // Every dependency occurrence
  pub sources: Sources,             // Sole owner of every `Source` reference
  pub version_groups: Vec<VersionGroup>, // Rules/policies
}
```

`Context::create` merges catalog dependency types into the config, validates every selector, and allocates instances. It is the main validation boundary: errors are returned as `ContextError`, and independent failures are currently reported one at a time.

### Disk (`src/disk.rs`)

The on-disk state of the workspace. Mutations (fix, format) write through here.

```rust
pub struct Disk {
  pub cwd: PathBuf,
  pub lerna_json: Option<File<serde_json::Value>>,
  pub package_json_files: Vec<File<serde_json::Value>>, // Includes the root
  pub package_json_root_idx: Option<usize>,
  pub package_manager: Option<PackageManager>,
  pub pnpm_workspace: Option<YamlFile>,
}
```

### Sources (`src/sources.rs`)

Maps every file that can hold a dependency declaration to a `Source`.

```rust
pub struct Sources {
  pub all: Vec<Source>,
  pub user_source_indices: Vec<usize>,     // Slots matching a user `source` pattern
  pub pnpm_yaml_source_idx: Option<usize>, // Slot holding `Source::PnpmYaml`
}
```

### Source (`src/source.rs`)

```rust
pub enum Source {
  Package { file_idx: usize, name: String, formatting_mismatches: Vec<FormatMismatch> },
  PnpmYaml,
}
```

`file_idx` indexes into `Disk::package_json_files`. Catalog definitions point at the holding file (the pnpm yaml, or the Bun root package.json).

### Instance (`src/instance.rs`)

A single dependency occurrence.

```rust
pub struct Instance {
  pub descriptor: InstanceDescriptor,        // Immutable facts
  pub expected_specifier: RefCell<Option<Rc<Specifier>>>,
  pub id: InstanceId,                        // "{dep} in {location} of {package}"
  pub is_local_instance: bool,
}
```

`InstanceDescriptor` holds the dependency name, type, original specifier, and the `source_idx` of the file it came from.

### InstanceState (`src/instance/instance_state.rs`)

```text
InstanceState
├── Unknown              // Not yet inspected
├── Valid(...)           // Passes rules
├── Invalid
│   ├── Fixable(...)     // Can auto-fix
│   ├── Unfixable(...)   // Needs human decision
│   └── Conflict(...)    // Conflicting rules
└── Suspect(...)         // Misconfiguration
```

To see all variants:

```bash
ast-grep -p 'pub enum ValidInstance' src/
ast-grep -p 'pub enum FixableInstance' src/
ast-grep -p 'pub enum UnfixableInstance' src/
ast-grep -p 'pub enum SuspectInstance' src/
```

### VersionGroup (`src/version_group.rs`)

Versioning policies. `Banned`, `Catalog`, `CatalogDefs`, `Ignored`, `Pinned`, `PreferredSemver`, `SameMinor`, `SameRange`, `SemverRangeOnly`, `SnappedTo`. Each group claims instances through a `GroupSelector`; the first matching group wins.

To see variants:

```bash
ast-grep -p 'pub enum VersionGroup' src/
```

### Rcfile (`src/rcfile.rs`)

The parsed `.synopkgrc`: dependency groups, semver groups, update groups, version groups, formatting flags, and the resolved list of every dependency type (`all_dependency_types`).

## Finding Files

Use `find_path` or `ast-grep` to locate files. Key entry points:

| Purpose               | File                            |
| --------------------- | ------------------------------- |
| Entry & dispatch      | `src/main.rs`, `src/synopkg.rs` |
| Central data          | `src/context.rs`                |
| Files on disk         | `src/disk.rs`                   |
| File → source mapping | `src/sources.rs`, `src/source.rs` |
| Config file parsing   | `src/rcfile.rs`, `src/rcfile/`  |
| State assignment      | `src/visit_packages.rs`         |
| Validation logic      | `src/version_group/`            |
| Formatting            | `src/visit_formatting.rs`       |
| Commands              | `src/commands/`                 |
| Tests                 | `src/**/*_test.rs`, `tests/`     |

## Key Patterns

### Command Structure

```rust
pub fn run(ctx: Context) -> i32 {
  ctx.version_groups.iter().for_each(|group| {
    group.get_sorted_dependencies(&ctx.config.cli.sort).for_each(|dependency| {
      dependency.get_sorted_instances()
        .filter(|instance| instance.is_invalid())
        .for_each(|instance| { /* process */ });
    });
  });
  if has_issues { 1 } else { 0 }
}
```

### Visitor Choice

- **visit_packages** — Dependency version validation (lint, fix, update, list, json)
- **visit_formatting** — package.json structure (format)

### Ownership

- `Rc<Instance>` and `Rc<Specifier>` for single-threaded sharing
- `Arc` only for cross-thread use (RegistryClient)
- `Context` flows through phases; commands consume it
- Instances are addressed by `InstanceIdx` (an index into `Context::instances`), not by pointer

## Location String Format

Instance IDs follow this pattern:

```
{dependency} in {location} of {package}
```

Examples:

- `react in /dependencies of pkg-a`
- `lodash in /devDependencies of pkg-b`
- `pnpm in /packageManager of pkg-c`

# Monorepo Tasks <Badge type="warning" text="experimental" />

Mise supports monorepo-style task organization with Bazel/Buck2-inspired target path syntax. This feature allows you to manage tasks across multiple projects in a single repository while maintaining clear boundaries and enabling powerful pattern-based execution.

## Overview

When `experimental_monorepo_root` is enabled in your root `mise.toml`, mise will automatically discover tasks in subdirectories and prefix them with their relative path from the monorepo root. This creates a unified task namespace across your entire repository.

### Benefits

- **Unified task execution**: Run tasks across multiple projects from a single location
- **Clear task namespacing**: Tasks are automatically prefixed with their location
- **Pattern-based execution**: Use wildcards to run tasks across multiple projects
- **Tool inheritance**: Subdirectory tasks inherit tools from parent configs
- **Automatic trust propagation**: When the monorepo root is trusted, all descendant configs are automatically trusted

## Configuration

### Enabling Monorepo Mode

Add `experimental_monorepo_root = true` to your root `mise.toml`:

```toml
# /myproject/mise.toml
experimental_monorepo_root = true

[tools]
# Tools defined here are inherited by all subdirectories
node = "20"
```

::: warning
This feature requires `MISE_EXPERIMENTAL=1` environment variable.
:::

### Example Structure

```
myproject/
├── mise.toml (with experimental_monorepo_root = true)
├── projects/
│   ├── frontend/
│   │   └── mise.toml (with tasks: build, test)
│   └── backend/
│       └── mise.toml (with tasks: build, test)
```

With this structure, tasks will be automatically namespaced:

- `//projects/frontend:build`
- `//projects/frontend:test`
- `//projects/backend:build`
- `//projects/backend:test`

## Task Path Syntax

### Absolute Paths

Use `//` prefix to specify the absolute path from the monorepo root:

```bash
# Run build task in frontend project
mise run '//projects/frontend:build'

# Run test task in backend project
mise run '//projects/backend:test'
```

### Current Directory Tasks

Use `:` prefix to run tasks in the current directory:

```bash
cd projects/frontend
mise run ':build'  # Runs the build task in current directory
```

### Wildcard Patterns

Mise supports two types of wildcards for flexible task execution:

#### Path Wildcards (`...`)

Use ellipsis (`...`) to match any directory depth:

```bash
# Run 'test' task in ALL projects (any depth)
mise run '//...:test'

# Run 'build' in all subdirs under projects/
mise run '//projects/...:build'

# Match paths with wildcards in the middle
mise run '//projects/.../api:build'  # Matches projects/*/api and projects/*/*/api
```

#### Task Name Wildcards (`*`)

Use asterisk (`*`) to match task names:

```bash
# Run ALL tasks in frontend project
mise run '//projects/frontend:*'

# Run all tasks starting with 'test:'
mise run '//projects/frontend:test:*'

# Run 'lint' task across all projects
mise run '//...:lint'
```

### Combining Wildcards

You can combine both types of wildcards for powerful patterns:

```bash
# Run all tasks in all projects
mise run '//...:*'

# Run all test tasks in all projects
mise run '//...:test*'

# Run build tasks in all frontend-related projects
mise run '//.../frontend:build'
```

## Tool Inheritance

Subdirectory tasks automatically inherit tools from parent config files in the hierarchy. This allows you to:

1. Define common tools at the monorepo root
2. Override tools in specific subdirectories
3. Add additional tools in subdirectories

### Example

```toml
# /myproject/mise.toml
experimental_monorepo_root = true

[tools]
node = "20"      # Inherited by all subdirectories
python = "3.12"  # Inherited by all subdirectories
```

```toml
# /myproject/projects/frontend/mise.toml
[tools]
node = "18"  # Overrides the root's node 20

[tasks.build]
run = "npm run build"  # Uses node 18
```

```toml
# /myproject/projects/backend/mise.toml
# No tools section - inherits node 20 and python 3.12 from root

[tasks.build]
run = "npm run build"  # Uses node 20 from root
```

### Inheritance Rules

1. **Base toolset**: Tasks start with tools from all global config files (including parent configs in the hierarchy)
2. **Subdirectory override**: Tools defined in the subdirectory's config file are merged on top, allowing overrides
3. **Task-specific tools**: Tools defined in the task's `tools` property take highest precedence

## Performance Tuning

For large monorepos, you can control task discovery depth with the `task.monorepo_depth` setting (default: 5):

```toml
[settings]
task.monorepo_depth = 3  # Only search 3 levels deep
```

This limits how deep mise will search for task files:

- `1` = immediate children only (`monorepo_root/projects/`)
- `2` = grandchildren (`monorepo_root/projects/frontend/`)
- `5` = default (5 levels deep)

Reduce this value if you notice slow task discovery in very large monorepos, especially if your projects are concentrated at a specific depth level.

## Discovery Behavior

### Included Paths

Mise will scan for tasks in subdirectories containing any of these config files:

- `.mise.toml`
- `mise.toml`
- `.mise/config.toml`
- `.config/mise/config.toml`

### Excluded Paths

The following directories are automatically excluded from task discovery:

- Hidden directories (starting with `.`)
- `node_modules`
- `target`
- `dist`
- `build`

## Listing Tasks

### View All Monorepo Tasks

```bash
# List all tasks in current directory hierarchy
mise tasks

# List tasks from entire monorepo, including sibling directories
mise tasks --all
```

### View Specific Project Tasks

```bash
# List all tasks in frontend project
mise tasks '//projects/frontend:*'
```

## Best Practices

### 1. Define Shared Tools at Root

Place commonly-used tools in the root `mise.toml` to avoid repetition:

```toml
# /myproject/mise.toml
experimental_monorepo_root = true

[tools]
node = "20"
python = "3.12"
go = "1.21"
```

### 2. Override Only When Necessary

Only override tools in subdirectories when they genuinely need different versions:

```toml
# /myproject/legacy-app/mise.toml
[tools]
node = "14"  # Override only for legacy app
# python and go inherited from root
```

### 3. Use Descriptive Task Names

Prefix related tasks with common names to enable pattern matching:

```toml
[tasks.test]
run = "npm test"

[tasks."test:unit"]
run = "npm run test:unit"

[tasks."test:e2e"]
run = "npm run test:e2e"
```

Then run all test tasks: `mise run '//...:test*'`

### 4. Group Related Projects

Organize projects in subdirectories to enable targeted execution:

```
myproject/
├── services/
│   ├── api/
│   ├── worker/
│   └── scheduler/
└── apps/
    ├── web/
    └── mobile/
```

Then run tasks by group:

```bash
mise run '//services/...:build'  # Build all services
mise run '//apps/...:test'       # Test all apps
```

## Troubleshooting

### Tasks Not Appearing

If monorepo tasks aren't appearing:

1. Verify `experimental_monorepo_root = true` is in the root config
2. Ensure `MISE_EXPERIMENTAL=1` is set
3. Check that subdirectory configs are within the `task.monorepo_depth` limit
4. Verify subdirectory config filenames match the expected patterns

### Tools Not Inherited

If subdirectory tasks can't find tools:

1. Verify tools are defined in a parent config file
2. Check that the parent config is within the hierarchy
3. Ensure you're running the task from within the monorepo
4. Try running with `MISE_DEBUG=1` to see toolset loading

### Path Length Limitations on Windows

On Windows, the command line has path length limitations. If you hit these:

1. Set `MISE_INSTALLS_DIR` to a shorter location, e.g., `C:\.mise-installs`
2. Use `powershell.exe` or `pwsh.exe` instead of `cmd.exe`
3. Re-organize config files to specify only necessary tools

## Related

- [Task Configuration](/tasks/task-configuration) - All task configuration options
- [Running Tasks](/tasks/running-tasks) - How to execute tasks
- [Configuration](/configuration) - General mise configuration

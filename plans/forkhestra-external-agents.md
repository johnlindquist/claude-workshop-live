# Forkhestra External Claude Agents Support

Add support for orchestrating native Claude Code agents (`.claude/agents/*.md`) from external projects.

## Problem Statement

Forkhestra currently supports two types of agents:
1. Compiled binaries in PATH (e.g., `forge-task-manager`)
2. Direct paths to executables

However, most Claude Code projects define agents as markdown files in `.claude/agents/`, which are invoked with `claude --agent <name>`. There's no ergonomic way to orchestrate these native agents from other projects.

Currently you'd have to manually run:
```bash
cd /path/to/other-project && claude --agent reviewer
```

## Design

### Path-Based Type Inference

Instead of requiring explicit type declarations, infer the agent type from its path pattern:

```
/path/to/project/.claude/agents/reviewer
         ↓ parse ↓
project: /path/to/project
agent:   reviewer
         ↓ execute ↓
claude --agent reviewer  (with cwd: /path/to/project)
```

**Resolution logic:**
1. If path matches `*/.claude/agents/<name>` → Claude agent mode
2. Otherwise → current behavior (binary/PATH lookup)

### Usage Examples

#### DSL Mode

```bash
# External Claude agent with loop
forkhestra "/home/user/project-a/.claude/agents/reviewer:3"

# Mixed chain: external Claude agent + forge binary
forkhestra "/home/user/project-a/.claude/agents/reviewer:3 -> forge-task-manager"

# Relative paths (relative to --cwd)
forkhestra "../other-project/.claude/agents/analyzer:5"

# Multiple external projects
forkhestra "/project-a/.claude/agents/planner -> /project-b/.claude/agents/builder:10"
```

#### Config Mode

```json
{
  "agents": {
    "external-reviewer": {
      "path": "/home/user/project-a/.claude/agents/reviewer",
      "defaultPrompt": "Review the recent changes"
    },
    "local-analyzer": {
      "path": "../other-project/.claude/agents/analyzer"
    },
    "forge-task-manager": {}
  },
  "chains": {
    "cross-project-workflow": {
      "description": "Use agents from multiple projects",
      "steps": [
        { "agent": "forge-task-manager", "iterations": 2 },
        { "agent": "external-reviewer", "iterations": 3 },
        { "agent": "local-analyzer" }
      ]
    }
  }
}
```

### Execution Modes

| Path Pattern | Execution |
|--------------|-----------|
| `forge-task-manager` | `spawn(["forge-task-manager", ...args])` via PATH |
| `/usr/bin/my-tool` | `spawn(["/usr/bin/my-tool", ...args])` |
| `/project/.claude/agents/name` | `spawn(["claude", "--agent", "name", ...args], { cwd: "/project" })` |
| `../proj/.claude/agents/name` | Resolve relative path, then Claude agent mode |

### Completion Marker Handling

External Claude agents participating in loops must also output `FORKHESTRA_COMPLETE` when done. This can be achieved by:

1. Adding the marker output to the agent's markdown instructions
2. Using a wrapper prompt that includes the marker contract

Example agent instruction (in `.claude/agents/reviewer.md`):
```markdown
When you have completed all reviews and there is no more work to do, output:
FORKHESTRA_COMPLETE
```

## Implementation Plan

### Task 1: Add Path Pattern Detection Utility

**File:** `lib/forkhestra/agent-resolver.ts` (new file)

```typescript
interface ResolvedAgent {
  type: "binary" | "claude-agent";
  command: string[];
  cwd: string;
}

/**
 * Detect if path is a Claude agent and resolve execution details
 */
export function resolveAgent(
  agentPath: string,
  baseCwd: string
): ResolvedAgent;

/**
 * Check if path matches .claude/agents pattern
 */
export function isClaudeAgentPath(path: string): boolean;

/**
 * Extract project root and agent name from Claude agent path
 */
export function parseClaudeAgentPath(path: string): {
  projectRoot: string;
  agentName: string;
} | null;
```

**Acceptance Criteria:**
- [ ] Detects `.claude/agents/<name>` pattern (with or without .md extension)
- [ ] Handles absolute paths
- [ ] Handles relative paths (resolved against baseCwd)
- [ ] Extracts project root correctly
- [ ] Extracts agent name correctly
- [ ] Returns binary type for non-matching paths
- [ ] Unit tests for various path patterns

### Task 2: Update Runner to Use Agent Resolver

**File:** `lib/forkhestra/runner.ts`

Update `runOnce()` and `runOnceWithMarkerDetection()` to use the resolver:

```typescript
import { resolveAgent } from "./agent-resolver";

function runOnce(options: RunOptions): Promise<RunResult> {
  const resolved = resolveAgent(options.agent, options.cwd || process.cwd());

  const cmdArgs = [...resolved.command.slice(1), ...args];
  if (prompt) cmdArgs.push(prompt);

  const proc = spawn([resolved.command[0], ...cmdArgs], {
    cwd: resolved.cwd,
    // ...
  });
}
```

**Acceptance Criteria:**
- [ ] Binary agents work as before (no regression)
- [ ] Claude agent paths spawn `claude --agent <name>`
- [ ] Claude agent cwd is set to project root
- [ ] Additional args are passed correctly to claude
- [ ] Prompt is passed correctly to claude
- [ ] Works in both single-run and loop modes
- [ ] Integration tests with mock Claude agent

### Task 3: Update Config Schema for Agent Path

**File:** `lib/forkhestra/config.ts`

Add optional `path` field to `AgentConfig`:

```typescript
interface AgentConfig {
  path?: string;  // NEW: path to agent (binary or .claude/agents/name)
  defaultPrompt?: string;
  defaultPromptFile?: string;
}
```

Update agent lookup in chain executor to check for path:

```typescript
function getAgentPath(agentName: string, config?: ForkhestraConfig): string {
  const agentConfig = config?.agents?.[agentName];
  return agentConfig?.path || agentName;
}
```

**Acceptance Criteria:**
- [ ] `path` field added to AgentConfig type
- [ ] Config validation accepts path field
- [ ] Agent lookup uses path when defined
- [ ] Falls back to agent name when no path
- [ ] Relative paths in config resolved against cwd
- [ ] Unit tests for config with paths

### Task 4: Update Chain Executor

**File:** `lib/forkhestra/chain.ts`

Pass resolved agent path to runner:

```typescript
for (const step of steps) {
  const agentPath = getAgentPath(step.agent, config);

  const result = await run({
    agent: agentPath,  // May be name, binary path, or .claude/agents path
    // ...
  });
}
```

**Acceptance Criteria:**
- [ ] Chain executor resolves agent paths from config
- [ ] Mixed chains (binaries + Claude agents) work
- [ ] Prompts passed correctly to Claude agents
- [ ] Verbose output shows resolved paths
- [ ] Dry-run shows what would be executed

### Task 5: Documentation

**Files:**
- `CLAUDE.md` - Update forkhestra section
- `docs/FORKHESTRA.md` - Add external agents section

Document:
- Path-based type inference
- How to use external Claude agents
- Completion marker contract for Claude agents
- Example configurations

**Acceptance Criteria:**
- [ ] CLAUDE.md updated with external agent examples
- [ ] docs/FORKHESTRA.md has dedicated section
- [ ] Example in forge/chains.json showing external agent

## Testing Strategy

### Unit Tests
- Path pattern detection (various formats)
- Agent resolution (binary vs Claude agent)
- Config parsing with path field
- Relative path resolution

### Integration Tests
- Run external Claude agent via forkhestra
- Mixed chain with binary and Claude agent
- Loop with Claude agent and marker detection

### Manual Tests
- Create test project with `.claude/agents/test-agent.md`
- Run via forkhestra DSL
- Run via config with path
- Verify cwd is correct for external project

## Edge Cases

1. **Path without .md extension**: `/project/.claude/agents/reviewer` should work (Claude doesn't require .md in --agent flag)

2. **Nested .claude directories**: Use the last `.claude/agents/` match
   - `/a/.claude/agents/b/.claude/agents/name` → agent is `name`, project is `/a/.claude/agents/b`

3. **Agent name with slashes**: Not supported, agent name must be simple identifier

4. **Missing project directory**: Error with clear message about project not found

5. **Missing agent file**: Let Claude error naturally (it will say agent not found)

## Future Considerations

- **Agent discovery**: List available agents from external projects
- **Project aliases**: Define project shortcuts in config
  ```json
  {
    "projects": {
      "backend": "/home/user/backend-api",
      "frontend": "/home/user/web-app"
    },
    "agents": {
      "api-reviewer": {
        "project": "backend",
        "agent": "reviewer"
      }
    }
  }
  ```
- **Remote agents**: Support git URLs for agent definitions

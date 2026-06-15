# Commander.js

**What it is:** The de-facto Node.js library for building command-line interfaces — it parses `argv` into commands, options, and arguments, and generates help.

**Why people use it:** Declarative command/option definitions, automatic `--help` and version output, subcommands, and typed parsing — instead of hand-rolling `process.argv`.

**Typically used for:** Dev tooling and CLIs — build scripts, code generators, and agent/LLM tools (`my-agent run`, `my-agent eval --model claude-opus-4-8`).

> TypeScript throughout. Install: `npm i commander`. Import: `import { Command, Option } from 'commander';`

---

## Mental model

```
program            the root Command
 ├─ option/argument     flags & positional args attached to a command
 ├─ action(fn)          the handler that runs for that command
 └─ command(...)        a subcommand (its own options + action)
program.parse()    reads process.argv and dispatches
```

Define commands declaratively, attach an `action`, call `parse()` once at the end.

---

## Minimal CLI

```ts
import { Command } from 'commander';

const program = new Command();

program
  .name('agent')
  .description('Run and evaluate LLM agents')
  .version('1.0.0');               // adds -V / --version

program
  .command('run')
  .description('Run the agent on a prompt')
  .argument('<prompt>', 'the user prompt')          // required positional
  .option('-m, --model <name>', 'model id', 'claude-opus-4-8')  // with default
  .option('-v, --verbose', 'log intermediate steps')            // boolean flag
  .action(async (prompt: string, opts: { model: string; verbose?: boolean }) => {
    if (opts.verbose) console.error(`[model=${opts.model}]`);
    await runAgent(prompt, opts.model);
  });

await program.parseAsync();        // parseAsync for async actions
```

```bash
agent run "summarize this" --model claude-sonnet-4-6 -v
agent run --help                   # auto-generated
```

---

## Options

```ts
program
  .option('-d, --debug', 'enable debug')                  // boolean
  .option('-p, --port <number>', 'port', '3000')          // value with default
  .option('-t, --tag <name...>', 'repeatable, collects')  // variadic → string[]
  .requiredOption('-k, --api-key <key>', 'API key')       // errors if missing
  .option('--no-color', 'disable color');                 // negatable → opts.color=false
```

Access via `cmd.opts()` (camelCased: `--api-key` → `opts.apiKey`).

```ts
.action((opts) => {
  opts.port;     // '3000' (string — parse it, see below)
  opts.apiKey;
  opts.color;    // false if --no-color passed, else true
});
```

### Typed / parsed option values

Pass a coercion function as the 3rd arg — turn strings into numbers, enums, etc. at parse time.

```ts
import { Option, InvalidArgumentError } from 'commander';

function parsePositiveInt(value: string): number {
  const n = Number(value);
  if (!Number.isInteger(n) || n <= 0) throw new InvalidArgumentError('Must be a positive integer.');
  return n;
}

program.option('-c, --concurrency <n>', 'parallel calls', parsePositiveInt, 4);

// Constrain to a fixed set with the Option builder:
program.addOption(
  new Option('-m, --model <name>', 'model').choices(['claude-opus-4-8', 'claude-sonnet-4-6']).default('claude-opus-4-8'),
);
// Pull a default from the environment:
program.addOption(new Option('--api-key <key>').env('ANTHROPIC_API_KEY').makeOptionMandatory());
```

---

## Arguments (positional)

```ts
program
  .command('eval')
  .argument('<suite>', 'eval suite name')        // required
  .argument('[runs]', 'number of runs', '1')     // optional with default
  .argument('<files...>', 'input files')         // variadic → string[] (must be last)
  .action((suite: string, runs: string, files: string[]) => { … });
```

---

## Subcommands

Two styles. Inline for small CLIs; separate files for larger ones.

```ts
// Inline — chain .command() off the root
program.command('login').action(() => { … });
program.command('logout').action(() => { … });

// Nested subcommands (git-style: agent tools list)
const tools = program.command('tools').description('manage agent tools');
tools.command('list').action(() => { … });
tools.command('add <name>').action((name) => { … });
```

```ts
// Separate files — build each Command and attach with addCommand()
// commands/run.ts
import { Command } from 'commander';
export const runCommand = new Command('run')
  .argument('<prompt>')
  .action(async (prompt) => { … });

// index.ts
import { runCommand } from './commands/run';
program.addCommand(runCommand);
```

---

## Errors & exit handling

Throw inside an action; Commander surfaces it. For controlled exits use the helpers — they print and set the exit code.

```ts
.action((opts) => {
  if (!fs.existsSync(opts.config)) {
    program.error(`Config not found: ${opts.config}`, { exitCode: 2 });  // print + exit(2)
  }
});

// Validate combinations after parse:
const opts = program.opts();
if (opts.json && opts.verbose) program.error('--json and --verbose are mutually exclusive');
```

Async actions: always `await program.parseAsync()` and let rejections bubble, or wrap:

```ts
program.parseAsync().catch((err) => {
  console.error(err instanceof Error ? err.message : err);
  process.exit(1);
});
```

---

## Hooks & shared setup

Run logic before/after any action — handy for global flags (set up logging, load config, init a client).

```ts
program
  .option('--verbose', 'verbose logging')
  .hook('preAction', (thisCommand, actionCommand) => {
    if (thisCommand.opts().verbose) setLogLevel('debug');
  });
```

Read parent options from a subcommand:

```ts
tools.command('list').action(function () {
  const globals = this.optsWithGlobals();   // merges parent + this command's opts
});
```

---

## Help customization

```ts
program
  .showHelpAfterError('(add --help for usage)')   // hint on parse error
  .configureHelp({ sortSubcommands: true });

program.command('run')
  .addHelpText('after', `
Examples:
  $ agent run "hello" --model claude-sonnet-4-6
  $ agent run "hello" -v`);
```

---

## Project recipe — a typed agent CLI

```ts
#!/usr/bin/env node
import { Command, Option } from 'commander';

const program = new Command();
program.name('agent').description('LLM agent toolkit').version(process.env.npm_package_version ?? '0.0.0');

program
  .command('run')
  .description('run the agent once')
  .argument('<prompt>', 'user prompt')
  .addOption(new Option('-m, --model <id>').choices(['claude-opus-4-8', 'claude-sonnet-4-6']).default('claude-opus-4-8'))
  .addOption(new Option('--api-key <key>').env('ANTHROPIC_API_KEY').makeOptionMandatory())
  .option('--json', 'output JSON')
  .action(async (prompt: string, opts: { model: string; apiKey: string; json?: boolean }) => {
    const result = await runAgent({ prompt, model: opts.model, apiKey: opts.apiKey });
    console.log(opts.json ? JSON.stringify(result) : result.text);
  });

program.parseAsync().catch((e) => { console.error(e.message); process.exit(1); });
```

```jsonc
// package.json — expose the binary
{ "bin": { "agent": "./dist/index.js" }, "type": "module" }
```

```bash
npm link        # symlink the bin locally for dev
agent run "hi"  # now on PATH
```

---

## Tips

- **`parseAsync`, not `parse`**, whenever any action is `async` — `parse` won't await.
- **Options camelCase**: `--dry-run` → `opts.dryRun`; `--no-x` makes `opts.x` default to `true`.
- **Coerce at parse time** (3rd-arg function or `Option`) so actions receive `number`/enum, not raw strings.
- **`requiredOption` / `makeOptionMandatory`** beats checking-then-erroring by hand.
- **One Command per file** for non-trivial CLIs; wire them with `addCommand()`.
- **Add `#!/usr/bin/env node`** to the entry file and set `bin` in `package.json` to ship it.

> Related: [nodejs.md](nodejs.md), [esbuild.md](esbuild.md) (bundle a CLI to a single file).

---
name: cli-tool-scripts
description: "Use this skill whenever writing a CLI tool, automation script, ops script, migration script, or any Node.js script that performs actions on external systems (databases, APIs, file systems, cloud services, CI/CD). Trigger on phrases like 'write a script to…', 'automate…', 'migration script', 'CLI tool', 'ops tool', 'batch process', 'cleanup script', 'deploy script', or any request for a Node.js script that does more than trivial file I/O. Also trigger when the user asks to add interactivity, logging, dry-run, or environment handling to an existing script."
---

# CLI Tool Scripts — Node.js

When writing any CLI tool or automation script in Node.js, follow every principle below. These aren't optional extras — they're the baseline for a production-grade tool that's safe to run, easy to debug, and pleasant to use.

## Core Architecture

Structure every script with these layers, in this order:

```
1. Config & Environment    → Load .env, parse flags, resolve environment
2. Pre-flight checks       → Validate inputs, check connectivity, confirm targets
3. Interactive confirmation → Show plan, ask user to approve/edit/reject
4. Execution               → Run actions with progress, retries, rollback tracking
5. Reporting               → Write JSON log + HTML report, print summary
```

---

## 1. Environment & Flags

Every script must support multiple environments and never hardcode secrets.

### Flags with `commander` or `yargs`

```js
import { program } from 'commander';

program
  .option('-e, --env <environment>', 'Target environment', 'staging')  // default to staging, NEVER prod
  .option('--dry-run', 'Show what would happen without executing', false)
  .option('--yes', 'Skip interactive confirmations (accept all)', false)
  .option('--no-interactive', 'Reject all confirmations automatically (useful for CI dry-run audits)', false)
  .option('--log-dir <path>', 'Directory for log/report output', './logs')
  .option('--verbose', 'Verbose output', false)
  .parse();
```

**Rules:**
- Default environment is always `staging`, never `production`. Require an explicit `--env production` flag.
- Add a visible warning banner when `--env production` is used (colored red).
- Load environment variables from `.env`, `.env.staging`, `.env.production` etc. using `dotenv`. Never hardcode API keys, tokens, database URIs, or any secret.
- If a required env variable is missing, fail immediately with a clear message saying which variable is missing and which `.env` file was checked.

### Interactive by default

When flags are missing or ambiguous, **prompt the user** instead of failing or assuming. Use `inquirer` or `@inquirer/prompts`:

```js
import { select, confirm, input } from '@inquirer/prompts';

// If --env not provided via flag, ask
const env = opts.env || await select({
  message: 'Which environment?',
  choices: [
    { name: '🟡 Staging', value: 'staging' },
    { name: '🔴 Production', value: 'production' },
  ],
});
```

---

## 2. Interactive Approval Flow

This is the most important UX feature. Before any destructive or important action, present a clear plan and let the user decide.

### Before execution: show the plan

```js
function presentPlan(actions) {
  console.log(chalk.bold('\n📋 Execution Plan:\n'));
  actions.forEach((action, i) => {
    const icon = action.destructive ? '🔴' : '🟢';
    console.log(`  ${icon} ${i + 1}. ${action.description}`);
    if (action.details) console.log(chalk.dim(`      ${action.details}`));
  });
  console.log();
}
```

### Per-action and batch approval

Offer these choices for each important/destructive action:

- **Yes** — execute this action
- **No / Skip** — skip this action, continue to next
- **Edit** — let the user modify parameters before executing (where applicable)
- **Yes to all** — stop asking, execute all remaining
- **No to all / Abort** — stop asking, skip all remaining

```js
async function confirmAction(action, opts) {
  if (opts.yes) return 'yes';            // --yes flag: accept all
  if (opts.noInteractive) return 'no';   // --no-interactive: reject all
  if (opts.dryRun) {
    logDryRun(action);
    return 'skip';
  }

  const answer = await select({
    message: `Execute: ${action.description}?`,
    choices: [
      { name: '✅ Yes', value: 'yes' },
      { name: '⏭️  Skip', value: 'skip' },
      { name: '✏️  Edit', value: 'edit' },
      { name: '✅ Yes to all remaining', value: 'yes_all' },
      { name: '🛑 Abort (skip all remaining)', value: 'abort' },
    ],
  });

  return answer;
}
```

---

## 3. Dry-Run Mode

When `--dry-run` is active, the script must:
- Walk through every step of the execution plan
- Log exactly what *would* happen (including API calls, file writes, DB queries)
- Never mutate any state — no writes, no API calls, no side effects
- Clearly prefix every output line with `[DRY RUN]`
- Still produce the full JSON + HTML report (marked as dry-run)

```js
function execute(action, opts) {
  if (opts.dryRun) {
    log.info(`[DRY RUN] Would execute: ${action.description}`);
    return { status: 'dry-run', action: action.id, details: action.params };
  }
  // ... actual execution
}
```

---

## 4. Idempotency

Scripts must be safe to re-run. Before performing any action:
- **Check current state first.** If the desired state already exists, skip with a log entry.
- **Use unique identifiers** (not sequential counters) for created resources.
- **Upsert over insert** when possible.

```js
// Example: idempotent user creation
const existing = await db.users.findOne({ email });
if (existing) {
  log.info(`⏭️  User ${email} already exists, skipping`);
  return { status: 'skipped', reason: 'already_exists' };
}
```

---

## 5. Rollback / Undo Support

Track every mutation so it can be reversed.

```js
const rollbackStack = [];

async function executeWithRollback(action) {
  const result = await action.execute();
  if (action.rollback) {
    rollbackStack.push({
      description: `Undo: ${action.description}`,
      execute: action.rollback,
      originalResult: result,
    });
  }
  return result;
}

async function rollbackAll() {
  console.log(chalk.yellow('\n⏪ Rolling back all changes...\n'));
  // Reverse order — undo last action first
  for (const step of rollbackStack.reverse()) {
    try {
      await step.execute();
      log.info(`  ✅ ${step.description}`);
    } catch (err) {
      log.error(`  ❌ Rollback failed: ${step.description} — ${err.message}`);
    }
  }
}
```

On any critical failure, offer the user the choice to rollback:

```js
catch (err) {
  log.error(`Fatal error: ${err.message}`);
  if (rollbackStack.length > 0) {
    const shouldRollback = await confirm({
      message: `${rollbackStack.length} actions were completed. Roll back?`,
    });
    if (shouldRollback) await rollbackAll();
  }
}
```

---

## 6. Retry & Timeout Logic

Wrap all network/API calls with retry and timeout:

```js
async function withRetry(fn, { retries = 3, delayMs = 1000, backoff = 2, timeoutMs = 30000 } = {}) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await Promise.race([
        fn(),
        new Promise((_, reject) =>
          setTimeout(() => reject(new Error('Timeout')), timeoutMs)
        ),
      ]);
    } catch (err) {
      if (attempt === retries) throw err;
      const wait = delayMs * Math.pow(backoff, attempt - 1);
      log.warn(`  ⚠️  Attempt ${attempt}/${retries} failed: ${err.message}. Retrying in ${wait}ms...`);
      await new Promise(r => setTimeout(r, wait));
    }
  }
}
```

---

## 7. Progress Indicators

Use `ora` for spinners on single operations and `cli-progress` for batch operations:

```js
import ora from 'ora';

const spinner = ora('Fetching users from API...').start();
try {
  const users = await fetchUsers();
  spinner.succeed(`Fetched ${users.length} users`);
} catch (err) {
  spinner.fail(`Failed to fetch users: ${err.message}`);
}
```

For batch processing, show a progress bar with ETA:

```js
import { SingleBar, Presets } from 'cli-progress';

const bar = new SingleBar({
  format: '  {bar} {percentage}% | {value}/{total} | ETA: {eta}s | {task}',
}, Presets.shades_classic);

bar.start(items.length, 0, { task: 'Processing...' });
for (const item of items) {
  await processItem(item);
  bar.increment(1, { task: item.name });
}
bar.stop();
```

---

## 8. Color-Coded Output

Use `chalk` for consistent color coding throughout:

| Color | Meaning |
|-------|---------|
| `chalk.green` | Success, completed |
| `chalk.yellow` | Warning, skipped |
| `chalk.red` | Error, failure, production warning |
| `chalk.cyan` | Info, progress |
| `chalk.dim` | Secondary details, metadata |
| `chalk.bold` | Section headers, important values |
| `chalk.bgRed.white.bold` | Production environment banner |

Production warning banner:

```js
if (env === 'production') {
  console.log(chalk.bgRed.white.bold('\n  ⚠️  PRODUCTION ENVIRONMENT  ⚠️  \n'));
  const confirmed = await confirm({ message: 'You are targeting PRODUCTION. Continue?' });
  if (!confirmed) process.exit(0);
}
```

---

## 9. Error Handling & Graceful Shutdown

Catch everything. Never let the script crash without a useful message:

```js
process.on('SIGINT', async () => {
  console.log(chalk.yellow('\n\nInterrupted by user.'));
  await writeReport(results, { interrupted: true });
  if (rollbackStack.length > 0) {
    const shouldRollback = await confirm({ message: 'Roll back completed actions?' });
    if (shouldRollback) await rollbackAll();
  }
  process.exit(1);
});

process.on('unhandledRejection', (err) => {
  log.error(`Unhandled rejection: ${err.message}`);
  process.exit(1);
});
```

Wrap the main function:

```js
async function main() {
  const results = { actions: [], errors: [], skipped: [], startedAt: new Date().toISOString() };
  try {
    // ... execution logic, push to results.actions / results.errors / results.skipped
  } catch (err) {
    results.fatalError = { message: err.message, stack: err.stack };
    log.error(chalk.red(`\n❌ Fatal: ${err.message}\n`));
  } finally {
    results.completedAt = new Date().toISOString();
    await writeReport(results);
    printSummary(results);
  }
}
```

---

## 10. Logging & Reports

Every run must produce two report files. This is non-negotiable — even failed or dry-runs get reports.

### JSON log (machine-readable)

```js
// logs/2025-03-31T14-30-00_staging_migrate-users.json
{
  "script": "migrate-users",
  "version": "1.0.0",
  "environment": "staging",
  "dryRun": false,
  "startedAt": "2025-03-31T14:30:00.000Z",
  "completedAt": "2025-03-31T14:32:15.000Z",
  "durationMs": 135000,
  "summary": {
    "total": 42,
    "succeeded": 38,
    "failed": 2,
    "skipped": 2
  },
  "actions": [
    {
      "id": "action_001",
      "description": "Create user john@example.com",
      "status": "succeeded",  // "succeeded" | "failed" | "skipped" | "dry-run" | "rolled-back"
      "startedAt": "...",
      "durationMs": 230,
      "details": {},
      "error": null
    }
  ],
  "errors": [],
  "rollbacks": [],
  "flags": { "env": "staging", "dryRun": false, "verbose": false }
}
```

### HTML report (human-readable)

Generate a self-contained HTML file (inline CSS, no external dependencies) with:

- **Header**: script name, environment (color-coded), timestamp, duration
- **Summary cards**: total / succeeded / failed / skipped counts with colors
- **Filterable action table**: status, description, duration, error message. Allow filtering by status.
- **Error details section**: expandable stack traces for failures
- **Rollback section**: what was rolled back and whether it succeeded
- **Dry-run badge**: prominent visual indicator if this was a dry run

Use this helper to generate both:

```js
async function writeReport(results, opts = {}) {
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
  const baseName = `${timestamp}_${results.environment}_${results.script}`;
  const logDir = opts.logDir || './logs';

  await fs.mkdir(logDir, { recursive: true });

  // JSON
  const jsonPath = path.join(logDir, `${baseName}.json`);
  await fs.writeFile(jsonPath, JSON.stringify(results, null, 2));
  log.info(chalk.dim(`📄 JSON log: ${jsonPath}`));

  // HTML
  const htmlPath = path.join(logDir, `${baseName}.html`);
  await fs.writeFile(htmlPath, generateHtmlReport(results));
  log.info(chalk.dim(`🌐 HTML report: ${htmlPath}`));
}
```

### Terminal summary

Always print a summary at the end, even on failure:

```js
function printSummary(results) {
  const { summary } = results;
  console.log(chalk.bold('\n' + '═'.repeat(50)));
  console.log(chalk.bold('  Summary'));
  console.log('═'.repeat(50));
  console.log(`  ${chalk.cyan('Total:')}     ${summary.total}`);
  console.log(`  ${chalk.green('Succeeded:')} ${summary.succeeded}`);
  console.log(`  ${chalk.red('Failed:')}    ${summary.failed}`);
  console.log(`  ${chalk.yellow('Skipped:')}   ${summary.skipped}`);
  console.log(`  ${chalk.dim('Duration:')}  ${(results.durationMs / 1000).toFixed(1)}s`);
  console.log('═'.repeat(50) + '\n');
}
```

---

## Package.json Dependencies

Every CLI tool script should include these in its `package.json`:

```json
{
  "type": "module",
  "dependencies": {
    "chalk": "^5.3.0",
    "commander": "^12.0.0",
    "@inquirer/prompts": "^7.0.0",
    "dotenv": "^16.4.0",
    "ora": "^8.0.0",
    "cli-progress": "^3.12.0"
  }
}
```

---

## Checklist — Before Delivering a Script

Verify every point before considering the script done:

- [ ] `--env` flag with staging default, production requires explicit flag + red warning
- [ ] `--dry-run` flag that logs everything but mutates nothing
- [ ] `--yes` and `--no-interactive` flags for CI/automation
- [ ] `.env` loading, no hardcoded secrets, clear error on missing vars
- [ ] Interactive prompts when flags are ambiguous or missing
- [ ] Per-action approval with yes/skip/edit/yes-all/abort choices
- [ ] Idempotency checks before every mutation
- [ ] Rollback stack tracking every mutation, rollback offered on failure
- [ ] Retry + timeout on all network calls
- [ ] Spinners for single ops, progress bars for batch ops
- [ ] Consistent color coding (green/yellow/red/cyan/dim)
- [ ] `SIGINT` handler with graceful shutdown and report writing
- [ ] JSON log written to `./logs/`
- [ ] HTML report written to `./logs/` (self-contained, filterable)
- [ ] Terminal summary printed at the end (total/succeeded/failed/skipped/duration)

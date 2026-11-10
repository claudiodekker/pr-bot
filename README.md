# pr-bot

Run commands on a pull request by commenting an emoji. Comment 🎨 and pr-bot formats the pull request's branch with Pint and Prettier, commits what changed and pushes it back. When a command fails, pr-bot replies with its output.

pr-bot is a reusable GitHub Actions workflow. It runs in your repo with the workflow's own token, so there's no app to install and no secret to store.

## Setup

Add this workflow to the repo as `.github/workflows/pr-commands.yml`:

```yaml
name: PR commands

on:
  issue_comment:
    types: [created]

permissions:
  contents: write
  pull-requests: write

jobs:
  commands:
    uses: claudiodekker/pr-bot/.github/workflows/commands.yml@v1
```

Then comment exactly 🎨 on a pull request. pr-bot reacts with 👀, runs the formatters, pushes a commit named "Apply 🎨 fixes" when anything changed, and finishes with 👍 or 👎.

pr-bot only replies when a command fails. The reply mentions you, links to the run and shows the command's output in a collapsible block, cut to the last 100 lines when it's longer.

## The built-in 🎨

🎨 runs the formatters the repo uses:

- Pint, when `composer.json` requires `laravel/pint`. pr-bot runs `composer install` first.
- Prettier, when `package.json` lists `prettier`. pr-bot installs with pnpm when the repo has `pnpm-lock.yaml`, and with npm otherwise.

Installs run without package scripts, because pr-bot runs the pull request's code with a token that can push. When the repo uses neither formatter, 🎨 fails and says so.

## Your own commands

Pass your commands as `commands`, one per line, as `trigger: script`:

```yaml
jobs:
  commands:
    uses: claudiodekker/pr-bot/.github/workflows/commands.yml@v1
    with:
      commands: |
        🎨: format && npx eslint --fix .
        ♻️: composer install --no-interaction && vendor/bin/rector process
        🚀: ./.github/bot/deploy.sh "$@"
```

- A trigger is one word, such as an emoji or `/fix`. A comment runs it when its first word is the trigger, so both `🚀` and `🚀 staging` run 🚀.
- The words after the trigger are the command line's `$1`, `$2` and so on, and `"$@"` passes them on to a script, as the 🚀 line shows. The whole text after the trigger is also in `$ARGS`. A comment like `🎨 looks good` runs 🎨 too, with `looks` and `good` as its arguments.
- The script runs in Bash from the root of the pull request's branch. It stops at the first failing command.
- `format` is the built-in 🎨, so a line for 🎨 can extend it rather than replace it.
- Anything longer than one line belongs in a script in your repo, like `./.github/bot/deploy.sh` above, which deploys to the environment named in the comment (`🚀 staging`).
- A command that changes no files commits nothing, so commands that only report or deploy work too.

### What a script can use

| Variable   | Value                                 |
| ---------- | ------------------------------------- |
| `ACTOR`    | The login of the commenter.           |
| `ARGS`     | The text after the trigger, or empty. |
| `COMMAND`  | The trigger that was commented.       |
| `GH_TOKEN` | The workflow's token, so `gh` works.  |
| `PR`       | The pull request's number.            |

Use them to decide who may run a command, or to reply in your own words:

```bash
#!/usr/bin/env bash
set -euo pipefail

environment="${1:-staging}"

if [ "$ACTOR" != "claudiodekker" ]; then
  gh pr comment "$PR" --body "Only @claudiodekker can deploy this pull request."
  exit 0
fi

./deploy.sh "$environment"
gh pr comment "$PR" --body "Deployed to $environment."
```

For full deployments with environments and locks, [github/branch-deploy](https://github.com/github/branch-deploy) is the better fit.

## Inputs

| Input          | Default          | Description                                            |
| -------------- | ---------------- | ------------------------------------------------------ |
| `commands`     |                  | Extra commands, one `trigger: script` per line.        |
| `node-version` | The runner's own | The Node.js version to install before the script runs. |
| `php-version`  | The runner's own | The PHP version to install before the script runs.     |

## Who can run commands

Only people with write or admin access to the repo can run commands, and only on pull requests from the repo's own branches. pr-bot ignores everything else quietly, without a reaction or a reply. It refuses forks because the workflow's token can't push to a fork, and running someone else's code with a token that can push isn't safe.

pr-bot uses [github/command](https://github.com/github/command) for the reactions.

A command with arguments only starts for commenters GitHub marks as an owner, member or collaborator of the repo. An organization member who keeps their membership private may show as neither, and then only commands without arguments work for them.

## Things to know

- **The checks wait for a click after pr-bot pushes.** GitHub doesn't start workflows automatically from a push made with the workflow's own token, so the pull request shows **Approve workflows to run** instead. One click starts them.
- **pr-bot never changes workflow files.** The workflow's token isn't allowed to push them, so pr-bot drops any change a command makes under `.github/workflows` before it commits.
- **Comments from people with write access start a short run.** GitHub can't split a trigger from its arguments before it starts a machine, so every pull request comment from someone with write access starts a run, which stops within seconds unless the first word is a trigger. That's free on public repos. On private repos, GitHub bills each run as at least a minute. Other comments cost nothing unless they're exactly a trigger.
- **The same comment runs once at a time per pull request.** Commenting the same command again waits for the run before it.

## License

pr-bot is open-source software licensed under the [MIT license](LICENSE).

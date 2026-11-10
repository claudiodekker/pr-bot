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
        🚀: ./.github/bot/deploy.sh
```

- A trigger can be any text, such as an emoji or `/fix`. The comment has to be exactly the trigger, so a trailing space or a sentence around it doesn't count.
- The script runs in Bash from the root of the pull request's branch. It stops at the first failing command.
- `format` is the built-in 🎨, so a line for 🎨 can extend it rather than replace it.
- Anything longer than one line belongs in a script in your repo, like `./.github/bot/deploy.sh` above.
- A command that changes no files commits nothing, so commands that only report or deploy work too.

### What a script can use

| Variable   | Value                                |
| ---------- | ------------------------------------ |
| `ACTOR`    | The login of the commenter.          |
| `COMMAND`  | The trigger that was commented.      |
| `GH_TOKEN` | The workflow's token, so `gh` works. |
| `PR`       | The pull request's number.           |

Use them to decide who may run a command, or to reply in your own words:

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ "$ACTOR" != "claudiodekker" ]; then
  gh pr comment "$PR" --body "Only @claudiodekker can deploy this pull request."
  exit 0
fi

./deploy.sh staging
gh pr comment "$PR" --body "Deployed to staging."
```

For full deployments with environments and locks, [github/branch-deploy](https://github.com/github/branch-deploy) is the better fit.

## Inputs

| Input          | Default          | Description                                            |
| -------------- | ---------------- | ------------------------------------------------------ |
| `commands`     |                  | Extra commands, one `trigger: script` per line.        |
| `node-version` | The runner's own | The Node.js version to install before the script runs. |
| `php-version`  | The runner's own | The PHP version to install before the script runs.     |

## Who can run commands

Only people with write or admin access to the repo can run commands. pr-bot ignores commands on pull requests from forks, because the workflow's token can't push to a fork, and running someone else's code with a token that can push isn't safe.

pr-bot uses [github/command](https://github.com/github/command) for these checks and for the reactions.

## Things to know

- **The checks wait for a click after pr-bot pushes.** GitHub doesn't start workflows automatically from a push made with the workflow's own token, so the pull request shows **Approve workflows to run** instead. One click starts them.
- **pr-bot never changes workflow files.** The workflow's token isn't allowed to push them, so pr-bot drops any change a command makes under `.github/workflows` before it commits.
- **A comment costs nothing unless it could be a command.** GitHub only starts a machine when the comment appears in your list of commands.
- **One command runs at a time per pull request.** Commenting the same trigger again waits for the run before it.

## License

pr-bot is open-source software licensed under the [MIT license](LICENSE).

# Alpacon Exec Action

[![GitHub marketplace](https://img.shields.io/badge/marketplace-alpacon--exec--action-blue?logo=github)](https://github.com/marketplace/actions/alpacon-exec-action)

Run commands on your servers from GitHub Actions without handing your workflow an SSH key.

An SSH key in a CI secret is usually a full shell on your server. With this action, the runner signs in to [Alpacon](https://alpacon.io) with a service token that runs only the command lines you list, on the servers you list. Alpacon is a server access platform: its agent on each server connects out to Alpacon, so no inbound port has to be open for the runner.

- [Official documentation](https://docs.alpacax.com/reference/actions/exec/)

> This action was named `alpacon-websh-action`. GitHub does not redirect `uses:` to a renamed action, so change `alpacax/alpacon-websh-action@v1` to `alpacax/alpacon-exec-action@v2` in your workflows and rename the `api-token` input to `service-token`.

## How access is controlled

- **Only the commands you list**—The token's access rules name the servers and command lines it may run. Anything else is refused, and the refusal is recorded with its reason. Keep the rules narrow: a rule such as `bash -c *` or `sudo *` allows anything.
- **No interactive access**—A service token cannot open a terminal session, a tunnel, or the file editor, and it never runs as `root`. Commands run as the server account its application owns.
- **No inbound port**—The Alpacon agent on each server connects out. Nothing on the server listens for the runner, so the runner's changing IP addresses don't matter.
- **Approved once, then unattended**—A token that can run commands waits for approval in the Alpacon web console and authenticates nowhere until it is approved. Runs after that don't wait for anyone.
- **Bounded**—Every token has an expiry date and a rate limit, and you can disable it at any time. You can also limit it to hours and days of the week, and, on runners with fixed addresses such as self-hosted runners, to IP ranges.
- **On the record**—Every command it runs, and every command an access rule refuses, appears in the workspace audit log.

## Authentication

Set `service-token` to an Alpacon service token (its key starts with `alpst-`). A service token belongs to an IAM Application (a machine identity), not to a person, so it keeps working when people leave. A service token does not need a Work Session; endpoint permissions still apply.

Store the token as a repository secret named `ALPACON_SERVICE_TOKEN`, as in the examples below. Logging in with a service token requires Alpacon CLI v1.7.4 or later.

See [Service tokens](https://docs.alpacax.com/use/iam/applications/service-tokens/) and the [GitHub Actions integration guide](https://docs.alpacax.com/integrate/github-actions/).

## Usage

### Basic deployment

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Alpacon CLI
        uses: alpacax/alpacon-setup-action@v1

      - name: Deploy to production
        uses: alpacax/alpacon-exec-action@v2
        with:
          workspace-url: ${{ secrets.ALPACON_WORKSPACE_URL }}
          service-token: ${{ secrets.ALPACON_SERVICE_TOKEN }}
          target: 'prod-server'
          script: cd /opt/myapp && git pull && npm install && pm2 restart app
```

### Pass environment variables

```yaml
- name: Deploy with env vars
  uses: alpacax/alpacon-exec-action@v2
  with:
    workspace-url: ${{ secrets.ALPACON_WORKSPACE_URL }}
    service-token: ${{ secrets.ALPACON_SERVICE_TOKEN }}
    target: 'prod-server'
    env: |
      APP_ENV=production
      DB_HOST=db.internal
    script: |
      echo "Deploying to $APP_ENV"
      systemctl restart myapp
```

The `env` input supports two formats:
- `KEY=VALUE`—sets the variable to the specified value
- `KEY`—forwards the variable from the runner environment

### How `script` runs

Each non-empty, non-comment line of `script` runs as its own command, in order, and the step stops at the first line that fails. A `cd` or a shell variable does not carry over to the next line, so put dependent steps on one line with `&&`. Every part of a chained line must match an access rule, and redirections such as `>` are always refused.

## Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `workspace-url` | Your Alpacon workspace URL | Yes | |
| `service-token` | Alpacon service token key (`alpst-`) | Yes | |
| `target` | The target server name to execute commands on | Yes | |
| `script` | The command or commands to run, one per line | Yes | |
| `username` | Server account to run the command as. A service token runs only as its application's account, never as `root` | No | |
| `env` | Environment variables to pass to the remote command (one per line, KEY=VALUE or KEY format) | No | |

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `alpacon: command not found` | CLI not installed | Add `alpacax/alpacon-setup-action@v1` before this action |
| `Login failed` | Wrong workspace URL or token; the token is expired, disabled or still waiting for approval; or the runner is outside the token's allowed IP ranges or hours | Check both secrets, and the token's status and restrictions in the web console |
| Command refused | The server or the command line isn't in the token's access rules, the line contains a redirection, or (Alpacon Cloud) the command was rated Critical | For a rule refusal, add a narrow rule for the server and command, and check every part of a chained line. Remove a redirection; no rule allows it. For a Critical rating, use a different command |
| Command not executing | Empty or comment-only script | Ensure script contains non-empty, non-comment lines |

## Related actions

- [alpacon-setup-action](https://github.com/alpacax/alpacon-setup-action)—Install Alpacon CLI (required)
- [alpacon-cp-action](https://github.com/alpacax/alpacon-cp-action)—Copy files to/from remote servers
- [alpacon-common-action](https://github.com/alpacax/alpacon-common-action)—Run any Alpacon CLI command

## Resources

- [GitHub Actions integration guide](https://docs.alpacax.com/integrate/github-actions/)
- [Service token command patterns](https://docs.alpacax.com/use/iam/applications/service-tokens/command-patterns/)
- [Alpacon CLI reference](https://docs.alpacax.com/reference/cli/)
- [alpacon exec command reference](https://docs.alpacax.com/reference/cli/exec/)

## Releasing

When creating a new release, always update the `v2` major version tag:

```bash
git tag -f v2 v2.x.0
git push origin v2 --force
```

This ensures users referencing `@v2` automatically get the latest release.

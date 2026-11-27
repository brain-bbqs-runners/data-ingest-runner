# Data Ingest (Runner)

Self-hosted runner for the BBQS [Data Ingest Task Force](https://github.com/brain-bbqs/data-ingest-task-force). Each run checks out that repository and runs its `dispatch/dispatch.py`, which downloads each lab's incoming dandiset, converts new sessions and uploads the standardized output.

## Workflows

- **Scheduled Ingest** (`scheduled_ingest.yml`) runs daily on its cron.
- **Manual Ingest** (`manual_ingest.yml`) runs from the Actions tab. Its inputs are `dry_run`, `only` (a lab, or `<lab>/<project>`) and `task_force_ref` (a branch, tag or SHA of the task-force repo, `main` by default).
- **Rebuild Runner Tools** (`rebuild_tools.yml`) reinstalls the runner's tools.
- **Runner Watchdog** (`runner_watchdog.yml`) emails when the runner goes offline or comes back.

The ingest job itself lives in the reusable `ingest.yml`. The ingest and rebuild workflows share one `concurrency` group, so they never overlap. A failed or cancelled ingest run sends an email.

## Runner setup

On a Linux host, as the account that will run the jobs, download the runner from Settings -> Actions -> Runners -> New self-hosted runner. Then, in its folder:

```bash
sudo ./bin/installdependencies.sh
./config.sh --url https://github.com/brain-bbqs-runners/data-ingest-runner \
    --token <REGISTRATION_TOKEN> --name <runner-name> --labels ember,ingest
./run.sh
```

The token is on that same settings page. To keep the runner running after logout, use `sudo ./svc.sh install && sudo ./svc.sh start` instead of `./run.sh`.

The host needs `python3` (3.10+), `git` and `curl`. It also needs unprivileged user namespaces, so `unshare --user --map-root-user true` must succeed. Nothing else is installed by hand. The first run installs Apptainer, git-annex, DataLad, datalad-container and con-duct into `~/ember-tools`, without root.

Repository secrets:

| Secret | What it is |
| --- | --- |
| `EMBER_DANDI_API_KEY` | DANDI API key for the `ember-dandi` instance |
| `TRACKING_TOKEN` | Fine-grained token for [`data-ingest-runner-tracking`](https://github.com/brain-bbqs/data-ingest-runner-tracking), Contents read and write |
| `MAIL_USERNAME`, `MAIL_PASSWORD` | Gmail address and app password for the notification emails |
| `RUNNER_STATUS_TOKEN` | Fine-grained token for this repository, Administration read and Variables read and write, used by the watchdog |

Optional repository variables are `TASK_FORCE_REF` (a standing override of the task-force ref), `DISK_ALERT_PERCENT` (the disk-usage alert threshold, 50 by default), and `REGISTRY_PATH` and `SESSIONS_PATH` (absolute paths on the runner to a `projects.json` and `sessions.json` that replace the task-force repository's copies).

## Why doesn't this repository allow pull requests from external forks?

A pull request from a fork could run hostile code on this self-hosted runner and reach its secrets. Please raise an issue instead.

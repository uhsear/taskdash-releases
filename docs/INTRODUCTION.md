# Introduction to TaskDash

## Introduction

TaskDash watches the scheduled tasks on one Windows computer. It runs every hour, reads the Task
Scheduler, and publishes what it finds to ArcGIS Online. A dashboard shows the result.

Use TaskDash to answer these questions:

- Which tasks failed, never ran, are disabled or are stuck?
- Which tasks are overdue?
- Which tasks changed recently, for example a new task, a removed task or a task that started to fail?
- What did each task's log files report, such as records processed or error lines?

TaskDash reads three sources on the computer:

- The task definitions and their last result.
- The run history in the Windows event log `Microsoft-Windows-TaskScheduler/Operational`.
- The log files of each task, which TaskDash mines for counts, durations and error lines.

It writes the rows to five hosted tables. It never changes a task. A read-only view of those
tables feeds the dashboard. If you turn on e-mail alerts, TaskDash sends an e-mail when a task fails
or is overdue, and when it recovers. It can also send one daily check e-mail.

TaskDash protects what it publishes. By default, it hides command lines, account names, paths and
log samples. It also removes secrets from the free text that it publishes. See [SECURITY.md](../SECURITY.md).

## Deploy the solution

Follow [INSTALL.md](INSTALL.md). It takes you from nothing to a working dashboard in about one
hour. The guide covers the install and the ArcGIS sign-in. It then covers the creation of the
ArcGIS items, the hourly task and the check of the first run.

After the install, use these guides:

- [CONFIGURE.md](CONFIGURE.md) describes every setting.
- [USE.md](USE.md) shows how to read the dashboard and the alert e-mails.
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) lists each error message and its fix.

## Requirements

### ArcGIS Online

- **Account that installs and runs TaskDash:** a built-in ArcGIS account with a user name and a
  password. Multifactor authentication must be off for it. An account that signs in only through
  single sign-on (SAML) cannot be used.
- **User type:** Creator. A Creator has the privileges that TaskDash needs. `taskdash preflight`
  checks these privileges:
  - create items;
  - create groups;
  - share items to groups;
  - edit features.
- **Sharing:** `taskdash setup` shares only the view and the dashboard, and only with its own
  private group. TaskDash never shares anything with everyone.
- **Internet access** from the computer over HTTPS to `www.arcgis.com`, your organization's
  `<org>.maps.arcgis.com` and `*.arcgis.com`. A proxy is possible. INSTALL.md shows how to set it.

ArcGIS Enterprise is not tested yet.

### External components

The computer that runs TaskDash needs these components:

- **Windows:** Windows 10, Windows 11 or Windows Server 2016 or later.
- **PowerShell:** Windows PowerShell (`powershell.exe`). TaskDash calls it to read the Task
  Scheduler, and the task installer is a PowerShell script.
- **Python:** 64-bit Python 3.11 from python.org, installed for all users. This is the tested version.
  The package declares Python 3.9 or later.
- **The `arcgis` package:** version 2.4 or later and earlier than 3. The wheel installs it.
- **The `keyring` package:** only if you turn on e-mail alerts with a mail user name. It is
  an optional extra, `taskdash[smtp]`. The mail password comes from the Windows keyring.
- **A Windows account for the task:** a member of the local Administrators group, with a password.
  A standard account sees only some of the tasks.
- **ArcGIS Pro:** TaskDash does not need it. Install it only where the tasks you monitor run arcpy.
  If it is there, you can install TaskDash into a clone of Pro's Python environment. INSTALL.md
  shows how.

### Tested scale

TaskDash is tested on one computer with 29 tasks that run at most hourly. Larger installs may work.
Your computer can monitor more tasks, or tasks that repeat more often than hourly. Then the message
on the `tested scale` line of `taskdash preflight` starts with `WARNING`. That line is not a failure.

## Information products

TaskDash creates these items in your ArcGIS Online organization. `taskdash setup --apply` puts the
items in one folder, named `TaskDash` by default. The group is not in the folder.

| Name | Item type | Minimum user type | Shared with |
|---|---|---|---|
| TaskDash (dashboard) | Dashboard | Viewer, not tested; Creator to create it | The viewers group |
| TaskDash (service) | Hosted feature layer with five tables | Creator | Nobody (owner only) |
| TaskInventory | Table in the service | Creator | Nobody (owner only) |
| TaskRuns | Table in the service | Creator | Nobody (owner only) |
| TaskMetrics | Table in the service | Creator | Nobody (owner only) |
| CollectorRuns | Table in the service | Creator | Nobody (owner only) |
| TaskChanges | Table in the service | Creator | Nobody (owner only) |
| TaskDash_view | Hosted feature layer view, query only | Viewer, not tested; Creator to create it | The viewers group |
| TaskDash viewers | Group, private, by invitation | Viewer, not tested; Creator to create it | Not applicable |
| Alert e-mail | E-mail, not an ArcGIS item | None | The addresses in `alerting.to` |

The dashboard reads the view, not the service. The view cannot edit data, and it hides the fields
that your privacy settings hide. Members of the viewers group can open the dashboard and read the
view. The names above are the defaults. The `--title-prefix`, `--folder` and `--group-title`
options of `taskdash setup` change them.

What each table holds:

| Table | One row per | Kept for |
|---|---|---|
| TaskInventory | scheduled task, with its current state and last-run outcome | While the task exists |
| TaskRuns | task run, taken from the Task Scheduler event log | 90 days |
| TaskMetrics | metric value mined from a task's log files | 90 days |
| CollectorRuns | run of the TaskDash collector, with its timing, counts and outcome | 90 days |
| TaskChanges | detected change in a task's state | 365 days |

These periods are the defaults of `retention.runs_days` (TaskRuns and CollectorRuns),
`retention.metrics_days` and `retention.changes_days`. TaskDash removes older rows. It also removes
an inventory row when its task is gone.

The alert e-mail is off by default. It needs `alerting.enabled` set to `true`, at least one address
in `alerting.to`, and a mail server in the `smtp` section of the config. It names the host, the
failed and overdue tasks, and the other tasks that need attention. When several tasks fail for one
reason, it names the likely shared cause. It quotes the last 15 lines of a failed task's log and
links to the dashboard. It removes secrets from the log lines it quotes. The daily check e-mail is
sent only when `alerting.heartbeat_hour` is set. The example config sets it to 7.

## Release notes

### 1.1.0 (2026-10-27)

The first public release. TaskDash no longer fails silently: a failed alert e-mail, a turned-off or
wrapped Task Scheduler log, an unreadable task and a slow first run each show in the run result. A
new `taskdash test-alert` command checks the e-mail path. Secret scrubbing covers more forms and
TaskDash's own log. `metrics.log_roots` can confine log reads. Each release carries Sigstore
signatures, a hash-pinned lock file and an SBOM. Before you upgrade, note that the default
collector task name is now `TaskDash Collector`; set `scheduler.collector_task_name` if your task
has another name. See [CHANGELOG.md](../CHANGELOG.md).

### 1.0.2 (2026-10-19)

Paged reads from the hosted tables are ordered by object ID. This keeps certificate checks on for
the whole run with the Python that ships with ArcGIS Pro 3.6 and 3.7. See
[CHANGELOG.md](../CHANGELOG.md).

### 1.0.1 (2026-10-18)

Fixes from the release audit. Setup keeps the view in the setup folder and lists a half-made view
for the undo. The demo explains that it signs in only through a profile. The release checksum file
works with `sha256sum -c`. See [CHANGELOG.md](../CHANGELOG.md) for the full list.

### 1.0.0 (2026-10-18)

The first stable version.

- Collects the task definitions, the run history and the log metrics of one Windows computer.
  Publishes them every hour to five hosted tables.
- Shows the result on a dashboard that reads a query-only view. The service stays owner only.
- Creates every ArcGIS item with one command, `taskdash setup --apply`, and checks the computer
  and the account with `taskdash preflight`.
- Installs as a Python wheel, with an installer script that registers the hourly task.
- Sends alert e-mails when a task fails or is overdue. One e-mail names a likely shared cause when
  several tasks fail for one reason. An optional daily check e-mail reports the state of all tasks.
- Pings an outside health-check address at the start and the end of each hourly run, if you
  configure one. A failed run pings the address with `/fail` added.
- Hides command lines, account names, paths and log samples from viewers by default.
- Removes secrets from the free-text fields it publishes and from alert e-mails.
- Keeps the ArcGIS password in a saved sign-in profile and the mail password in the Windows
  keyring. It stores neither in the config.
- Writes nothing to ArcGIS Online and sends no e-mail unless you pass `--apply`. Without it, a
  command only checks and reports.

See [CHANGELOG.md](../CHANGELOG.md) for the full history.

## Known limits

- One computer per install is recommended. Several computers can publish to one service, because
  every row holds the host name. This layout is not tested.
- The dashboard shows run times in the time zone of the viewer's device. The schedules use the
  clock of the computer that runs the tasks.
- TaskDash is tested on ArcGIS Online only. ArcGIS Enterprise is not tested yet.
- Scrubbing secrets from command lines and log samples is best effort. A secret in an unusual form
  may pass. Keep the privacy switches on.
- TaskDash is tested on one computer with 29 tasks that run at most hourly.
- TaskDash must run on the computer it watches. It cannot read another computer's tasks.
- The install must run at the computer's console or through Remote Desktop. A PowerShell remoting
  session cannot read the saved sign-in.
- The task account must be an administrator on the computer.
- The ArcGIS account must be a built-in account without multifactor authentication.
- Run history starts at the moment the Task Scheduler event log is turned on.

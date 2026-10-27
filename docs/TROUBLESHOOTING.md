# Troubleshoot TaskDash

TaskDash stops with a message when it cannot continue safely. This guide lists every such message,
what causes it, and how to fix it. Find your message by its first words. A `<name>` in a message
stands for a value that changes, such as a path or an item ID.

In the commands below, `taskdash` stands for the `taskdash.exe` that you installed. The install
guide calls it `$td`. Run the commands in the folder where TaskDash keeps its config, as step 1 of
[INSTALL.md](INSTALL.md) describes.

Entries are grouped by command:

- [Read the result of a run](#read-the-result-of-a-run)
- [Any command](#any-command)
- [Sign-in and config](#sign-in-and-config)
- [Collect and publish](#collect-and-publish)
- [Setup](#setup)
- [Preflight](#preflight)
- [Dashboard](#dashboard)
- [Demo](#demo)
- [Status](#status)

## Read the result of a run

If the scheduled task failed, start with its last result and its newest log. Step 8 of
[INSTALL.md](INSTALL.md) explains both, and lists what each `LastTaskResult` value means.

1. In PowerShell as administrator, run
   `(Get-ScheduledTaskInfo -TaskName "TaskDash Collector").LastTaskResult`.
2. If the value is `1`, TaskDash started and stopped with an error. Open the newest
   `taskdash_*.log` file in the `Logs` folder of the install, such as `C:\TaskDash\Logs`.
3. Find the line that contains `TaskDash exiting:`. The text after it is the message. Look it up in
   this guide.

If the config sets `logging.log_dir`, the logs are in that folder instead.

If no log has the time of the failed run, or the newest log has no `TaskDash exiting:` line,
`collect` stopped before it opened its log. A config problem causes this, because `collect` reads
the config first. To see the message, run `taskdash preflight --offline`. The `config` line shows
it.

If the log has the line `TaskDash failed with an uncaught exception.`, TaskDash met an error that
this guide does not list. The lines after it show where. Remove identifying values from them, as the rule in
[Issues](../README.md#issues) says. Then send them to the TaskDash maintainer.

When you run a command in a window, it prints its message there. The `collect` command also
writes the message to its log. The `setup`, `dashboard`, `demo` and `status` commands write no log
of their own. A message ends the command with exit code 1.

The `preflight` command does not stop at a message. It shows the message in a `[!!]` line, after
`SystemExit:`, and runs the other checks.

To see the last line of the newest log without opening it, run `taskdash status --offline`.

## Any command

### A command says it is not part of this install

> `<module>`.py is not part of this install, so this command is unavailable. Reinstall TaskDash from a complete wheel.

**Cause.** The `taskdash` command found the command's Python file missing. The files are
`taskdash_publish` (`collect`), `taskdash_setup`, `taskdash_preflight`, `taskdash_dashboard` and
`taskdash_demo`. A damaged install or an incomplete wheel causes this.

**Fix.** Run the install again with the full wheel from the release. If pip says the package is
already installed, add `--force-reinstall --no-deps` to the `pip install` command from step 1 of
[INSTALL.md](INSTALL.md).

### A module failed to import

> `<module>` failed to import. Reinstall TaskDash; if it persists, send this traceback to the maintainer.

**Cause.** The command's Python file exists but raised an error when Python loaded it. A damaged
or mixed-version install causes this. A wheel install has no wrapper script, so TaskDash also
saves the message and the traceback in `Logs\bootstrap_FAIL_<time>.log` under the `TASKDASH_HOME`
folder, or the working folder if that is not set. A wheel install saves a config error there too,
before the run log exists. TaskDash hides secrets in the text and deletes the file with the other
old logs. The folder install keeps its own `bootstrap_FAIL_` file through `run_taskdash.cmd`.

**Fix.** Read the traceback. Run the install again with the full wheel from the release, as in the
fix above.

### A command prints a usage line and an error

> taskdash: unknown command '`<name>`'

**Cause.** The command name or a flag is wrong, or two flags cannot be used together. The
command-line parser then prints a `usage:` line and an error, and ends with exit code 2. For an
unknown command name, `taskdash` prints the message above and the list of commands. It also ends
with exit code 2.

**Fix.** Read the error line. Run `taskdash <command> --help` to see the flags of that command.
The reference part of [CONFIGURE.md](CONFIGURE.md) lists the flags.

### A log line was withheld

> log line withheld: secret scrubbing failed

**Cause.** TaskDash removes secrets from every line before it writes it to its own log. The
scrubber raised an error on one line, so TaskDash wrote this text instead of the line. It never
writes a line that it could not scrub. The run continues.

**Fix.** If the text repeats, check the rules files for a recent change to `secret_words`, then
re-run. If it still repeats, send the time of the line to the maintainer. Do not send the task's
command line.

## Sign-in and config

These messages come from reading `TaskDash.config.json` and from signing in to ArcGIS Online. Every
command that reads the config can show them.

### The config file is missing

> Config not found at `<path>`: copy TaskDash.config.example.json to `<file name>` in `<folder>` and fill it in.

**Cause.** TaskDash looks for `TaskDash.config.json` in the folder it starts in. It did not find the
file there, or the file named by `--config` does not exist.

**Fix.** Do one of these:

1. Change to the install folder with `Set-Location C:\TaskDash`, then run the command again.
2. Pass the file with `--config <path>`.
3. Set the `TASKDASH_HOME` environment variable to the folder that holds the config.
4. If the file does not exist, create it as step 2 of [INSTALL.md](INSTALL.md) describes.

### The config is not valid JSON

> Config at `<path>` is not valid JSON (line `<number>`): `<detail>`

**Cause.** The file has a JSON syntax error at or just before the named line. A missing comma, an
extra comma after the last item, or a missing quote are the usual causes.

**Fix.** Open the file in an editor, go to the line, and correct the syntax. Run
`taskdash preflight --offline` to check it again.

### The config is not UTF-8 text

> Config at `<path>` is not UTF-8 text. Open it in an editor and save it as UTF-8, then run again.

**Cause.** The file uses another encoding. Windows PowerShell 5.1 writes UTF-16 with `Out-File`.

**Fix.** Open the file in an editor and save it as UTF-8. Run `taskdash preflight --offline` to
check it again.

### The config cannot be read

> Cannot read the config at `<path>`: `<detail>`. Give the account that runs TaskDash read access to the file, then run again.

**Cause.** The account that runs TaskDash has no read access to the file, or another program
locks it.

**Fix.** Give the task account read access to the file. Close any program that holds it open.

### The config is not a JSON object

> Config at `<path>` must be a JSON object, not `<type>`.

**Cause.** The file is valid JSON, but its top level is not an object. For example, it is a list.

**Fix.** Make the file start with `{` and end with `}`. Compare it with
`TaskDash.config.example.json`.

### The config has problems

> Config at `<path>` has `<number>` problem(s) -- fix and re-run:

One line follows for each problem.

**Cause.** A key has a value of the wrong type, or a key in the `privacy` section is unknown. A
line of the first kind reads `<key> must be <type>, got <value>`. A line of the second kind reads
`unknown key 'privacy.<key>' (valid privacy keys: ...)`. A mistyped privacy key stops the run on
purpose. Otherwise the field it was meant to hide would stay published. TaskDash shows a password
value as `<redacted>` in these lines.

**Fix.** Correct each line, then run the command again. The `_notes` section of
`TaskDash.config.example.json` explains each key. [CONFIGURE.md](CONFIGURE.md) lists each key and
its type.

### The config has no rules profiles

```text
Config at <path> has no rules.profiles. Add "rules": {"profiles": []} to use only rules/default.json, or list your rules files, e.g. {"profiles": ["rules/<org>.json"]}.
```

**Cause.** The config has no `rules.profiles` key. TaskDash will not guess, because an install that
has its own rules file would lose those rules without a warning.

**Fix.** Add `"rules": {"profiles": []}` to the config to use only the shipped `rules/default.json`.
To use your own rules file, list its path in `profiles`. TaskDash reads that path relative to the
install folder.

### A rules file has a problem

> Rules problem in `<where>`: `<problem>`. Fix: `<fix>`, then re-run.

**Cause.** A rules file (`rules/default.json` or a file listed in `rules.profiles`), or the
`metrics.tasks` section of the config, has an entry that TaskDash cannot load. `<where>` names the
file and the entry. `<problem>` says what is wrong. These problems are the usual ones:

- A pattern is not a valid regular expression. In JSON, write one backslash as two.
- The file cannot be read, is not valid JSON, or does not hold a JSON object.
- A section name is unknown. Start the name of a note with `_`.
- An entry has no `order` number, or two entries use the same `order`.
- A `cause_text` entry is missing for a cause that a rule can produce.
- A `secret_words` entry is empty, or the section is not a list of words.

**Fix.** Do what the `Fix:` part of the message says. Paths in `rules.profiles` are relative to the
install folder, the folder that TaskDash starts in.

### TaskDash cannot reach ArcGIS

```text
Cannot reach arcgis.url '<url>'. Check the URL in the config (ArcGIS Online is https://www.arcgis.com or https://<org>.maps.arcgis.com), the network and any proxy. Detail: <error>
```

**Cause.** The computer could not open an HTTPS connection to `arcgis.url`. The URL may be wrong,
the network may be down, or a proxy may block the connection.

**Fix.**

1. Check that `arcgis.url` is spelled correctly and uses the same URL as your stored sign-in.
2. If the computer uses a proxy, set the `HTTPS_PROXY` variable as the requirements in
   [INSTALL.md](INSTALL.md) describe. Open a new PowerShell window afterwards.
3. If only the scheduled task still fails, restart the computer. Task Scheduler then sees the new
   variable.

### The stored sign-in does not work

> arcgis.profile '`<name>`' did not sign in. Store it once with GIS(url, user, password, profile='`<name>`') as the account the scheduled task runs under.

**Cause.** The Windows account that runs the command has no saved sign-in under that profile name,
or the saved password is no longer correct. Each Windows account has its own Credential Manager.
The scheduled task runs as the task account, so that account needs its own profile.

**Fix.** Store the profile again for the Windows account that ran the command. Use the create
command and the check command from step 3 of [INSTALL.md](INSTALL.md). If the scheduled task failed,
do this in a `runas` window as the task account.

### TaskDash finds no ArcGIS credentials

> No ArcGIS credentials resolved. Set arcgis.username/password in the config, or set arcgis.username_env/password_env to environment variable names.

**Cause.** `arcgis.profile` is empty. TaskDash then looks for a user name and a password in the
config or in environment variables, and found none.

**Fix.** Set `arcgis.profile` to `taskdash`, and store the sign-in as step 3 of
[INSTALL.md](INSTALL.md) describes. This is the safe option, because the password stays out of the
config.

### The config holds a plaintext password

> arcgis.password is stored in plaintext in the config. Store a profile once, as the account the scheduled task runs under: GIS(url, user, password, profile="name"), then set arcgis.profile to that name and delete arcgis.password. To accept the risk instead, set arcgis.allow_plaintext_password to true.

**Cause.** The config has an `arcgis.password` value, and no `arcgis.profile`. TaskDash refuses to
use a password that anyone with access to the file can read.

**Fix.**

1. Store a profile, as step 3 of [INSTALL.md](INSTALL.md) describes.
2. Set `arcgis.profile` to the profile name.
3. Delete `arcgis.password` from the config.

Only if you accept the risk, set `arcgis.allow_plaintext_password` to `true` instead.

## Collect and publish

These messages come from `taskdash collect`, which the hourly task runs as `collect --apply`. The
`setup` and `dashboard` commands can show some of them too.

If `collect --apply` cannot sign in or find the service, it still collects and checks alerts
before it exits with the error.

### The Task Scheduler log cannot be read

> Cannot read event log '`<log name>`': `<reason>`. Run this collector elevated, or confirm the channel exists on this host.

**Cause.** TaskDash reads run history from the Windows event log
`Microsoft-Windows-TaskScheduler/Operational`. The log does not exist, or the account that runs
TaskDash may not read it. `<reason>` is the error that Windows reported.

**Fix.**

1. Run the command as administrator. The task account must be in the local Administrators group.
2. If the reason says the log was not found, run
   `Get-WinEvent -ListLog Microsoft-Windows-TaskScheduler/Operational` in PowerShell. If that
   command also fails, the log is missing on this computer.

### The Task Scheduler log is turned off

> Event log '`<log name>`' is disabled, so run history cannot be collected. Enable it with: wevtutil sl "`<log name>`" /e:true (run elevated), then retry.

**Cause.** Windows often ships this log turned off. TaskDash cannot collect run history while it is
off. The run still publishes the task list and the overdue state, and it still sends alerts. It
records this message in `error_text` of the `CollectorRuns` row, then exits with code 1.

**Fix.** Open PowerShell as administrator and run `taskdash preflight --enable-event-log --apply`.
Step 5 of [INSTALL.md](INSTALL.md) lists the full steps. Run history starts when you turn the log on.

### A task could not be read

> WARNING: `<n>` scheduled task problem(s); the readable tasks are still collected: `<task>: <message>`

**Cause.** Task Scheduler refused to read one task definition. A common message is 0x80041321, "The
task image is corrupt or has been tampered with". Access denied gives the same warning. The run does
not fail. It lists every readable task, and it leaves out each task named in the warning.

**Fix.** Open Task Scheduler and look at the named task. Re-create the task if it is corrupt. If the
message says access denied, give the task account read access to the task. The `CollectorRuns` row
holds the same names in `error_text`, and its exit status stays success.

### TaskDash cannot list any scheduled task

> Cannot list scheduled tasks: `<task>: <message>`. Check that the Task Scheduler service is running and that the task account can read the task folders, then run again.

**Cause.** Task Scheduler returned errors and no task at all. The service may be stopped, or the task
account may have no access to the task folders. The run fails because an empty list would publish an
empty inventory and send a false "recovered" e-mail.

**Fix.** Start the Task Scheduler service, or give the task account read access to the task folders.
Then run the collector again.

### Event records were skipped

> WARNING: `<n>` event record id(s) were skipped: the oldest record left in the Operational log is `<id>` but the last one read was `<id>`. The log wrapped or was cleared since TaskDash last read it. Raise the log size with the preflight event-log fix.

**Cause.** The Task Scheduler log is a ring. It overwrote old records between two runs.
The run history for those records is lost. This run is fine, and the exit code stays 0.
TaskDash compares the oldest record with the newest record the last published run read. It keeps
that id in `Logs\event_read_mark.json`. It warns once for each gap. Without the file, which is the
case on the first run after an upgrade, it skips the check.

**Fix.** Raise the log size so it holds more than the longest gap between runs. Run preflight and
apply its event-log fix. The `CollectorRuns` row holds the same warning in `error_text`.

### Log folders were skipped

> WARNING: skipped `<n>` log folder(s) outside metrics.log_roots or reached through a link. To read them, add their parent folder to metrics.log_roots.

**Cause.** `metrics.log_roots` is set, and some task logs are outside every listed folder. A log that
is reached through a symbolic link or a junction is skipped too. The affected tasks get no log
metrics. The run itself is fine.

**Fix.** Add the parent folder of those logs to `metrics.log_roots`. If a link points to the logs,
list the link's target folder instead. See "Limit where TaskDash reads logs" in
[CONFIGURE.md](CONFIGURE.md).

### PowerShell failed

> PowerShell collection failed (exit `<code>`): `<error text>`

**Cause.** The PowerShell command that reads the task list or the event log ended with a non-zero
exit code. `<error text>` is the first 500 characters of its error output.

**Fix.** Read the error text. If it says access is denied, run the command as administrator. TaskDash
starts PowerShell without a profile, so a profile script is not the cause.

### PowerShell returned text that is not JSON

> PowerShell returned non-JSON output (`<detail>`): `<first 300 characters>`

**Cause.** PowerShell ran, but its output was not the JSON that TaskDash expects. The output after
the colon shows what it printed instead.

**Fix.** Read the printed text. TaskDash starts PowerShell without a profile, so a profile script
is not the cause. If you cannot find the cause, remove identifying values from the message, as the rule in
[Issues](../README.md#issues) says. Then send it to the TaskDash maintainer.

### An ArcGIS call failed after several tries

> `<label>` failed after `<number>` attempts: `<last error>`

**Cause.** TaskDash called ArcGIS and the call failed every time. It retries a network error, a
timeout, and the error codes 429, 498, 499, 500, 502, 503 and 504. It waits longer between tries.
`<label>` names the table and the action. `<last error>` is the error from the final try.

**Fix.** Read the last error.

- If it names a network or proxy problem, fix the connection. See "TaskDash cannot reach ArcGIS".
- If it names a sign-in or token problem, test the stored sign-in. Use the check command from step
  3 of [INSTALL.md](INSTALL.md).
- If ArcGIS Online had an outage, wait. The next hourly run tries again.

### ArcGIS refused a write

> `<label>` `<adds, updates or deletes>` failed: `<ArcGIS error>` -- a row with this key already exists (unique index). The next run matches it and updates it instead; if this repeats, look for a key written twice.

or

> `<label>` `<adds, updates or deletes>` failed: `<ArcGIS error>` -- check field lengths and types against the table schema.

**Cause.** ArcGIS accepted the request but rejected a row. The first form means a row with the
same key is already stored. The second form covers every other row error. A value that does not
fit its field is one cause.

**Fix.**

1. In the first form, wait for the next run. It finds the stored row and updates it. If the message
   repeats, search the table in ArcGIS Online for the key in the error.
2. In the second form, run `taskdash collect --migrate`. It lists the fields that the service
   lacks. If it lists missing fields, run `taskdash collect --migrate --apply`.
3. Also look in the log for `WARNING: schema type mismatch`. That line names a field whose type in
   the service differs from the type that TaskDash writes. The migration never changes a field
   type, so correct that field by hand.

### The service is not in the config

> No arcgis.`<item_id or test_item_id>` in the config. A new install runs `taskdash setup --apply`, which creates the service and fills the id in. To add a service to an existing install, run --create-service, then paste the returned item id into the config.

**Cause.** The config key for the service is empty. `--target prod`, the default, reads
`arcgis.item_id`. `--target test` reads `arcgis.test_item_id`.

**Fix.** For a new install, run `taskdash setup --apply`, as step 6 of [INSTALL.md](INSTALL.md)
describes. To add a service to an existing install, run `taskdash collect --create-service --apply`
and paste the item ID it prints into the config.

### The service or view item cannot be found

> ArcGIS item `<id>` not found or not visible to this account.

> View item `<id>` not found or not visible to this account.

> Service item `<id>` not found or not visible to this account.

**Cause.** The item ID in the config does not match an item that the signed-in account may see. The
ID may be wrong, the item may be deleted, or the account may lack access.

- The first message is about the service in `arcgis.item_id` or `arcgis.test_item_id`.
- The second message is about the view in `arcgis.view_item_id` or `arcgis.test_view_item_id`.
- The third message comes from the `dashboard` command and from setup. It names the item that the
  dashboard reads. That item is the view, or the service if the config sets no view.

**Fix.** Open the item in ArcGIS Online and compare its ID with the config. Check that the account
in the stored sign-in owns the item or has access to it.

### A service table is missing

> Target service is missing table(s): `<table names>`. Run --create-service.

> Target service has no TaskRuns table. Run --migrate --apply first.

> Target service has no TaskMetrics or TaskInventory table. Run --migrate --apply first.

**Cause.** The hosted service does not hold all the tables that TaskDash writes.

**Fix.** Run `taskdash collect --migrate --apply`. It creates the missing tables and fields. The
`--migrate` flag alone shows what it would change.

### The service schema is out of date

> Schema drift -- table(s) are missing field(s) the current code writes: Run `--migrate --apply` to add them.

The real message lists each table and its missing fields between the colon and `Run`.

> Schema drift -- table(s) are missing the unique index a retried write relies on to avoid duplicating rows: Run `--migrate --apply` to add them.

The real message lists the tables and fields between the colon and `Run`.

**Cause.** An older TaskDash created the service. The new code writes a field or relies on an
index that the service does not have yet.

**Fix.** Run `taskdash collect --migrate --apply`, then run the failed command again.

While the schema is out of date, `collect --apply` writes nothing to the service. It still sends the
failed-task alert, and it exits 1. The log says `WARNING: refusing to write to AGOL`. The next
successful run writes the saved `CollectorRuns` row.

### A schema change failed

> `<table>`: add_to_definition failed creating the table: `<reply>`

> `<table>`: created but not found on re-query -- re-run --migrate --apply.

> `<table>`: add_to_definition failed adding field(s) `<fields>`: `<reply>`

> `<table>`: add_to_definition failed adding `<unique>` index on `<fields>`: `<reply>`

**Cause.** ArcGIS Online did not accept a change to the service while `--migrate --apply` ran. The
reply at the end of the message is what ArcGIS returned. For the second message, ArcGIS reported
success, but the new table did not appear in the next query.

**Fix.** Read the reply. Check that the stored sign-in belongs to the owner of the service. Then run
`taskdash collect --migrate --apply` again. The migration skips what already exists.

### A unique index cannot be added

> `<table>`: cannot add a unique index on `<fields>` -- duplicate key(s) exist, e.g. `<key>`. Resolve the duplicates first.

**Cause.** The table holds more than one row with the same key. A unique index cannot be built until the key
is unique.

**Fix.** In ArcGIS Online, open the table data, find the rows with that key, and delete the extra
rows. Then run `taskdash collect --migrate --apply` again.

### The service has a newer schema than TaskDash

> The service `<id>` has schema version `<number>`, newer than this TaskDash (`<version>`, schema `<number>`). Upgrade TaskDash.

**Cause.** A newer TaskDash migrated this service. The older program you ran does not know the
newer schema, so it refuses to change it.

**Fix.** Install the newer wheel, as step 1 of [INSTALL.md](INSTALL.md) describes. Do not lower the
version stamp by hand.

### The version stamp could not be saved

> Could not stamp the version on service `<id>`. Re-run --migrate --apply.

**Cause.** The migration finished, but ArcGIS did not save the TaskDash version in the item.

**Fix.** Run `taskdash collect --migrate --apply` again. If it fails again, check that the stored
sign-in belongs to the owner of the service.

### A service with that name already exists

> A feature service named '`<name>`' already exists (`<id>`). Delete it or choose another name with --service-name.

**Cause.** `--create-service --apply` found a service with the same name. TaskDash never reuses an
existing service.

**Fix.** Delete the extra service in ArcGIS Online, or pass another name, such as
`--service-name <new name>`.

### Public sharing is refused

> TaskDash never shares an item with everyone (public). Share it with the organization or named groups instead; there is no override flag.

**Cause.** A sharing call asked for public access. No TaskDash command does this, and no flag turns
the check off.

**Fix.** If you see this message, remove identifying values from it, as the rule in
[Issues](../README.md#issues) says. Then send it to the TaskDash maintainer. Do not share the item
with everyone.

### An item is not private

> Item `<id>` should be private but its access is '`<access>`'. Set its sharing to Owner in ArcGIS Online, then re-run.

**Cause.** TaskDash expected a private item, and the item has a wider sharing level. The message
shows the level. `taskdash preflight` fails for the editable service at any level except Owner. A
`collect` run only warns at `org` or `shared`, and it refuses to continue at `public`.

**Fix.** In ArcGIS Online, open the item and set its sharing to **Owner**. Then run the command
again.

### An item was not shared with the group

> Item `<id>` was not shared with group(s) `<group ids>`. Share it with them by hand, or add the collector account to those groups and re-run.

**Cause.** ArcGIS did not add the item to the group. The signed-in account may not be a member of
the group.

**Fix.** Add the account to the group, then run the command again. Or share the item with the group
in ArcGIS Online.

### The service is shared with everyone

> The service `<id>` is shared with everyone. Set its sharing to Owner in ArcGIS Online, then re-run. Share only the read-only view.

**Cause.** The service is public. A view cannot hide data that the service itself exposes.

**Fix.** In ArcGIS Online, set the sharing of the service to **Owner**. Share only the read-only
view, with the organization or with named groups.

While the service is public, `collect --apply` writes nothing to it. It still sends the
failed-task alert, and it exits 1.

### The view or dashboard is shared with everyone

> The item `<id>` (arcgis.view_item_id) is shared with everyone. Set its sharing to your organization or to the viewers group in ArcGIS Online, then re-run.

**Cause.** The view or the dashboard in the config is public. The key in the message,
`arcgis.view_item_id`, `arcgis.test_view_item_id` or `arcgis.dashboard_id`, names the item. A public
view lets anyone read your task data.

**Fix.** In ArcGIS Online, set the sharing of the item to your organization or to the viewers group.
Sharing with the organization or with groups is allowed.

While the item is public, `collect --apply` writes nothing to it. It still sends the failed-task
alert, and it exits 1. If TaskDash cannot find the item, it warns and skips this check. Correct the
ID in the config, or share the item with the collector account.

### A view update failed

> View `<what>`: update_definition failed: `<reply>`

**Cause.** ArcGIS did not accept a change to the view, such as hiding fields. `<reply>` is what
ArcGIS returned.

**Fix.** Read the reply. Check that the stored sign-in belongs to the owner of the service. Then run
the command again.

### A view already exists

> arcgis.`<view_item_id or test_view_item_id>` is already `<id>`; the view exists. Remove the key only if you mean to create another view.

**Cause.** You ran `taskdash collect --create-view`, and the config already holds a view ID.

**Fix.** Do not create another view unless you need one. If you do, clear the key in the config and
run the command again.

### A view with that name exists, and it may be half built

> A feature service named '`<name>`' already exists (`<id>`). It may be half-configured by an earlier failed run, with no fields hidden. Delete it in ArcGIS Online and re-run; do not set arcgis.`<view_item_id or test_view_item_id>` to it.

**Cause.** An earlier `--create-view` or `setup` run may have stopped after it made the view and
before it hid the fields. That view could show data that the privacy switches hide.

**Fix.** Delete that view in ArcGIS Online. Run the command again. Do not put its ID in the config.

### The new view has no tables

> The new view `<id>` has no table(s) `<names>`. Run --migrate --apply on the service first, delete the view, and retry.

**Cause.** The service lacks tables, so the view built from it lacks them too. TaskDash reads the
new view several times over about two minutes before it stops.

**Fix.**

1. Run `taskdash collect --migrate --apply`.
2. Delete the new view in ArcGIS Online.
3. Run the command that creates the view again.

### Anonymous users may read the data

> Anonymous users may read TaskDash data, or the check could not prove they cannot:

One line follows for each table. After the lines, the message adds the fix.

**Cause.** TaskDash queries each table without signing in. It does this after
`--migrate --apply`, after `--create-view --apply` and after `setup --apply`. Each line in the
message names a table whose answer was not a clear refusal. The table answered with rows, or the
answer was inconclusive. TaskDash does not treat a bare HTTP 401 or 403 as proof, because a proxy
can send those.

**Fix.**

1. In ArcGIS Online, set the sharing of the service to **Owner**.
2. Set the sharing of the view to your organization or to named groups.
3. Run `taskdash collect --check-anonymous`. It exits with 0 when every table refuses.
4. If a line says the check was inconclusive, open that query URL in a private browser window. It
   must ask you to sign in.

### A collection from the wrong host is refused

> Refusing to publish: host_name_override is '`<name>`' but this machine is '`<computer name>`'. Collected data would describe the wrong host. Run this on '`<name>`', or pass --allow-host-mismatch if you are certain.

**Cause.** `host_name_override` in the config differs from the name of this computer. With
`--apply`, TaskDash would store this computer's tasks under the other name.

**Fix.** If you installed TaskDash by [INSTALL.md](INSTALL.md), set `host_name_override` to `""`, so
TaskDash uses the name of this computer. Or run the command on the computer that the config names.
Pass `--allow-host-mismatch` only if you are sure.

### The inventory update would delete most stored tasks

> `<label>`: refusing to delete `<number>` of `<number>` stored task(s) for host '`<name>`' -- a drop of more than half usually means collection ran on the wrong host. Re-run on the right machine, or pass --force if the shrink is real.

**Cause.** More than half of the tasks stored for this host are missing from this collection.
That usually means the command ran on another computer, or ran as an account that sees only some
tasks.

**Fix.**

1. Check the computer name and the account that ran the command. The task account must be in the
   local Administrators group.
2. If the drop is real, for example after you removed many tasks, run
   `taskdash collect --apply --force`.

### Run collection is stalled

> Run collection is stalled: the oldest events past the watermark filled the whole budget and none belongs to a monitored task. Raise history.max_events_per_run, or run once with --full-history to jump to the current log window.

**Cause.** TaskDash reads run events in order from a saved position. The next block of events held
no run of a monitored task, so the position could not move. Without this stop, every run would read
the same events.

**Fix.** Do one of these:

1. Raise `history.max_events_per_run` in the config.
2. Run `taskdash collect --apply --full-history` once. It ignores the saved position and reads the
   event-log window, up to `history.first_run_max_events` events.

### PowerShell did not finish

> PowerShell did not finish within `<n>` s. Raise powershell_timeout_seconds in the config, or check that the computer is not overloaded.

> Reading up to `<n>` events from the Operational log did not finish in `<n>` s. Raise powershell_timeout_seconds, or lower history.first_run_max_events.

**Cause.** A PowerShell call took longer than `powershell_timeout_seconds` (default 180). The second
message is the event-log read. On a first run it asks for up to `history.first_run_max_events`
events (default 30000), which a busy log cannot always return in time. For a later run, the
message names `history.max_events_per_run` instead.

The run still publishes the task list and sends the alerts, then exits 1. It reads no events, so the
watermark does not move.

**Fix.** Do one of these:

1. Raise `powershell_timeout_seconds` in the config.
2. Lower `history.first_run_max_events`.
3. Wait. You need to do nothing else. A first read that times out saves half of its size in
   `Logs\first_run_budget.json`. Each later run halves it again, down to 250 events, until a read
   gets through. From 30000, that takes at most 8 runs.

The read that gets through keeps only the newest events in the window, so older runs are not
backfilled. TaskDash then deletes `first_run_budget.json`. To read the full size again sooner,
delete the file, or change `history.first_run_max_events`.

## Alert e-mail

### The alert e-mail failed

> Alert e-mail failed: `<reason>`. The data was published, but nobody was told about the problem. Check smtp.host, smtp.port, smtp.from and smtp.user in the config and the stored SMTP password, then run `taskdash test-alert --apply` to prove the mail path. The next run tries again.

**Cause.** An alert was due, and TaskDash could not send it. The mail server refused or did not
answer, the stored SMTP password is wrong or missing, or alerting is on but there are no usable mail
settings. The run still publishes everything first. It then saves `Alert e-mail failed: <reason>` in
`error_text` of the `CollectorRuns` row, sends the failure ping to the health check, and exits with
code 1. The row shows `alert_sent` as 0 and an exit status of failed.

**Fix.** Do these steps:

1. Run `taskdash test-alert`. It prints the recipients and the server, and it sends nothing.
2. Run `taskdash test-alert --apply`. It sends one test message and prints the reason if it fails.
3. Fix the `smtp` settings or the stored password named in the reason. The mail password steps are
   in [CONFIGURE.md](CONFIGURE.md).
4. Wait for the next hourly run. TaskDash keeps the alert state until a send works, so it sends the
   same alert again. Nothing is lost.

### The test alert has no recipient

> alerting.to is empty, so the test message has no recipient. Add at least one address to alerting.to in the config.

**Cause.** `taskdash test-alert` sends to the addresses in `alerting.to`, and the list is empty.

**Fix.** Add at least one address to `alerting.to`, then run the command again.

### The test alert has no mail settings

> There are no usable mail settings, so the test message cannot be sent. Fix the smtp block in the config (host, from, user and the stored password); the WARNING above says which part is missing.

**Cause.** The `smtp` object lacks `smtp.host` or `smtp.from`, or the password for `smtp.user` is
not in the keyring of this Windows account.

**Fix.** Complete the `smtp` object, and store the password as the account that runs the task. The
steps are in [CONFIGURE.md](CONFIGURE.md). Run the command again.

### The legacy mail file warning

> WARNING: alerting.mail_config_path is deprecated and TLS is not verified for it, so the mail server's certificate is never checked. Move to the smtp block (smtp.host, smtp.from, smtp.user; store the password with 'python -m keyring set taskdash-smtp <smtp.user>'), then clear alerting.mail_config_path.

> WARNING: the legacy mail file holds a login but turns TLS off, so the password crosses the network in plain text. Move to the smtp block with smtp.starttls set to true.

**Cause.** `alerting.mail_config_path` is set and the `smtp` object is empty. TaskDash still sends
alerts through the old file. It does not check the mail server's certificate for that file. The
collector logs the first warning once per run, and `taskdash preflight` shows it on the `smtp` line.
The second warning appears when the file also holds a login and turns TLS off.

**Fix.** Fill the `smtp` object, store the password in the keyring, and clear
`alerting.mail_config_path`. The steps are in [CONFIGURE.md](CONFIGURE.md). Run `taskdash preflight`
and `taskdash test-alert --apply` to prove the new path. Alerts keep working until you move.

## Setup

These messages come from `taskdash setup`. Setup checks each name before it writes anything, so
the first five messages leave ArcGIS Online unchanged. For the steps that follow a failure after
setup began to write, see step 6 of [INSTALL.md](INSTALL.md).

### The config already holds an install

> The config already sets `<keys>`, so an install exists. Setup is for a new install: use a config without those keys, or maintain this one with taskdash_publish.py --migrate and taskdash_dashboard.py.

**Cause.** One or more of `arcgis.item_id`, `arcgis.view_item_id` and `arcgis.dashboard_id` is
not empty. Setup does not change an install that already exists.

**Fix.** To keep the install, maintain it with `taskdash collect --migrate --apply` and
`taskdash dashboard --target prod`. To start again, do these steps:

1. Delete the ArcGIS items, as Undo step 2 of [INSTALL.md](INSTALL.md) describes.
2. Empty the three keys in the config.
3. Run `taskdash setup --apply` again.

### The folder name is empty

> --folder must be a folder name. Pass e.g. --folder TaskDash.

**Cause.** The `--folder` value was empty or held only slashes.

**Fix.** Pass a folder name, such as `taskdash setup --folder TaskDash`.

### The folder already exists

> The folder '`<name>`' already exists for this account. Choose another with --folder, or delete the empty folder in ArcGIS Online.

**Cause.** The signed-in account already has a folder with this name. Setup does not reuse it.

**Fix.** Pass another name with `--folder`. Or delete the empty folder in ArcGIS Online.

### An item with the planned title already exists

> `<item type>` '`<title>`' already exists (`<id>`). Setup never reuses an item: pass --title-prefix to pick other titles, or delete that item in ArcGIS Online.

**Cause.** The account already owns an item with one of the planned titles. The item type is
`Feature Service` or `Dashboard`.

**Fix.** Pass `--title-prefix <text>` to put text before every title. Or delete the item in
ArcGIS Online, if nothing uses it.

### Another account holds the service name

> Another account in this organization already has a hosted service named '`<title>`'. Pass --title-prefix to pick other names.

**Cause.** Hosted service names are unique across the whole organization. Another account owns a
service or view with one of the planned names. Setup checks this before it creates anything.

**Fix.** Pass `--title-prefix <text>` to put text before every title. If `taskdash demo --apply`
shows this message, another account already ran the demo. Ask that person to delete the `DEMO_`
items, or run the demo from that account.

### The viewers group already exists

> A group titled '`<title>`' already exists (`<id>`). Choose another with --group-title, or delete the group.

**Cause.** A group with the planned title already exists. Setup does not reuse it.

**Fix.** Pass another title with `--group-title`. Or delete the group in ArcGIS Online.

### The group could not be created

> ArcGIS did not create the group '`<title>`'. Check that the account may create groups, then re-run; delete anything listed above first.

**Cause.** ArcGIS returned no group. The account may lack the privilege to create groups.

**Fix.**

1. In ArcGIS Online, delete the items that setup listed above this message. Use the order of Undo
   step 2 in [INSTALL.md](INSTALL.md).
2. Copy the newest `TaskDash.config.json.bak_setup_*` file over `TaskDash.config.json`.
3. Ask your ArcGIS administrator to let the account create groups.
4. Run `taskdash setup --apply` again.

### The new dashboard is not visible

> The new dashboard `<id>` is not visible to this account. Find it in ArcGIS Online, move it into '`<folder>`' and share it with the group.

**Cause.** Setup created the dashboard but could not read it back. The ID is already in the config.

**Fix.**

1. Find the dashboard in ArcGIS Online.
2. Move it into the named folder, and share it with the viewers group that setup created.
3. Run `taskdash collect --check-anonymous`. Setup stopped before its own anonymous check.

Do not run setup again, because the config now holds the install.

### The dashboard could not be moved

> Could not move dashboard `<id>` into '`<folder>`': `<reply>`

**Cause.** ArcGIS refused to move the dashboard into the folder.

**Fix.**

1. In ArcGIS Online, move the dashboard into the folder by hand.
2. Share the dashboard with the viewers group that setup created.
3. Run `taskdash collect --check-anonymous`. Setup stopped before its own anonymous check.

The ID is already in the config, so do not run setup again.

### Setup ends with exit code 0

Exit code 0 is not an error. Setup gives 0 when it finished or when a dry run printed its plan. A
message from this guide gives exit code 1.

## Preflight

### Preflight ends with exit code 1

Preflight prints no error message for this exit code. The last line counts the problems and the
warnings, such as `Preflight: 3 problem(s), 0 warning(s).` The exit
code is:

- 0 when the count of problems is 0.
- 1 when the count of problems is 1 or more.

A warning line also starts with `[!!]`, but a warning alone does not change the exit code.

With `--enable-event-log`, the exit code is 1 in these cases:

- `wevtutil` could not start. The output starts with `Could not run wevtutil`.
- With `--apply` only: `wevtutil` failed. The output starts with `wevtutil sl failed`.
- With `--apply` only: the log is still not enabled, or its size is not the wanted size.

In every other case it is 0.

Step 5 of [INSTALL.md](INSTALL.md) explains each preflight line.

## Dashboard

These messages come from `taskdash dashboard`. The command builds the dashboard definition and
checks it before it posts anything. Without `--apply`, it writes the definition to a local file,
prints its plan, and changes nothing in ArcGIS Online.

### The collector interval is not above 0

> scheduler.collector_interval_minutes must be a number above 0, got `<value>`. Set it to the minutes between collector runs, or remove the key for 60.

**Cause.** The dashboard uses this value to decide when a collector run is late. The value is 0 or
less.

**Fix.** Set `scheduler.collector_interval_minutes` to the minutes between runs, such as `60`. Or
remove the key.

### The service has no tables

> The `<target>` service has no `<table names>` table(s). Create them first: python taskdash_publish.py --migrate --target `<target>` --apply

**Cause.** The item that the dashboard reads lacks tables. That item is the view, or the service if
the config sets no view.

**Fix.** Run `taskdash collect --migrate --target <target> --apply`, then run the dashboard command
again. If the message repeats for a view, see "The new view has no tables".

### The dashboard item is wrong

> arcgis.dashboard_id `<id>` is not a Dashboard visible to this account. Correct the id in the config; this script never creates a second prod dashboard.

**Cause.** With `--target prod`, TaskDash updates the dashboard in `arcgis.dashboard_id`. That item
is not a dashboard, or the account cannot see it.

**Fix.** Open the dashboard in ArcGIS Online. Copy its ID into `arcgis.dashboard_id`.

### The test title matches the production dashboard

> Refusing: the test title '`<title>`' resolves to the prod dashboard `<id>`. Rename one of them or pass a different --title.

**Cause.** With the default `--target test`, the title of the test dashboard matches the production
dashboard. `--apply` would overwrite the production dashboard.

**Fix.** Pass a different title, such as `taskdash dashboard --title <new title>`.

### More than one dashboard has the title

> `<number>` dashboards are titled '`<title>`' (`<ids>`). Delete the extras so --apply cannot update the wrong one.

**Cause.** Several dashboards that you own have the same title. TaskDash will not guess which one to
update.

**Fix.** Delete the extra dashboards in ArcGIS Online. Or pass another title with `--title`.

### The backup could not be read

> Could not read dashboard `<id>` for a backup; not overwriting it.

**Cause.** Before `--apply` replaces a dashboard, TaskDash saves its live definition. The read
returned nothing, so TaskDash left the dashboard unchanged.

**Fix.** Open the dashboard in ArcGIS Online and check that it opens. Then run the command again.

### The dashboard could not be published

> Dashboard publish failed: `<reply>`

**Cause.** ArcGIS rejected the request to create or update the dashboard. `<reply>` is its answer.

**Fix.** Read the reply. Check that the account may create items and may edit the dashboard. Then
run the command again.

### Offline and apply cannot be combined

> --offline never signs in, so it cannot --apply. Drop one of the two flags.

**Cause.** `--offline` builds and checks the definition without signing in. `--apply` needs a
sign-in.

**Fix.** Remove `--offline` or remove `--apply`.

### The dashboard has no service to read

> No arcgis.`<item_id or test_item_id>` in the config: the `<target>` dashboard needs its service.

**Cause.** The config key for the service is empty. `--target test`, the default, reads
`arcgis.test_item_id`. `--target prod` reads `arcgis.item_id`.

**Fix.** If you meant the production dashboard, add `--target prod`. Setup fills only
`arcgis.item_id`, so a test dashboard needs `arcgis.test_item_id` set to a test service.

### The widget templates are missing

> Widget templates unavailable: dashboard_templates.json is missing and source dashboard `<id>` no longer exists. Reinstall the TaskDash wheel to restore dashboard_templates.json.

```text
No widget templates: dashboard_templates.json is missing. Reinstall the TaskDash wheel to restore it, or pass --source-dashboard <id> of any dashboard whose widget structures to borrow.
```

> dashboard_templates.json is missing required template widget(s): `<names>`

**Cause.** TaskDash builds the dashboard from `dashboard_templates.json`. The wheel installs this
file under `share\taskdash` in the Python environment. The file is missing, or it lacks a widget
that the build needs.

**Fix.** Reinstall TaskDash from the full wheel. That restores the file. Do not edit it.

### The definition check refuses the dashboard

The `dashboard` command and `setup` check the definition before they post it. Each check below
protects viewers from a widget that Dashboards would show wrongly. The command stops and posts
nothing. If setup stops here, it has already created the items that it lists. Step 6 of
[INSTALL.md](INSTALL.md) explains how to recover. The first message is a check on the definition
as a whole. The table lists the others.

> Refusing to publish: definition still contains a hardcoded host filter.

**Cause.** The definition holds the name of this computer or of `host_name_override` in quotes, or
it holds `"server_name"` with its quotes. A dashboard with a fixed computer name would show only that
computer.

**Fix.** If you edited `dashboard_templates.json`, remove the name. Otherwise remove identifying
values from the message, as the rule in [Issues](../README.md#issues) says, and send it to the
TaskDash maintainer.

The messages below come from a defect in the widget templates or the build code. Your ArcGIS setup
does not cause them.

**Fix for every message in this table.** If you edited `dashboard_templates.json`, reinstall
TaskDash from the full wheel. That restores the shipped copy. Otherwise remove identifying values
from the message, as the rule in [Issues](../README.md#issues) says, and send it to the TaskDash
maintainer.

| Message | What it checks |
|---|---|
| Dashboard references `<field>`, which is not a field on `<table>`. Add it to the matching *_fields() list in taskdash_publish.py, or correct the widget. | A widget names a field that the table does not have. |
| A list template names a field, but its Arcade has no single 'attributes: {' block to carry it -- add the field to the expression. | A list widget cannot carry the field it names. |
| Refusing to publish: '`<widget>`' is categoryType 'features' but its dataset still declares groupByFields/statisticDefinitions -- a features chart must read raw rows, not a grouped/aggregated query. | A chart that plots rows also asks for grouped data. |
| Refusing to publish: '`<widget>`' series is bound to x=`<x>`/y=`<y>` but its dataset returns x=`<x>`/y=`<names>`. | A chart series does not match its data. |
| Refusing to publish: '`<widget>`' filters `<field>` as `<type>` but the schema says `<type>`. Dashboards renders a wrong filter type as 'Cannot access data'. | A filter uses the wrong field type. |
| '`<widget>`': valueFields and columnSettings disagree. | A table widget lists different columns in two places. |
| Refusing to publish: '`<widget>`' shows `<fields>`, which its Arcade data source does not return. Add them to the script's fields. | A table shows a column that its script does not return. |
| Refusing to publish: '`<widget>`' column `<name>` is a raw layer field but is neither a groupByField nor a statistic alias of this grouped dataset -- Dashboards can only return group-by fields and aggregated statistics from a grouped query. | A grouped table shows a raw column. |
| '`<widget>`' computes `<names>` but shows no column for them. Dashboards renders an undisplayed statistic as an extra unlabelled column with an error marker on every row. | A statistic has no column. |
| '`<widget>`' column `<name>` has no number format; an unrounded average renders with 10+ decimals (defect 4.10). | An average column has no number format. |
| Refusing to publish: `<source>` -> `<target>` targets a widget that does not exist in this view. | A click action points to a missing widget. |
| Refusing to publish: `<source>` -> `<target>` names dataset `<name>`, which '`<widget>`' does not have. | A click action points to a missing dataset. |
| Refusing to publish: `<source>` -> `<target>` is not a 5.x whereClause target; a legacy flat action filters nothing and shows no error. | A click action uses an old format. |
| Refusing to publish: `<source>` -> `<target>` crosses from `<table>` to `<table>` without a fieldMap; keyed on OBJECTID it matches nothing (ce0221b). | A click action joins two tables without a key. |
| Refusing to publish: `<source>` -> `<target>` maps `<number>` fields. Dashboards applies only the first (probed 2026-09-28); join on one key field instead. | A click action maps more than one field. |
| Refusing to publish: `<source>` -> `<target>` maps `<name>`, which the Arcade data source of '`<widget>`' does not return. | A click action maps a field that the script does not return. |
| Refusing to publish: '`<widget>`' tells the reader to select a row, but selecting does nothing (defect 4.1). | A caption asks for a click that has no action. |
| Refusing to publish: '`<widget>`' requires a selection from `<names>`; one row click can never satisfy all of them. Keep requiresSelection true on the row source only. | A widget needs several selections at once. |
| Refusing to publish: '`<widget>`' caps its query at `<number>` rows without saying so in its title (no silent caps). | A query limits its rows without telling the reader. |
| Refusing to publish: '`<widget>`' has no help text. Add it to HELP. | A widget has no help text. |
| Refusing to publish: '`<widget>`' indicator Arcade must return a flat object with middleText; the list contract renders blank. | An indicator script returns the wrong shape. |
| Refusing to publish: '`<widget>`' reads Arcade data source `<id>` with no refreshInterval; the heartbeat would never age. | A script source never refreshes. |
| Refusing to publish: widget data sources name item(s) `<ids>`, not the configured `<id>`. | A widget reads an item other than the configured one. |
| Refusing to publish: Arcade source `<name>` does not read the configured item `<id>`. | A script reads an item other than the configured one. |

The last message depends on your config.

> Refusing to publish: a widget reads `<table>`.`<field>`, which the view hides. Turn the privacy switch off or remove the widget's read.

**Cause.** A `privacy` switch hides a field in the read-only view, and a widget needs that field.

**Fix.** If you edited `dashboard_templates.json`, remove the read of that field. Otherwise remove
identifying values from the message, as the rule in [Issues](../README.md#issues) says, and send it
to the TaskDash maintainer. Turn the privacy switch off only if you accept that viewers
see the field.

## Demo

These messages come from `taskdash demo`. The command builds synthetic data in a `demo` folder. That
folder is inside the install folder, or inside the source checkout if you run the demo from one.
With `--apply`, the demo also publishes that data to separate `DEMO_` items in the `TaskDash_Demo`
folder in ArcGIS Online.

### The demo needs a real sign-in

> --apply signs in only through the real config's arcgis.url and arcgis.profile, and one of them is empty. A username or password is never copied to the demo. Store a profile as INSTALL.md step 3 describes, set arcgis.profile, then re-run `taskdash demo --apply`.

**Cause.** The demo signs in with `arcgis.url` and `arcgis.profile` from the real config. One of them
is empty, or the real config does not exist. A config that signs in with `arcgis.username` and a
password also gets this message, because the demo never copies a password.

**Fix.** Set both keys as steps 2 and 3 of [INSTALL.md](INSTALL.md) describe. Then run
`taskdash demo --apply` again.

### The demo and the real install share an item

> The demo config uses item id(s) `<ids>` that the real config also uses, so publishing would overwrite real data. Delete the demo folder's TaskDash.config.json and run `taskdash demo --apply` again.

**Cause.** The config in the `demo` folder holds an item ID that the real config also holds. The
demo would write synthetic data into your real service.

**Fix.** Delete `TaskDash.config.json` in the `demo` folder. Run `taskdash demo --apply` again. The
demo then creates its own `DEMO_` items.

### The demo install is incomplete

> The demo config lacks arcgis.`<keys>`. A failed demo setup left a partial install. Delete the DEMO_ items in the TaskDash_Demo folder and `<path>`, then re-run `taskdash demo --apply`.

**Cause.** The config in the `demo` folder holds some of `arcgis.item_id`, `arcgis.view_item_id` and
`arcgis.dashboard_id`, but not all three. An earlier `taskdash demo --apply` stopped during setup.

**Fix.**

1. In ArcGIS Online, delete the `DEMO_` items, the `DEMO_TaskDash viewers` group and the
   `TaskDash_Demo` folder. Use the order of Undo step 2 in [INSTALL.md](INSTALL.md).
2. Delete the file that the message names. It is `TaskDash.config.json` in the `demo` folder.
3. Run `taskdash demo --apply` again.

### The demo collect run failed

> The demo collect run exited with code `<code>`. Read the newest log in `<folder>`, fix the cause, then re-run `taskdash demo`.

**Cause.** A collect run on the invented data did not finish. The demo stops so that it does not
report data it never published.

**Fix.** Open the newest log file in the `Logs` folder inside the `demo` folder. Look up its last
error in this guide, fix the cause, and run the same `taskdash demo` command again.

### Demo ends with exit code 0

Exit code 0 is not an error. The demo gives 0 when the check or the publish finished.

## Status

These messages come from `taskdash status`. Without `--offline`, the command signs in and reads the
newest row of the `CollectorRuns` table.

### Status has no service to read

> No arcgis.`<item_id or test_item_id>` in the config, so there is no `<target>` service to read. Set it, or use `taskdash status --offline`.

**Cause.** The config key for the service is empty. `--target prod`, the default, reads
`arcgis.item_id`. `--target test` reads `arcgis.test_item_id`.

**Fix.** Set the key, or run `taskdash setup --apply` for a new install. To read only the local log,
run `taskdash status --offline`.

### Status cannot find the service item

> Service item `<id>` was not found or is not visible to this account.

**Cause.** The item ID in the config does not match an item that the signed-in account may see.

**Fix.** Open the item in ArcGIS Online and compare its ID with the config. Check the stored
sign-in.

### The service has no CollectorRuns table

> The service has no CollectorRuns table. Run `taskdash collect --migrate --apply` to create it.

**Cause.** The service does not have the table that records collector runs.

**Fix.** Run `taskdash collect --migrate --apply`.

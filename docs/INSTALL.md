# Install TaskDash

TaskDash watches the scheduled tasks on one Windows computer. Every hour it publishes their health,
run history and log metrics to ArcGIS Online. A dashboard shows the result.

This guide takes you from nothing to a working dashboard. Plan for about one hour. Run every step
on the computer whose tasks you want to watch, at its console or through Remote Desktop. A
PowerShell remoting session (`Enter-PSSession`) does not work: Windows gives it no access to the
saved sign-in, and TaskDash fails with "A specified logon session does not exist".

## Requirements

- **Windows:** Windows 10, Windows 11 or Windows Server 2016 or later.
- **Your Windows account:** sign in with the account you use every day. Steps 1 to 6 and the
  checks in step 8 run in a normal PowerShell window as this account. You also need administrator rights on
  the computer, for the event-log fix in step 5, steps 7 and 8, and Undo step 1. If your
  administrator rights come from a separate admin account, use it only where the guide says
  "as administrator".
- **Python:** 64-bit Python 3.11 from python.org, the tested version, **installed for all users**.
  The last 3.11 release with a Windows installer is 3.11.9.
  In the installer, select **Customize installation**, keep **py launcher** selected, and on the
  next page select **Install Python 3.11 for all users**. The task account runs this Python, and it
  cannot run a Python installed in your own user profile. To check, run `py -0p`. The path for 3.11
  must be under `C:\Program Files`, not under `C:\Users`.
- **Internet access** from the computer over HTTPS:
  - to `pypi.org` and `files.pythonhosted.org`, for step 1;
  - to `www.arcgis.com`, your organization's `<org>.maps.arcgis.com`, and `*.arcgis.com`, in steps
    3 to 8 and every hour after that.

  If the computer reaches the internet through a proxy, run this once in an administrator window
  before step 1, with your proxy's address:

  ```powershell
  [Environment]::SetEnvironmentVariable("HTTPS_PROXY", "http://<proxy>:<port>", "Machine")
  ```

  Then close every PowerShell window and open new ones, because an open window does not see the
  new value. If the first run in step 8 still cannot reach ArcGIS Online, restart the computer, so
  that Task Scheduler sees the new value too.
- **ArcGIS Online:** a built-in ArcGIS account (a user name and a password) without multifactor
  authentication. It must be able to create items, create groups, share to groups and edit features.
  A Creator user type has these privileges, and step 5 checks them. An account that signs in only
  through your organization's single sign-on (SAML), or that requires multifactor authentication,
  cannot be used. Ask your ArcGIS administrator for a suitable account.
- **A Windows account for the task.** The collector runs as this account. It must be a member of
  the local Administrators group on this computer. It needs a password, because the task stores it.
  This guide calls it `DOMAIN\svc-taskdash`. Use your own account name everywhere you see it. For a
  local account, write `COMPUTERNAME\name`, for example `TASKHOST1\svc-taskdash`.
- **The TaskDash wheel:** from <https://github.com/uhsear/taskdash-releases/releases/latest>, download the
  file `taskdash-<version>-py3-none-any.whl` into your `Downloads` folder. Keep only one TaskDash
  wheel in that folder. If this computer cannot reach GitHub, download the file on another computer
  and copy it into `Downloads`. The release page is public, so you need no GitHub account.
  - The release also lists `SHA256SUMS.txt`. It has one line each for the wheel, `requirements.lock`
    and the `.cdx.json` SBOM. To check the download, compare its value with the
    output of `Get-FileHash $env:USERPROFILE\Downloads\taskdash-*.whl`.
  - Optional: check who built the files. The release lists a `.sigstore.json` bundle for the wheel
    and for `SHA256SUMS.txt`. Download both bundles into `Downloads`. In a separate virtual
    environment, install the checker with `python -m pip install sigstore`. Then run this command
    for each file. Replace `<tag>` with the release tag, for example `v1.1.0`, and `<file>` with the
    file name:

    ```powershell
    python -m sigstore verify github --cert-identity https://github.com/uhsear/taskdash/.github/workflows/release.yml@refs/tags/<tag> --bundle <file>.sigstore.json <file>
    ```

    The command prints `OK: <file>` when the release workflow of the TaskDash repository signed
    the file. Any other output means you must not install the file. Ask the TaskDash maintainer.

TaskDash is tested on one computer with 29 tasks that run at most hourly. Larger installs may work.

## 1. Install TaskDash

Open a normal PowerShell window as your daily account. Run these commands. They create
`C:\TaskDash`, lock its permissions, make a Python virtual environment inside it, and install the
wheel from your `Downloads` folder. In the `icacls` line, replace `DOMAIN\svc-taskdash` with the
task account.

```powershell
New-Item -ItemType Directory -Force C:\TaskDash | Out-Null
icacls C:\TaskDash /inheritance:r /grant:r "*S-1-5-18:(OI)(CI)F" "*S-1-5-32-544:(OI)(CI)F" "${env:USERDOMAIN}\${env:USERNAME}:(OI)(CI)F" "DOMAIN\svc-taskdash:(OI)(CI)M"
py -3.11 -m venv C:\TaskDash\venv
$whl = (Get-ChildItem "$env:USERPROFILE\Downloads\taskdash-*.whl" | Select-Object -First 1).FullName
$whl
C:\TaskDash\venv\Scripts\python.exe -m pip install $whl
C:\TaskDash\venv\Scripts\taskdash.exe --version
```

The `icacls` line prints `Successfully processed 1 files; Failed processing 0 files`. A new folder
under `C:\` lets every signed-in user change its files, and TaskDash runs from this folder every
hour as an administrator. The line removes those inherited permissions. It allows only SYSTEM, the
Administrators group, you and the task account. Step 4 checks the result.

The `$whl` line prints the full path of the wheel. If it prints nothing, the wheel is not in
`Downloads`. Then set the path yourself, for example `$whl = "D:\Files\taskdash-1.1.0-py3-none-any.whl"`.
Keep the original file name; pip rejects a renamed wheel, such as one with ` (1)` in its name.

If you want e-mail alerts that sign in to a mail server, install the `smtp` extra instead. It adds
the `keyring` package. Run this in place of the `pip install $whl` line:

```powershell
C:\TaskDash\venv\Scripts\python.exe -m pip install "$whl[smtp]"
```

You can also add it later by running that line again. Alerts without a mail sign-in need no extra.

### Optional: locked install

The release also lists `requirements.lock`. It pins every package that TaskDash needs, with a
SHA-256 hash for each file, including `keyring` from the `smtp` extra. Use it if you want pip to
refuse any package that differs from the one the release was built with. The lock is for Python 3.11
64-bit on Windows. On any other Python, use the normal install above.

1. Download `requirements.lock` into `Downloads`.
2. Compare its SHA-256 value with the line in `SHA256SUMS.txt`, as you did for the wheel.
3. Check the wheel the same way. Do not install a file that does not match.
4. Run these commands in place of the `pip install $whl` line:

```powershell
C:\TaskDash\venv\Scripts\python.exe -m pip install --require-hashes -r "$env:USERPROFILE\Downloads\requirements.lock"
C:\TaskDash\venv\Scripts\python.exe -m pip install --no-deps $whl
```

The first command stops with an error if a downloaded file does not match its hash. The second
command installs only the wheel, because the first command already installed its dependencies.
The release also lists a CycloneDX software bill of materials, `taskdash-<version>.cdx.json`. It
lists the same packages. Keep it if your organization asks for one.

The install takes a few minutes. It ends with a line that starts `Successfully installed` and
names `taskdash-<version>`. Lines that start `[notice] A new release of pip is available` are
harmless; you do not need to upgrade pip. The last command prints `TaskDash <version>`, for example
`TaskDash 1.1.0`.

TaskDash keeps its config, logs and state in the folder it starts in. **Run these two lines in every
new PowerShell window before you use `$td`:**

```powershell
Set-Location C:\TaskDash
$td = "C:\TaskDash\venv\Scripts\taskdash.exe"
```

## 2. Create the config

Copy the example config into `C:\TaskDash` and open it:

```powershell
Copy-Item C:\TaskDash\venv\share\taskdash\TaskDash.config.example.json C:\TaskDash\TaskDash.config.json
notepad C:\TaskDash\TaskDash.config.json
```

The `_notes` section in the file explains every key. Change only these keys:

| Key | Set it to |
|---|---|
| `arcgis.url` | Your organization's URL, such as `https://myorg.maps.arcgis.com`. `https://www.arcgis.com` also works. Step 3 must use the same URL. |
| `arcgis.profile` | Keep `taskdash`. Step 3 stores a sign-in under this name. |
| `host_name_override` | `""`, so TaskDash uses this computer's name |
| `alerting.enabled` | Keep `false`. Set up e-mail alerts after the install works. |

Keep `arcgis.item_id`, `arcgis.view_item_id` and `arcgis.dashboard_id` empty (`""`). Step 6 fills them.
Keep the `privacy` switches on. They hide command lines, accounts, paths and log samples from viewers.

The other defaults are safe. For example, `scheduler.exclude_microsoft_tasks` is `true`, so the
built-in tasks under `\Microsoft\` are not monitored.

Save the file. Then check it:

```powershell
& $td preflight --offline
```

The first line must be `[ok] config: config loaded, no warnings`. If it is `[!!] config: ...`, the
line names each key to fix. A JSON syntax error names the line and column; a missing comma is the
usual cause. The other lines do not matter yet.

## 3. Store the ArcGIS sign-in

TaskDash never keeps your ArcGIS password in the config. It reads a saved sign-in, called a profile.
The password goes into Windows Credential Manager. Each Windows account has its own Credential
Manager, so you store the profile twice: once for yourself, and once for the task account. Use the
same ArcGIS account both times.

**For yourself,** in your normal PowerShell window, run the create command. Replace `ARCGIS_URL`
with your `arcgis.url` and `ARCGIS_USER` with the ArcGIS account name. Type the ArcGIS password at
the `Password:` prompt; it does not echo.

```powershell
C:\TaskDash\venv\Scripts\python.exe -c "import getpass; from arcgis.gis import ProfileManager; ProfileManager().create('taskdash', 'ARCGIS_URL', 'ARCGIS_USER', getpass.getpass())"
```

Then run the check command. It prints the ArcGIS account name:

```powershell
C:\TaskDash\venv\Scripts\python.exe -c "from arcgis.gis import GIS; print(GIS(profile='taskdash').users.me.username)"
```

If the check fails or prints `None`, the URL, the user name or the password is wrong. Run the create
command again with the right values; it replaces the profile.

**For the task account,** open a PowerShell window as that account. Windows asks for its password:

```powershell
runas /user:DOMAIN\svc-taskdash powershell.exe
```

In the new window, run the create command and the check command above. If `runas` says the account
cannot log on here, the account lacks the "Allow log on locally" right on this computer, or a Group
Policy sets "Deny log on locally" for it. A "Deny" setting always wins over "Allow". Run
`gpresult /h C:\gp.html` and look under **User Rights Assignment** to see which applies. Ask your
domain administrator for a temporary exception for the task account while you run the two commands.
Close the window when you are done.

## 4. Check the folder permissions

The task writes logs and state into `C:\TaskDash`, and runs the code in it. In your normal window,
run:

```powershell
icacls C:\TaskDash
```

The output must list exactly four entries, each with `(OI)(CI)`: your account and
`NT AUTHORITY\SYSTEM` and `BUILTIN\Administrators` with `(F)`, and the task account with `(M)`.
If it lists `Users` or `Authenticated Users`, or the task account is missing, run the `icacls` line
from step 1 again with the right task account name.

## 5. Check the computer

In your normal window, run the preflight check. It changes nothing.

```powershell
& $td preflight
```

Each line starts with a mark:

- `[ok]` means the check passed.
- `[!!]` means a problem. The line ends with the fix.
- `[--]` means the check does not apply to your config, such as `smtp` while alerting is off.

The last line counts the problems, such as `Preflight: 3 problem(s), 0 warning(s).` At this point
these three lines are expected, because steps 6 and 7 have not run yet:

- `[!!] service: SystemExit: No arcgis.item_id in the config. A new install runs ...`
- `[!!] sharing: the service was not read ...`
- `[!!] collector logon type: collector task 'TaskDash Collector' not found. ...`

`[--] view: no arcgis.view_item_id configured ...` and
`[--] collector account: collector task 'TaskDash Collector' is not registered yet` are also expected.

A `health check` line that starts with `WARNING` means `alerting.healthcheck_url` is empty. It is not
a failure, and preflight still exits 0. See "Set up an outside health check" after step 8.

A `tested scale` line that starts with `WARNING` is not a problem. It says that this computer has
more tasks than the tested install, or tasks that repeat more often than hourly.

### The event log line

TaskDash reads each task's run history from the Windows event log
`Microsoft-Windows-TaskScheduler/Operational`. Windows often ships this log turned off, or too small
to hold more than a few days. If the `event log` line is `[!!]`, fix it:

1. Open PowerShell **as administrator** (right-click PowerShell, then **Run as administrator**).
2. Run the two lines from step 1 that set the folder and `$td`.
3. Run `& $td preflight --enable-event-log --apply`.
4. Check that the last line it prints says `enabled=true` and a max size of 67108864 bytes (64 MB) or more.
5. Close the administrator window. In your normal window, run `& $td preflight` again. The
   `event log` line is now `[ok]`.

Run history starts at the moment the log is turned on. Runs before that do not appear.

Fix every other `[!!]` line before you continue.

## 6. Create the ArcGIS items

In your normal window, first run setup without `--apply`. It signs in, checks that no name is taken,
and prints its plan.

```powershell
& $td setup
```

The plan lines start with `Plan:`. The last line is `DRY RUN - nothing written. Re-run with --apply.`
If setup stops with a message instead, the message names the clash and the fix.

Then create the items:

```powershell
& $td setup --apply
```

Setup does the following:

1. It backs up the config to a file named like `TaskDash.config.json.bak_setup_20261017_202238`.
2. It creates a folder named `TaskDash` in your ArcGIS content.
3. It creates the hosted service `TaskDash`, its read-only view `TaskDash_view`, a private group
   `TaskDash viewers` and the dashboard `TaskDash`.
4. It writes the three item IDs into the config. Each one prints as `Config: arcgis.<key> = <id>`.

The end of the output looks like this:

```text
Open: https://myorg.maps.arcgis.com/apps/dashboards/<dashboard id>
Created by this run (delete these in ArcGIS Online to undo):
    Folder           <id>  TaskDash
    Feature Service  <id>  TaskDash
    Feature Service  <id>  TaskDash_view
    Group            <id>  TaskDash viewers
    Dashboard        <id>  TaskDash
Setup complete. The service is private; the view and dashboard are shared with the group 'TaskDash viewers' only. Add members to that group to give them access.
```

Keep the `Open:` link. Step 9 uses it.

To use other names, pass `--folder`, `--title-prefix` or `--group-title`.

If setup stops partway, it still lists what it created. To recover:

1. In ArcGIS Online, delete only the items in that list, in the order of Undo step 2.
2. Copy the newest `TaskDash.config.json.bak_setup_*` file over `TaskDash.config.json`.
3. Run `& $td setup --apply` again.

## 7. Register the hourly task

Open PowerShell **as administrator**. Run the two lines from step 1, then run the installer in check
mode. `-Python` names the `taskdash.exe` that the task runs.

```powershell
powershell.exe -ExecutionPolicy Bypass -File C:\TaskDash\venv\share\taskdash\install_task.ps1 `
    -InstallDir C:\TaskDash -Python C:\TaskDash\venv\Scripts\taskdash.exe -User DOMAIN\svc-taskdash
```

The TaskDash PowerShell scripts are not signed. If Group Policy sets the execution policy to
`AllSigned`, `-ExecutionPolicy Bypass` cannot override it and PowerShell refuses the script. Ask
your IT department for an exception for this one file.

It registers nothing. It prints the task it would register:

```text
Task name:         TaskDash Collector
Runs as:           DOMAIN\svc-taskdash (Password)
Command:           C:\TaskDash\venv\Scripts\taskdash.exe collect --apply
Working directory: C:\TaskDash
Trigger:           every 1 hour from <next full hour>, no end
Limits:            2 hour run limit, no second instance, catch up a missed run
Needs:             DOMAIN\svc-taskdash in the local Administrators group; a standard account misses other accounts' tasks
CHECK MODE - nothing registered. Re-run with -Apply.
```

Check that `Runs as` names your task account and that the working directory is `C:\TaskDash`. Then
run the same command with ` -Apply` added at the end. The script asks for the task account's
password and prints `Registered 'TaskDash Collector'. Principal.LogonType = Password`.

The script uses the Password logon type. It refuses S4U ("Do not store password"), because an S4U
task cannot read the saved sign-in from step 3.

### Why the task account must be an administrator

- **It must see every task.** Windows shows a standard account only some tasks. In a test, a
  standard account saw 170 of 229 tasks: none that other accounts had registered. Those tasks would
  be missing from the dashboard, and nothing would report an error.
- **It must log on as a batch job.** A task with a stored password logs on this way. By default,
  only Administrators, Backup Operators and Performance Log Users hold the right. Without it, the
  task never starts. `LastTaskResult` stays `267011` (0x41303, "has not yet run"), and Task
  Scheduler logs event 104 with error value `2147943785`.

To check the membership, run `net localgroup Administrators` in the administrator window. The list
must name the task account, or a domain group that contains it. To add the account, run
`net localgroup Administrators DOMAIN\svc-taskdash /add`.

## 8. Run it once and check

In the administrator window, start the task now instead of waiting for the hour:

```powershell
Start-ScheduledTask -TaskName "TaskDash Collector"
```

The first run takes one to five minutes. Repeat these two commands until the state is `Ready`:

```powershell
(Get-ScheduledTask -TaskName "TaskDash Collector").State
(Get-ScheduledTaskInfo -TaskName "TaskDash Collector").LastTaskResult
```

What `LastTaskResult` means:

- `267009` (0x41301): the task is still running. Wait and repeat.
- `0`: the run succeeded.
- `1`: TaskDash ran and stopped with an error. The newest file in `C:\TaskDash\Logs` names it in
  its last line.
- `267011` (0x41303): the task never started. See "Why the task account must be an
  administrator" in step 7.
- `2147943726` (0x8007052E): Windows rejected the task account's password. Delete the task with
  `Unregister-ScheduledTask -TaskName "TaskDash Collector" -Confirm:$false`, then run step 7 with
  `-Apply` again and type the right password.
- Any other value: read the task's **History** tab in Task Scheduler, as described below.

If the state is still `Running` after 10 minutes, open the newest log file in `C:\TaskDash\Logs`
and read its last lines. If no log file has today's time, TaskDash did not start. Then open Task
Scheduler, select the task, and read the **History** tab. Event 101, 103 or 104 names the reason,
such as a wrong password.

When the run succeeded, check the result **in your normal window**:

```powershell
& $td status
& $td preflight
```

`status` prints two lines like these:

```text
Local: taskdash_20261017_202547.log: <time> - INFO - TaskDash publish complete.
Service: last collector run started 2026-10-17 20:25:47 UTC, exit_status success
```

Preflight now shows `[ok]` for the service, the view, sharing, the collector logon type and the
collector account. The collector account line checks that the task account is in the local
Administrators group. If that line starts `[!!] collector account: WARNING:`, the account is not a
direct member, and preflight names the groups that may include it. Confirm that one of them does.

If the collector logon type line still says `collector task 'TaskDash Collector' not found`, your
normal window may not be allowed to see a task that runs as another account. Run
`& $td preflight --offline` in the administrator window. That line must be
`[ok] collector logon type: task 'TaskDash Collector' logs on as Password`.

### Set up an outside health check

If the collector never starts, nothing inside TaskDash can tell you. An expired password or a
disabled task causes this. Only an outside monitor notices. Create a check at a service such as
Healthchecks.io, then put its ping address in `alerting.healthcheck_url`. The
"Watch for a silent collector" section of `docs/CONFIGURE.md` explains the pings. The address holds a
secret ID, so keep it out of tickets and chat. Run `& $td preflight`. The `health check` line then
says `alerting.healthcheck_url is set` and does not show the address.

## 9. Open the dashboard and add viewers

Open the `Open:` link from step 6. To find the dashboard again, open ArcGIS Online, select
**Content**, then open the `TaskDash` folder. The first run fills the health tables. Run history and
trends fill in over the next hours.

To give people access, add them to the `TaskDash viewers` group. In ArcGIS Online, select
**Groups**, open `TaskDash viewers`, and select **Invite members**. Members can open the dashboard
and read the view. Nobody else can, and nobody can edit the data through the view.

## Undo the install

1. In PowerShell as administrator, run
   `Unregister-ScheduledTask -TaskName "TaskDash Collector" -Confirm:$false`.
2. In ArcGIS Online, delete in this order: the `TaskDash` dashboard, the `TaskDash_view` view, the
   `TaskDash` service, the `TaskDash viewers` group, and then the empty `TaskDash` folder. Delete
   the view before the service, because the view reads the service's data.
3. Remove the saved sign-in for each Windows account that stored one in step 3. Run this once in
   your normal window, and once in a `runas` window as the task account:

   ```powershell
   C:\TaskDash\venv\Scripts\python.exe -c "from arcgis.gis import ProfileManager; ProfileManager().delete('taskdash')"
   ```

4. Close every window that is in `C:\TaskDash`. Then delete the `C:\TaskDash` folder.

The undo leaves three changes in place, because other software may use them:

- The larger, enabled Task Scheduler event log from step 5. It is harmless to keep.
- The task account's membership of the Administrators group. Remove it if nothing else needs it.
- Python. Uninstall it in **Settings > Apps** if nothing else needs it.

## Use a cloned ArcGIS Pro environment instead

If the computer has ArcGIS Pro, you can install TaskDash into a clone of Pro's Python environment
instead of the virtual environment of step 1. Pro already contains `arcgis` and `keyring`. In
step 1, run these commands in place of the `py -3.11 -m venv` and `pip install` lines. The clone
takes a few minutes and about 2 GB.

```powershell
& "C:\Program Files\ArcGIS\Pro\bin\Python\Scripts\conda.exe" create --clone arcgispro-py3 -p C:\TaskDash\proenv --yes
Test-Path C:\TaskDash\proenv\python.exe
C:\TaskDash\proenv\python.exe -m pip install --no-deps $whl
C:\TaskDash\proenv\Scripts\taskdash.exe --version
```

- `Test-Path` must print `True`. A clone into a deep folder path can stop partway and still
  report success. `C:\TaskDash\proenv` is short enough, and the task account can read it.
- Use `--no-deps`. Without it, pip replaces packages that conda manages in the clone, such as
  `cffi` and `pywin32`, and the clone can break.
- In the `$td` line of step 1, use `C:\TaskDash\proenv\Scripts\taskdash.exe` in place of
  `C:\TaskDash\venv\Scripts\taskdash.exe`. Where a later step names
  `C:\TaskDash\venv\Scripts\python.exe`, use `C:\TaskDash\proenv\python.exe`. The `$whl` line needs
  no change. In step 7, pass the same `taskdash.exe` as `-Python`.

## Known limits

- One computer per install is recommended.
- The dashboard shows run times in the time zone of the viewer's device.
- ArcGIS Online only. ArcGIS Enterprise is not tested.
- Scrubbing secrets from command lines and log samples is best effort. Keep the privacy switches on.

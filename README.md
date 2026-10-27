# TaskDash

TaskDash is a monitor for the Windows scheduled tasks on one computer. Every hour it reads the Task
Scheduler, the run history and each task's log files. It publishes the health, history and log
metrics of every task to hosted tables in ArcGIS Online. A dashboard shows the result. Optional
e-mail alerts tell you when a task fails or is overdue.

Not affiliated with, endorsed by or supported by Esri.

## Features

- Shows the state of every scheduled task: success, running, warning, error, disabled or never run.
- Detects overdue tasks, stuck tasks, and tasks that changed.
- Keeps the run history from the Windows event log, and mines each task's log files for counts,
  durations and errors.
- Publishes to five hosted tables. The dashboard reads a query-only view of them. The editable
  service stays owner only.
- Sends optional e-mail alerts, including one e-mail that names the likely shared cause when
  several tasks fail together.
- Hides command lines, account names, paths and log samples by default. Removes secrets from the
  free text it publishes.
- Installs as a Python wheel. One command creates the ArcGIS items, and a preflight check tells you
  what is missing before the first run.
- Writes nothing to ArcGIS Online and sends no e-mail unless you pass `--apply`.
- Includes `taskdash demo`, which shows the dashboard on invented data before you point it at
  real tasks. See [Try the demo](docs/USE.md#try-the-demo).

## Instructions

1. Read [docs/INTRODUCTION.md](docs/INTRODUCTION.md) for what TaskDash does, what it creates and its
   known limits.
2. Follow [docs/INSTALL.md](docs/INSTALL.md) to install it. Plan for about one hour.
3. Use [docs/CONFIGURE.md](docs/CONFIGURE.md) to change a setting.
4. Use [docs/USE.md](docs/USE.md) to read the dashboard and the alert e-mails.
5. If something fails, look up the message in [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md).

## Requirements

- Windows 10, Windows 11 or Windows Server 2016 or later, with PowerShell.
- 64-bit Python 3.11 installed for all users. The package needs Python 3.9 or later; 3.11 is the
  tested version.
- The `arcgis` package, version 2.4 or later and earlier than 3. The wheel installs it.
- An ArcGIS Online built-in account of the Creator user type, without multifactor authentication.
- A Windows account for the hourly task. It must belong to the local Administrators group.
- ArcGIS Pro only if the tasks you monitor run arcpy.

TaskDash is tested on one computer with 29 tasks that run at most hourly. ArcGIS Enterprise is not
tested yet. [docs/INTRODUCTION.md](docs/INTRODUCTION.md) lists the details and all known limits.

## Resources

- [Install guide](docs/INSTALL.md)
- [Introduction](docs/INTRODUCTION.md)
- [Configuration reference](docs/CONFIGURE.md)
- [Use guide](docs/USE.md)
- [Troubleshooting](docs/TROUBLESHOOTING.md)
- [Security policy](SECURITY.md)
- [Changelog](CHANGELOG.md)

## Issues

Release files are public at <https://github.com/uhsear/taskdash-releases/releases/latest>. To report
a problem or ask for a change, contact the repository owner through GitHub. For a vulnerability,
do not open an issue. Follow [SECURITY.md](SECURITY.md).

Follow this rule for every report. Before you send a message or a log line, remove host names,
account names, paths and other values that identify your site. Never send a captured fixture file
or the config file. Never include secrets.

## Licensing

Copyright 2026 Asir Khan.

TaskDash is licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for the terms and
[NOTICE](NOTICE) for the attribution notice.

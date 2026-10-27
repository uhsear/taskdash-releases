# Use TaskDash

TaskDash publishes the health of your scheduled tasks to an ArcGIS Online dashboard once an hour.
This guide shows two ways to read it. The first part is the daily check of the GIS administrator.
The second part is the view of a manager who asks how reliable the tasks are over time.

This guide assumes that the install in [INSTALL.md](INSTALL.md) works. Two other guides go with it:

- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) explains the error messages and how to fix them.
- [CONFIGURE.md](CONFIGURE.md) explains every config key, such as the thresholds and the retention days.

## Try the demo

The demo shows a dashboard on invented data. Run the commands from the TaskDash folder.

- `& $td demo` runs offline. It writes only to the `demo` folder inside the TaskDash folder.
- `& $td demo --apply` needs a finished `taskdash setup`. It creates separate `DEMO_` items in an
  ArcGIS Online folder named `TaskDash_Demo`. It does not change your real items.

## Open the dashboard

Open the link that `taskdash setup --apply` printed after `Open:`. To find the dashboard again,
open ArcGIS Online, select **Content**, then open the `TaskDash` folder. The dashboard is shared
only with the `TaskDash viewers` group.

The dashboard has three tabs:

| Tab | What it shows |
|---|---|
| **Overview** | The health tiles, the Task health table, Task detail, failed runs and two charts. |
| **Workflow Metrics** | The numbers that the task scripts write to their logs, such as record counts. |
| **Collector & Changes** | The collector's own runs, the recent changes to tasks and the setting notes. |

The **Filters** panel is at the left edge. It has five pickers: **Host**, **Task**,
**Status (Overview and Setting notes)**, **Schedule (Overview and Setting notes)** and
**Metric (Workflow Metrics)**. A picker changes the tables, lists and charts that its label names.
It never changes the tiles, so a filter cannot hide a problem from the tiles.

Each widget has an information icon. It opens a short help text. The text tells you what the widget
shows and why it matters. For most widgets, it also tells you what to do.

The dashboard shows run times in the time zone of your device. It shows schedules by the server's
clock. The heartbeat tiles and **Metric change** update while the page is open. The other widgets
do not. To see newer data in them, reload the page.

## Part 1: The daily check for the administrator

The check takes about five minutes. Do the steps in this order. Step 1 tells you whether the other
numbers are current.

### 1. Check that the data is current

Read the **Collector heartbeat** tile first. It shows how long ago the collector last saved data.
The collector runs every hour by default.

| Tile state | Time since the last save | Meaning |
|---|---|---|
| **OK** | Less than 90 minutes | The collector runs as planned. |
| **Late** | 90 to 180 minutes | The collector missed one run. |
| **Down** | More than 180 minutes | The collector missed two runs or more. |

The limits are 1.5 and 3 times `scheduler.collector_interval_minutes`. If the tile shows **Down**
with the text `no data`, the service has no saved data yet. With more than one host, the tile shows
the host that saved data longest ago.

If the tile shows **Late** or **Down**, every other number on the dashboard is old. Go to step 6
and find out why the collector stopped.

### 2. Read the tiles

The eight tiles on the **Overview** tab count tasks. They ignore every filter.

| Tile | What it counts |
|---|---|
| **Needs attention** | Enabled tasks with a problem now. The word **Open** appears under a non-zero number. |
| **Failed (last run)** | Enabled tasks whose last run failed. |
| **Overdue** | Tasks that did not start within the grace time after their planned start. |
| **Other warnings** | Tasks with another problem: a stuck run, a blocked task or a Task Scheduler warning. |
| **Collector heartbeat** | The time since the collector last saved data. See step 1. |
| **Lowest 7-day success** | The lowest success rate of any task that failed twice or more in 7 days. |
| **Disabled** | Tasks that are off in Windows Task Scheduler. |
| **Tasks tracked (all)** | All tasks that the collector monitors. |

**Failed (last run)**, **Overdue** and **Other warnings** add up to **Needs attention**. Each task
counts in one of the three tiles only. A task that is both failed and overdue counts as failed.

Read the three parts as follows:

- **Failed (last run)** is the most urgent. Repair these tasks first.
- **Overdue** means a task did not start on time, also when its last run was OK. The grace time is
  15 minutes. To change it for all tasks, set `scheduler.overdue_grace_minutes`.
- **Other warnings** counts tasks, not runs. These tasks need a check, but no run failed. A task is
  here for one of these reasons:
  - **Stuck:** the task is running for longer than 30 minutes and longer than three times its usual
    long run. A run longer than the task's own time limit is also stuck.
  - **Blocked:** Task Scheduler skipped two or more new runs in 24 hours, because an earlier run was
    still going.
  - **Warning:** Task Scheduler reports a missed run or a warning result code. A task with no next
    run time, when its schedule needs one, is also here.

A **Disabled** task never shows as failed. If a disabled task must run, enable it in Task Scheduler.

If **Tasks tracked (all)** falls and you did not remove a task, open the **Collector & Changes**
tab. Look for **Task removed** rows in **Recent changes - newest 50**.

**Lowest 7-day success** shows **OK** at 90% or more, **Low** from 70% up to 90%, and **Poor** below
70%. The tile ignores a task with only one failure. The **Most failures in 7 days - top 10** chart
still shows that task. With no such task, the tile shows `100.0%`. If no task ran in 7 days, the tile
shows `--`.

If the tile shows **Low** or **Poor**, look for the task in two places. One is the
**Most failures in 7 days - top 10** chart. The other is the **Failures (7 days)** column of the
**Task health** table.

### 3. Read the Task health table

The **Task health** table is the full work list, one row for each task. Tasks with a problem are at
the top. Failed tasks come first, then overdue tasks, then other warnings, then disabled tasks.

| Column | Meaning |
|---|---|
| **Status** | The problem kind, or the health of a task with no problem. |
| **Task** | The task name. |
| **Why** | The reason for the status. |
| **Last run** | When the task last ran. |
| **Next run** | When it runs next. A disabled task shows `Off`. |
| **Failures (7 days)** | The failed runs in the last 7 days. |
| **Host** | The computer that runs the task. |

If the next run time of a task with no problem is already past, **Next run** shows its schedule
group, for example `Daily`. The next collector run updates the time.

A row with a problem has a colored background. The **Status** column shows a word and a symbol.
A reader who cannot see the color still gets the state.

| Status word | Meaning |
|---|---|
| **Failed** | The last run failed. |
| **Overdue** | The task did not start on time. |
| **Stuck** | The task has run far longer than usual, or longer than its time limit. |
| **Blocked** | Task Scheduler skips new runs, because an earlier run is still going. |
| **Warning** | Task Scheduler reports a missed run, a warning result or no next run time. |
| **OK** | The last run succeeded. |
| **Running** | The task is running now. |
| **Not yet run** | The task has not run yet. |
| **Disabled** | The task is off in Task Scheduler. |

A row with no problem can still show text in **Why**:

- `N log errors` in amber: the newest log of the task holds error lines. The status stays **OK**
  and the tiles do not change, because Windows reported success.
- `N setting notes`: the task has a risky setting. See step 5.
- `Running now` or `Disabled in Task Scheduler.`

A long reason is cut at a word. The text `(+2 more)` at the end counts the other reasons. Task
detail shows all of them.

### 4. Find out why a task failed

1. In the **Task health** table, select the row of the task. The row fills **Task detail**, the
   **Failed runs** list and the **Duration trend** chart.
2. In **Task detail**, read the line that starts with **Why:**. It gives the reason. Then read
   these lines:
   - **Last run:** the time, the Windows result text and how long the run took.
   - **Last 7 days:** the runs, the failures and the skipped runs.
   - **Runs as:** the account and how it logs on.
   - **Script:** the program and its arguments.
   - **Latest log:** the age of the newest log, and its number of error and warning lines.
     The full path of the log is on the next line.
3. In **Failed runs - newest 25**, read the cards of the task. A card gives the status word, the
   start time, the age, the duration and the Windows result text. A pink card is a failure from the
   last 7 days. A white card is older. A failure card can show one of these words: **Failed**,
   **Did not start**, **Account not signed in**, **Stopped at time limit** or **Terminated**.
4. On the computer that runs the task, open the newest log of the task. The status comes from the
   Windows result, not from the log. The log tells you what the script did.

The `privacy.hide_*` switches are on by default. Then **Runs as:** and **Script:** show `--`, and
the log path line is empty. See [CONFIGURE.md](CONFIGURE.md).

A task can fail again and again but look **OK** in the table. The
**Most failures in 7 days - top 10** chart shows such a task. Select a bar to narrow the Task health
table and the failed runs to that task.

A task that runs for a longer time each week can fail later, also when it succeeds now. To see
this, use **Duration trend - newest 200 runs**. The chart shows the successful runs of the selected
task, in seconds. Sixty seconds is one minute.

Look at **Recent changes - newest 50** on the **Collector & Changes** tab too. A changed task is a
common cause of new failures. Select the task in the **Filters** panel, or type its name in the
search box of the list. To find a removed task, use the search box.

The kinds of change are **Started failing**, **Became overdue**, **Recovered**,
**Setting note added**, **Setting note removed**, **Task added**, **Task removed** and
**Task changed**. The time on a row is when the collector found the change. It can be up to one
hour after the change happened.

To repair the task, change it in Windows Task Scheduler or fix its script. TaskDash does not change
tasks. The next hourly run updates the dashboard.

### 5. Acknowledge a known condition

TaskDash has no button to hide a failure. A failed task stays in the tiles until a run succeeds.
Three settings handle a condition that you know and accept.

**A setting note that you accept.** The **Setting notes** list on the **Collector & Changes** tab
shows tasks with a risky setting. An example is a task that skips a run when the computer was off.
A setting note is a risk, not a failure. It does not change a status or a tile. If the setting is
wrong, change it in Task Scheduler. If the setting is on purpose, accept the note:

1. On the computer that runs the collector, open `workflow_descriptions.json` in `C:\TaskDash`.
   If the file does not exist, create it. The wheel does not include it.
2. Add an entry for the task under `tasks`. The key is the task name from the **Task** column.
   The name of the script file also works. Case does not matter.
3. In `acknowledge_config_warnings`, write a word that appears only in the note that you accept.
   In `acknowledge_reason`, write why:

   ```json
   {
     "tasks": {
       "Nightly Export": {
         "acknowledge_config_warnings": ["StartWhenAvailable"],
         "acknowledge_reason": "Catch-up is off on purpose. A catch-up run would load the server in business hours."
       }
     }
   }
   ```

4. Save the file. The next collector run reads it. The note leaves the **Setting notes** list. The
   reason shows in **Task detail**, after `Accepted:` on the **Settings:** line.

The collector drops every note of that task that contains your word. Case does not matter. Choose a
word that only one note contains. For example, `To fix` is in most notes.

After the next run, **Recent changes** shows a **Setting note removed** row. That row does not
mean that someone fixed the setting. It can mean that you accepted the note.

**A task that starts late on purpose.** A task can start later than the grace time, and this can
be normal. Then add `"overdue_grace_minutes": 45` to the entry of that task in the same file. Any
other number also works. The number replaces `scheduler.overdue_grace_minutes` for that task only.

**Planned maintenance.** The config key `alerting.maintenance_windows` stops alert e-mails during a
time window. It applies to the tasks whose names match a pattern. It does not change the dashboard:
the task still shows as a problem. See [CONFIGURE.md](CONFIGURE.md).

### 6. Check the collector itself

If the dashboard looks wrong, or the heartbeat is **Late** or **Down**, check the collector.

**On the collector computer,** open a normal PowerShell window. Run the two lines from step 1 of
[INSTALL.md](INSTALL.md) (the folder and `$td`). Then run:

```powershell
& $td status
```

The command only reads. It prints two lines:

```text
Local: taskdash_20261017_202547.log: <time> - INFO - TaskDash publish complete.
Service: last collector run started 2026-10-17 20:25:47 UTC, exit_status success
```

- The **Local** line gives the newest log file in the `Logs` folder and its last line. A run that
  ended well has the last line `TaskDash publish complete.`
- The **Service** line gives the start time of the newest row in the **CollectorRuns** table and
  its `exit_status`. The values are `success` and `failed`. The time is in UTC.
- To print only the **Local** line, without a sign-in to ArcGIS, add `--offline`.

The `Logs` folder is in `C:\TaskDash`, unless `logging.log_dir` names another folder.

The command prints a result and does not judge it. It returns `0` also when the last run failed.

**In the dashboard,** open the **Collector & Changes** tab. The **Collector runs** table lists the
newest 48 runs of the collector task.

| Column | Meaning |
|---|---|
| **Status** | **OK** or **Failed**. A failed row has a colored background. |
| **Started** | When the run started. |
| **Duration** | How long the run took. |
| **Events read** | The Task Scheduler events read since the last run, also for tasks that are not listed. |
| **Runs saved** | The task runs added or updated. |
| **Task rows sent** | The task rows sent. TaskDash sends every row each time, changed or not. |
| **Metric rows sent** | The metric rows sent, counted the same way. |
| **Email sent** | **Yes** when the run sent an e-mail. |
| **Host** | The computer that ran the collector. |
| **Error** | The first 300 characters of the error of a failed run. |

Use the table as follows:

- For a **Failed** row, read **Error**. The full error is in the log of that run, on that host. The
  log name is `taskdash_YYYYMMDD_HHMMSS.log`, with the start time of the run. Then use
  [TROUBLESHOOTING.md](TROUBLESHOOTING.md).
- A gap in **Started** means the collector did not run in that time. For example, Task Scheduler
  stopped the run, or the computer was off. A run that Task Scheduler stopped leaves no row.
- A run that could not reach ArcGIS Online appears here after the next run that can.
- A run can send an e-mail and not save data. Compare **Email sent** with **Runs saved**.

**Email sent** is **Yes** when a problem started, changed or cleared. It is also **Yes** when a
problem is still open after `alerting.realert_hours` (24 by default). It is also **Yes** for the
daily check at the hour in `alerting.heartbeat_hour`. TaskDash sends alert e-mails only when
`alerting.enabled` is `true` and `alerting.to` has an address. See [CONFIGURE.md](CONFIGURE.md).

## Part 2: Reliability over time for the manager

A manager asks if the tasks are reliable, and if that is getting better or worse. The dashboard
answers with the widgets below. Open the **Overview** tab first. The run history keeps 90 days by
default. See `retention.runs_days` in [CONFIGURE.md](CONFIGURE.md).

### Success rate

- **Lowest 7-day success** gives the success rate of the weakest task in the last 7 days. It counts
  only tasks with two failures or more. The rate of a task is its runs that did not fail, divided by
  all its runs. The tile cuts the value to one decimal place.
- The **Failures (7 days)** column of the **Task health** table gives the failures of every task.
- In **Task detail**, the line **Last 7 days:** gives the runs, the failures and the skipped runs of
  one task. Compare the runs too. Two failures in a task that runs every 15 minutes are not the same
  as two failures in a daily task.
- **Most failures in 7 days - top 10** ranks the tasks by failures. It shows only tasks with at
  least one failure.

### Run history

- **Failed runs - newest 25** lists the newest failures of all tasks. It shows if a task fails once
  or every night. The history reaches back up to 90 days, but the list shows only the newest 25.
  To see only the failures of one task, pick the task in the **Filters** panel.
- **Duration trend - newest 200 runs** shows if a task becomes slower. It draws the successful runs
  of the task that you select in the Task health table.
- In **Task detail**, a line such as `Duration: 2.1 times the usual` can appear. It shows that the
  last run took much longer or much shorter than usual. The line needs a task with no other problem.
  The task also needs five or more finished runs in the last 30 days.

### Metric trends

The **Workflow Metrics** tab shows what the scripts write to their logs, for example the number of
copied records. A task can end without an error and still process much less data than usual.

- **Metric change** compares the latest value with the usual value for each task and metric. The
  usual value is the median of up to 30 earlier values. A metric needs three earlier values to have
  one. Flagged rows are at the top.
- **Metric range** gives the lowest, average and highest value, and how many times it was logged.
- **Metric trend** draws the line of one metric over time. To draw it, select a row in
  **Metric change**. A count that falls slowly over weeks passes each daily check, but the line
  shows it.
- **Latest metric values - newest 60** and **All values of the selected task** give the exact
  values and the log file of each value.

**Metric change** has three flags:

| Flag | Meaning |
|---|---|
| **Far below usual** | The latest value is 50% or more below a usual value of 10 or more. It is also lower than each of the last 30 earlier values. |
| **Far over usual** | A problem count is 50% or more over a usual value of 10 or more. It is also higher than each of the last 30 earlier values. |
| **Not in last run** | A metric that the task logs in at least 80% of its runs is missing from its newest log. |

A problem count is a count of failures, unmatched or blank items, or stale registrations. Gaps or
chains over a limit are problem counts too. A drop in a problem count is good news, so it gets no flag.

The metrics keep 90 days of values by default. See `retention.metrics_days`.

### Changes to task definitions

**Recent changes - newest 50**, on the **Collector & Changes** tab, records each change to a task.
It records a task that was added, removed or changed. It records a task that started failing,
became overdue or recovered. It also records a setting note that appeared or went away.

A **Task changed** row shows what changed. The `privacy.hide_command_lines`,
`privacy.hide_accounts` and `privacy.hide_paths` switches hide those parts of the row. The switches
are on by default. TaskDash keeps the change history for 365 days by default, but the list shows
only the newest 50. See `retention.changes_days`.

### What the dashboard does not show

- It does not chart the success rate over time. **Lowest 7-day success** shows the weakest task
  now. The dashboard keeps no history of that tile.
- It shows no success rate for a month or a quarter. The history of runs is 90 days by default.
- The failed-run list and the change list show 25 and 50 rows. A long outage can push older rows
  out of the list, though they stay in the service.

## Use the dashboard on a phone

The phone view has no Filters panel and no tables. It shows the eight tiles, the list **Tasks that
need attention or are disabled**, and **Failed runs - newest 10**. Use it for a quick look. For the
cause of a failure, open the dashboard on a computer.

## Colors and symbols

Every status has a word, a symbol and a color. The word is the main signal. The color is the second.

| Color | Words |
|---|---|
| Green | **OK**, **None** |
| Blue | **Running** |
| Amber | **Warning**, **Overdue**, **Stuck**, **Blocked**, **Late**, **Low**, **Open** |
| Red | **Failed**, **Did not start**, **Account not signed in**, **Stopped at time limit**, **Terminated**, **Down**, **Poor** |
| Grey | **Disabled** |
| Gold | **Not yet run** |
| Slate | **Setting note** |

## Limits

- The dashboard shows what the collector saved at its last run. The collector runs every hour by
  default, so a problem can take up to one hour to show.
- TaskDash judges a task by its Windows result code. A script that exits with `0` after an error
  looks **OK**. The `N log errors` text in the **Why** column shows it. The **Latest log:** line of
  **Task detail** shows it too.
- Scrubbing secrets from command lines and log samples is best effort. See "Known limits" in
  [INSTALL.md](INSTALL.md).

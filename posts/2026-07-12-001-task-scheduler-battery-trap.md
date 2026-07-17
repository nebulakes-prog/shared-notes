# I booted my laptop after six days away — and Windows Task Scheduler silently refused to revive my AI agent

I run a small personal automation stack on a Windows laptop: a Telegram gateway that captures notes into my knowledge base (the "AI agent" of the title), a nightly batch job, a morning briefing job, and a handful of maintenance tasks. All of them are registered in Windows Task Scheduler.

Work kept me away from this laptop for six days — it sat powered off the whole time. When I finally booted it, I expected the stack to come back up on its own. Instead, the capture bot stayed silent. No crash dump, no error notification, nothing in my own logs. The service hadn't failed — it had simply **never been started**.

## The investigation

Task Scheduler's history showed the launch attempts ending with result code `0x800710E0`:

> The operator or administrator has refused the request.

That message is spectacularly unhelpful. Nobody had refused anything — I'm the only operator, and I never touched those tasks. It's also a generic code (`ERROR_REQUEST_REFUSED`), so it doesn't point at any specific condition by itself.

In my case, comparing the task's power conditions against the laptop's battery state gave the answer: **the task's own conditions were refusing the launch**. And the condition responsible was one I had never consciously set.

## The root cause: two defaults nobody reads

When you register a scheduled task — whether through the GUI or PowerShell's `Register-ScheduledTask` — two settings default to the conservative option:

| Setting | Default | Effect |
|---|---|---|
| `DisallowStartIfOnBatteries` | `True` | Task will not **start** while on battery power |
| `StopIfGoingOnBatteries` | `True` | Running task is **killed** when AC power is unplugged |

These defaults make sense for the original use case: heavy maintenance jobs on office PCs, where you don't want a defrag draining someone's battery.

They make no sense for a laptop that hosts resident services. When I booted mine, it was running on battery — as a laptop usually is when you open it away from a desk. Every scheduled launch was being refused, and no amount of waiting would have changed that: a task whose start condition is never met stays dead indefinitely. Eight of my tasks were in this state — the gateway, the nightly batch, the morning briefing, all of them.

The worst property of this failure mode is the **silence**. Your own application logs show nothing, because your application never ran. Unless you routinely open Task Scheduler's history pane (who does?), the first signal is a user noticing that something has been dead for days.

## The fix

Change the two properties on the *existing* settings object and save it back:

```powershell
$task = Get-ScheduledTask -TaskName "my-resident-service"
$task.Settings.DisallowStartIfOnBatteries = $false
$task.Settings.StopIfGoingOnBatteries = $false
Set-ScheduledTask -InputObject $task
```

A tempting shortcut — building a fresh settings object with `New-ScheduledTaskSettingsSet` and passing it to `Set-ScheduledTask -Settings` — is a trap of its own: it replaces the task's **entire** settings block, silently resetting anything else you had configured (restart policy, execution time limit, `StartWhenAvailable`, …) back to defaults. Patch the existing object instead.

Note the awkward naming: the properties are *negative* (`DisallowStartIfOnBatteries`), so the fix sets them to `$false`. Double negatives are a great way to misread an audit, so verify after applying:

```powershell
(Get-ScheduledTask -TaskName "my-resident-service").Settings |
    Select-Object DisallowStartIfOnBatteries, StopIfGoingOnBatteries
# Both should now be False
```

I applied this to all eight tasks. The next morning, the overnight batch and the morning briefing both ran with exit code 0 — on battery.

One honest caveat: these flags remove the *task-level* power gate. Windows Battery Saver and idle conditions can still defer some scheduled work, so "flags cleared" is necessary but not a blanket guarantee.

## Lessons for anyone running resident services on a laptop

1. **Registration-time checklist, not memory.** Any task that must survive on a laptop needs the two battery flags cleared *at registration time*. I added this to the checklist my automation follows whenever it registers a new task, because I will forget, and so will you.

2. **Silent failures need an independent heartbeat.** The service can't report its own absence. Something *outside* the scheduled task should verify liveness — a check on the log file's last-modified time, a ping to a dead-man's-switch, anything that turns "dead indefinitely" into "noticed within a day."

3. **Scope your fix to the actual failure.** These two flags only control start/stop on battery. They do *not* wake a sleeping machine — `-WakeToRun` wakes the machine at trigger time, and `-StartWhenAvailable` handles a missed trigger after it wakes; sleeping through a trigger is a different failure I haven't addressed yet. Knowing precisely what you fixed keeps you from false confidence.

4. **This is a mostly-Windows trap.** cron has no built-in power-source gate, and systemd doesn't apply one by default (a unit *can* opt in via `ConditionACPower=`). If you're porting automation habits from Linux to a Windows laptop, this class of failure won't be on your radar — it wasn't on mine.

To be fair about the damage: the six-day capture gap was my own doing — the laptop was off, and a backfill I'd built recovered the missed messages from the source. What the flags added was worse in kind: an outage that would have lasted *indefinitely* after boot, with no error surfaced anywhere, on a machine I believed had come back to life. Diagnosis took about an hour; the fix took two minutes. That ratio — a silent, open-ended failure versus minutes of prevention — is exactly why default settings deserve a place in your threat model.

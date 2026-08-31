---
title: "Why Fluent Bit misses logs from short-lived pods"
description: "A connector pod finished in under a minute and its logs never reached Loki. The cause wasn't log deletion — it was a discovery race in the tail plugin."
pubDate: 2026-08-30
tags: ["kubernetes", "fluent-bit", "loki", "observability", "eks", "aws"]
---

A developer on my team came to me with a question that sounds simple and isn't:

> "My source connector ran. I can see the pod completed. But there are no logs in Loki. Did the task actually finish, or did it fail silently?"

That second sentence is the real problem. A missing log line isn't just an inconvenience — it removes the ability to answer whether work happened at all. The pod status says `Completed`, but `Completed` only means the process exited zero. It says nothing about whether the connector wrote the rows it was supposed to write.

So we had a pod that claimed success and no evidence to back it up.

## The first wrong assumption

My initial suspicion was the application. That's where most people start: maybe the connector isn't logging at all, maybe it's writing to a file instead of stdout, maybe the log level is wrong in this environment.

That turned out to be a dead end, and in hindsight I was looking in the wrong place for a specific reason: **I didn't know these were short-lived pods.**

They looked like ordinary workloads in the dashboard. Once I understood what they actually were — source connectors that spin up, do one job, and terminate — the shape of the problem changed completely. This wasn't an application question anymore. It was a log-shipping question.

If you take one thing from this post, take that: *before debugging missing logs, find out how long the pod lived.* It changes which half of the system you should be looking at.

## The detail that made it worse

Here's the part that still amuses me.

These connector pods were generously provisioned. Plenty of CPU, plenty of memory. The workload was computationally heavy, so someone had sized it well, and the pod was finishing its task in **under a minute**.

The pod was fast *because* it was well-tuned. And it lost its logs *because* it was fast.

Performance work broke observability. Nobody who tuned those resources could have predicted that, and there's no warning anywhere that tells you a faster pod is a less observable one.

## What actually happens to the logs

Let's be precise about the mechanics, because this is where most explanations go wrong.

When a container writes to **stdout/stderr**, the log does not live inside the container's overlay filesystem. The container runtime (containerd) writes it to the node's filesystem:

```
/var/log/pods/<namespace>_<pod-name>_<uid>/<container-name>/0.log
```

with symlinks under `/var/log/containers/`. This is the path your Fluent Bit DaemonSet tails.

That distinction matters. I've seen this problem explained as "the pod's volume was cleaned up, so the logs went with it." For a stdout-logging container, that's not what happens. The overlay filesystem is irrelevant here — the logs were never in it.

What deletes that file is **kubelet**. When the pod object is removed from the API server, kubelet garbage-collects the corresponding directory under `/var/log/pods`.

So far this sounds like a straightforward race: the pod dies, the file disappears, and Fluent Bit doesn't get there in time.

That's close, but it's still not right.

## It isn't log deletion. It's a discovery race.

Fluent Bit's `tail` plugin holds an open file descriptor on every file it's reading. On Linux, an open descriptor keeps the inode alive even after the file is unlinked. If Fluent Bit has already opened `0.log`, kubelet can delete the directory and Fluent Bit will still finish reading the remaining bytes.

Deletion, by itself, is survivable.

The failure is one step earlier. Fluent Bit never opened the file, because it never knew the file existed.

The `tail` input discovers new files by periodically re-scanning its glob path. That scan happens on `Refresh_Interval`, and **the default is 60 seconds**.

Now line up the timeline:

```
t=0s    Pod is scheduled and starts. containerd creates 0.log
t=55s   Pod completes. Controller cleans up the pod object
t=56s   kubelet garbage-collects /var/log/pods/<...>/
t=60s   Fluent Bit performs its next directory scan
        → finds nothing → no logs in Loki
```

The file existed for 56 seconds and Fluent Bit looked at the wrong moment. There was no data loss in the sense of bytes being dropped in flight. There was never a read at all.

This reframe matters because it points at a different fix. If you believe the problem is deletion, you go looking for ways to preserve files. If you understand it's discovery, you go looking for ways to shorten the gap between file creation and first read — or to widen the window in which the file exists.

## Fixes, in the order I'd try them

### 1. Shorten the discovery interval

```ini
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Refresh_Interval  2
```

This is the smallest change and it addresses the race directly. Dropping from 60s to 2s means a pod has to live for less than two seconds to escape entirely.

**Trade-off:** every refresh re-scans the glob. On a node with a high pod count and a lot of log files, this costs CPU and inotify watches. Measure the Fluent Bit pod's resource usage before and after rather than assuming it's free.

This narrows the window. It does not close it.

### 2. Keep the pod object alive a little longer

This is the more reliable fix, and it's the one I'd reach for on anything important.

kubelet only cleans up the log directory once the pod object is gone. So don't delete the pod object immediately.

For a `Job`:

```yaml
spec:
  ttlSecondsAfterFinished: 300
```

For a `CronJob`:

```yaml
spec:
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
```

Five minutes of retained pod objects buys the log pipeline an enormous margin compared to a 60-second — or even 2-second — scan interval.

Worth checking: if you have automation that reaps completed pods aggressively for dashboard cleanliness, that automation may be your actual root cause. It's worth auditing before you tune anything in Fluent Bit.

### 3. Check for backpressure

This one is invisible until you go looking for it.

If a `tail` input hits `mem_buf_limit`, Fluent Bit pauses that input until the buffer drains. If a short-lived pod's file appears and disappears during that pause, it's gone — and this failure looks identical to the discovery race from the outside.

```ini
[SERVICE]
    storage.type              filesystem
    storage.path              /var/log/flb-storage/
```

Filesystem-backed buffering makes pauses far less likely, and lets buffered records survive a Fluent Bit restart. On a node that gets bursty traffic, this is worth having regardless of the short-lived pod problem.

### 4. Stop tailing, start pushing

The three fixes above all make the race less likely. None of them eliminate it, because tail-based collection is *structurally* a race when the thing you're collecting from may outlive nothing.

The structural fix is to invert the direction. Have the application emit logs directly — OTLP to a collector, or straight to Loki — so nothing depends on a file being noticed before it's removed.

A sidecar is the middle option, but be careful: the sidecar terminates alongside the main container, so you need a `preStop` hook and an explicit flush, or you've just moved the race somewhere less visible.

This is a real architectural change and it isn't always worth it. For a pipeline where "did this task complete?" is a question someone will actually need answered — a payments job, a data sync, a compliance-relevant task — it usually is.

## What I'd tell you honestly

None of these give a hard guarantee.

Tail-based log collection on Kubernetes is best-effort by design. You are reading a file that something else owns and may remove at any moment. You can make the window very wide and the scan very fast, and you will still, occasionally, lose a pod that lived for 400 milliseconds.

If a log line is genuinely critical — if a compliance or financial process depends on it — it should not travel through a log pipeline at all. It should be a database write, a queue message, or an explicit status update from the job itself.

Logs are for debugging. They're not a system of record, and short-lived pods are where that distinction stops being academic.

## The takeaway

The developer's original question was "where are my logs?" The useful question turned out to be "how long did the pod live?"

Once I knew the answer was under a minute, everything else followed — including the uncomfortable realisation that giving the pod more resources is what made it disappear from Loki.

If you're running source connectors, Jobs, CronJobs, or anything else that exists briefly and exits, go check your `Refresh_Interval` right now. If it's still at the default, you have this problem. You just haven't had a developer walk over and ask about it yet.

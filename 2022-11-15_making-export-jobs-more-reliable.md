# Making export jobs more reliable

**Author:** Jamis Buck, Programmer, SIP
**Published:** November 15, 2022
**Source:** <https://dev.37signals.com/making-export-jobs-more-reliable/>

*Long-running jobs can create maintenance and support nightmares as they run up against resource and time constraints. What if we could break them up—automatically—into smaller chunks of work?*

---

It was August 2022, and export requests—asynchronous jobs that assemble, zip, and upload an offline-viewable version of a customer’s data—had become a bit of a headache. Most of the time they ran without a hitch, but for some customers with a lot of data the exports would randomly die, bubbling up in our On-call chat room as `Resque::DirtyExit` exceptions. Some weeks we’d see more than a dozen of these.

Over time, we had applied various bandaids—increasing memory limits, reducing the number of active workers, and so forth—which made things incrementally better, but the problem itself persisted. Each failed export would be retried, sometimes repeatedly, and the worst offenders would have to be run manually in a console session.

Ultimately, the real issue was how long the export took to run. Each time we deployed Basecamp, we would hot-swap our Resque pool, which orphaned any active jobs. We allowed those orphaned jobs to run gracefully to conclusion…but with a caveat: if they took too long we would kill them.

We fiddled a bit with the definition of “too long”, but no matter what threshold we set there, the long tail ensured that there would always be an account that would run longer. We needed to fix this, once and for all.

---

## Interruptibility

The solution we hit on was to make these long-running export jobs *interruptible*. The flow would go something like this:
- The job is picked up by a worker and execution begins.
- After some amount of time, the job expires, re-enqueues itself, and exits. (Or, alternatively: a [retryable exception](https://api.rubyonrails.org/classes/ActiveJob/Exceptions/ClassMethods.html#method-i-retry_on) is caught, and the job is automatically re-enqueued before exiting.)
- The job is picked up *again* by another worker and resumes from where it left off.
- Repeat steps 2 and 3 until the export finishes.

It seemed simple enough on paper. Here’s how it came together in practice.

#### Job expiry

It’s tempting to look at that process description and think of “expiry” as a hard-stop via some kind of asynchronous timer process. We opted for a much simpler system.

When the job starts (or restarts), we set an `interrupt_at` deadline:

```
interrupt_at = Time.now + time_until_interrupt
```

That `time_until_interrupt` value defaults to 15 minutes. (More on that in a bit.)

Once that deadline is set, the export proceeds as normal, but every time it finishes writing a file to the scratch disk it now compares the current time to that `interrupt_at` deadline:

```
if Time.now >= interrupt_at
  increment! :interrupt_count
  raise ProgressInterruption
end
```

The `ProgressInterruption` exception is then rescued and the job re-enqueues itself:

```
# ...
rescue ProgressInterruption
  perform_later
# ...
```

It’s important to note that this implementation does not strictly enforce the `interrupt_at` deadline; depending on how long it takes to write the next file, it may miss that deadline by several minutes. For example, our exports pull some customer files from S3, and exceptionally large ones may take a while to download. The job will appear to stall while this is happening.

Don’t panic! The default interrupt interval of 15 minutes means that even with a file that takes a ridiculously long time to download—say a half hour or more—our export job will simply download the file, write it, and *then* interrupt itself. We’re safe from export jobs that monolithically run for dozens of hours.

#### Job resumption

How does a resuming export job know where it left off? We decided to let the scratch disk itself be the source of truth.

When an export resumes, it simply starts again from the beginning, using the original scratch disk and directory. If it recognizes that some chunk of work has already been exported, it will skip that chunk and move to the next one.

For example, when exporting a project, the exporter will first write all the message boards, todo sets, folders, etc, belonging to that project. After all those are done, it will then write the project’s perma page.

```
def process
    @chat_transcripts  = dock.chat_transcript_recordings
    @inboxes           = dock.inbox_recordings
    @message_boards    = dock.message_board_recordings
    # ...

    export @chat_transcripts, @inboxes, @message_boards, ...

    render "exporter/project/index", to: export_project_index_path(@project)
  end
```

This means, the next time around, the exporter can first look to see if the project perma already exists on disk; if it does, then the project has already been exported, and all the associated work (message boards, todo sets, etc.) can be skipped.

```
def export(*models)
    models.each do |model|
      writer_for(model).visit_if_unvisited do |writer|
        writer.process
      end
    end
  end
```

In this case, the project perma is a kind of *waypoint*: it indicates, by its existence, where the export has already been. Attachments, uploads, and other files can also be considered waypoints. Instead of downloading those files from the cloud again, we can see that they already exist, and skip them.

Some entities are more complicated, like multi-page chat transcripts. In cases like this, the exporter will write an explicit waypoint file to a temporary location in the scratch directory once all the transcript pages have been written.

Thus, every entity has *something* that represents the work done to add it to the export, and those little *somethings*—those waypoints—are what allow the exporter to quickly get up to speed when it resumes.

#### Waypoint performance concerns

Is that true, though? *Is* the exporter able to get up to speed *quickly*, when it resumes? We were a little worried, initially, that running the export from the beginning each time might get expensive, but it turns out that with good waypoint placement, the overhead is negligible. A behemoth of an export that runs for three days might be interrupted and resumed dozens of times, and in all that time spend only a total of two or three minutes resuming. That’s less than 0.2% of the export’s total duration.

This is because, with waypoints wisely placed, the most expensive bits of work (exporting entire projects) are skipped entirely. Even within a project, by placing waypoints for top-level entities like todo sets and message boards, we skip the need to query and re-analyze individual messages and todo lists. Resuming within a project is thus quite fast, as well.

---

## Pinning jobs to a single server

The last major hurdle to figure out was how to make sure the export could access the original scratch disk when it resumed. As presented so far, interrupted exports might be resumed on a different server, and possibly in an entirely different data center. Without access to that scratch disk, the export would have to start over and lose all benefit of the work it had done so far. In the worst case, a job might even “ping pong” back and forth indefinitely between servers.

Because jobs might be resumed anywhere, network drives are not an efficient option. Instead, we created *per-server Resque queues*—queues that are only handled by a single server—and made sure each job added itself to the appropriate server’s queue when it was interrupted.

We accomplished this by rewriting portions of the resque configuration on the fly, at deploy time, on each of the export servers. Each server references the default export queue (`bc3_production_export`) but also defines a queue specific to that server (e.g. `bc3_production_export_bc3-export-01`).

```
# <% hostname = `hostname -s`.chomp %>
production:
  "bc3_production_export,bc3_production_export_<%= hostname %>": 6
```

The rewriting itself is done via a Rake task, invoked on each server during the deployment process:

```
task "resque:config:rewrite" do
  Dir[Rails.root.join("config/resque_pools/*.yml")].each do |config_file|
    new_config = ERB.new(File.read(config_file)).result
    File.write(config_file, new_config)
  end
end
```

With those queues defined, the job’s [`queue_as`](https://api.rubyonrails.org/classes/ActiveJob/QueueName/ClassMethods.html#method-i-queue_as) declaration can place the job in the appropriate queue, depending on whether or not it has been interrupted before:

```
queue_as do
  export = self.arguments.first

  if export.interrupt_count > 0
    :"bc3_production_export_#{`hostname -s`.chomp}"
  else
    :bc3_production_export
  end
end
```

Thus, a brand-new export request will go in the default export queue, and an export that has been interrupted will go in the queue specific to the current server, ensuring that it retains access to its scratch disk and directory.

---

## Selecting an optimal time_until_interrupt

When we deployed this, we chose the 15 minute interrupt interval primarily so that we could see it succeed or fail in real-time—no one wants to wait hours to see if something is going to break under production loads. We figured we’d bump it up to an hour or four after we were confident the system was working as intended.

However, in retrospect, the 15 minute interval seems about right. What started as an empirical “guess” has actual numbers behind it now, and it looks like we guessed well. During the first two weeks of this new implementation, 8,000 exports were requested, and *more than 98% of them* finished in less than 15 minutes.

This means that fewer than 2% of all exports ever even hit this “interrupt/resume” logic. Sure, at the other end of the spectrum we have some handful of exports that will be interrupted and resumed dozens of times—in fact, the record so far is more than 80 interruptions for a single export. But in practice, that has little impact on how long the export takes to finish.

Short-running processes have other benefits, versus their longer-running cousins:
- They are quicker to pick up new changes from a deploy.
- They have less opportunity to balloon in memory usage.
- They allow us to more quickly identify potential issues with the interrupt/resume system, and with exports in general.

Ultimately, while that 15 minute interval could be experimented with, we probably wouldn’t want to go higher. For now, we’re content to stay with our initial guess.

---

## Analysis

Since deploying these interruptible exports, programmer toil (relating to exports) has gone down significantly, from a dozen failed exports per week, to zero, with that zero holding steady now for more than four weeks.

It’s probably safe to say that we reached our goal: export jobs are far more reliable now.

Case closed!

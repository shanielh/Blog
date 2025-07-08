From Developer Tool to Production Asset: Our Journey with Async Profiler

When we created Upsolver, our goal was to build a high-performance, headless big data ELT tool. To achieve this, we deliberately chose a monolithic architecture—and we're not ashamed of it. While monoliths come with trade-offs, they gave us the performance and control we needed early on.

One challenge, though, was debugging performance issues. In a monolithic system, a performance regression isn’t usually tied to a single recent code change—it could be buried within thousands of lines modified over weeks. Unlike microservices, where isolated services simplify blame assignment and performance debugging, our architecture required more robust tooling to understand what was happening inside the JVM in production.

Upsolver is a low-code SQL engine that lets users create pipelines to move data from source to destination, supporting transformations, joins, aggregations, and more. Since we deploy Upsolver into customer-managed environments, we often don’t have access to the actual machines or data.

So when a customer reported, "This job seems slow", we took it seriously. We started with metrics. If the metrics revealed a bottleneck, we fixed it. But if they didn’t, we needed a way to profile production workloads remotely—without affecting customer environments.

That’s where Async Profiler came in.

## What JFR and Async Profiler Bring to the Table
Java Flight Recorder (JFR) is a tool for collecting diagnostic and profiling data about a running JVM application. The data can be visualized as flame graphs that show CPU usage, memory allocations, or lock contention.

Flame graphs are built from stack traces, where the width of each bar indicates how much time or resources a particular function consumed. They’re an intuitive way to spot hotspots in your code.

Async Profiler is a low-overhead, sampling-based profiler that can generate JFR files. It’s particularly well-suited to production environments, and supports CPU, memory, lock, and itimer profiling.

## How We Made It Easy to Profile Applications

### Triggering a JFR dump

At Upsolver, one of the first thing internal tools we built, is a configuration utility
that allows changing configuration values at runtime, you can map a configuration
value (convert a string to long, for example), and even register for changes on a configuration
value, so we created a configuration value that is most likely dummy, but when it changes,
it ran Async Profiler and then copied the output to an S3 bucket in a known path format: 

```s3://thread-dumps/yyyy/MM/dd/HH/mm/millisecond-<cluster-id>-<instance-name>.jfr```

### Pulling JFRs

Copying the JFR locally would be an easy operation if you have access to the bucket:

```bash
aws s3 sync s3://thread-dumps/yyyy/MM/dd /tmp/dumps/ --exclude '*' --include '<cluster_id>'
```

### Opening JFRs

Opening the JFR to flame graphs would require running a script that is stored in our git repository:

```bash
fd --type f '.jfr$' -x ~/Documents/GitHub/repo/scripts/open-jfr.sh
```

[The script is attached here](https://gist.github.com/shanielh/b80c02d4164eeff1c0b979713bdce988), 
it would download any dependency needed in order to open the JFR, 
so developers will not have to install anything in order for it to run.

## Impact

Any performance issue we tackled had a ticket opened by the Customer, that if required
extra investigation, would be converted also to a Jira Ticket, because it was so easy
to create JFR files, our Customer Operations team handled creating them and gave 
developers a line of code to run in order to copy the JFRs locally. We gave a great
customer support using the dumps, since it was taken about days to figure out how
to tackle and solve a problem.

At top of that, at some point, Upsolver changed its pricing model to a consumption based
model (Pay per GB processed) instead of a compute based model (pay per Core * Time),
that aligned the customer’s interest in reducing compute costs with our own engineering 
goals to optimize for less compute resources, so eventually we took JFR dumps every few weeks
in order to see what's left to optimize and do that.

#AsyncProfiler #JVM #JFR #ProductionDebugging #PerformanceEngineering #Observability
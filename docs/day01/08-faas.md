# 8. Lambda functions

Today's sections have been about renting ever-smaller slices of computing. The next step is to
rent *only the execution of one function*, and nothing else. This is **FaaS** (Function as a
Service), usually called **lambda functions** after AWS Lambda (2014), the service that
popularised it. AWS in turn borrowed the name from the anonymous functions (`lambda x: ...`)
of functional programming.

## 8.1 The motivation: the idle server

Consider the following task: *"whenever a new data file lands in the bucket, validate it
and update the catalogue."* Ten seconds of compute, a few hundred times a day.

Built with what you know so far, this becomes a VM (section 4) running a small service that
polls the bucket 24 hours a day, of which it computes for perhaps 30 minutes. On a commercial 
provider you would pay for, patch, secure and monitor a full server that is 98% idle. There 
is also no elasticity (section 2.3) here, because a single VM does not grow or shrink with 
the load.

FaaS offers a different arrangement. **You give the platform a function; the platform runs one
copy of it per event, and bills only the milliseconds it runs.** Between events there are no
processes and no cost. During a burst of a thousand events, a thousand copies run in parallel
automatically.

```{mermaid}
flowchart LR
    E1["Event:<br/>file uploaded"] --> F
    E2["Event:<br/>HTTP request"] --> F
    E3["Event:<br/>timer / schedule"] --> F
    F["Your function<br/>(runs per event, then vanishes)"] --> O["Side effects:<br/>write to bucket / DB,<br/>call an API, notify"]
```

:::{note} Jargon: *serverless*
FaaS is the main member of the **serverless** family. Servers still exist, of course; the
word means that *you* no longer see, size or patch them, or pay for them while they sit idle.
:::

## 8.2 What a lambda function looks like

A lambda function is an event handler and nothing more. The AWS Lambda shape, in Python, looks
like this:

```python
def handler(event, context):
    """Called once per event. 'event' says what happened."""
    bucket = event["bucket"]
    key = event["key"]                      # e.g. "incoming/scan-042.nc"
    result = validate(bucket, key)          # your actual work
    return {"key": key, "valid": result}
```

You upload this (with its dependencies) and declare its **trigger** (an object landing in a
bucket, an HTTP endpoint, a cron schedule, a message on a queue), and that is all. There is no
machine to create, no OS, no floating IP and no host to keep updated.

The arrangement comes with constraints, and these determine what FaaS is suitable for:

- **Stateless.** Each invocation may run on a fresh instance; nothing survives in memory or
  on local disk between calls. State lives outside, in object storage or a database.
- **Short.** Platforms cap the runtime at minutes rather than days. A weather model would not
  run in a lambda, but the notification that its output has arrived could.
- **Cold starts.** After a period of idleness, the first event incurs a startup delay while the
  sandbox is created. This is fine for pipelines and can be noticeable for latency-critical APIs.

**Good fits**: per-file processing on arrival (validate, convert, thumbnail, index), glue between
services, webhooks, scheduled small jobs, fan-out orchestration.

**Poor fits:** long simulations, anything needing GPUs-for-hours, stateful services,
latency-critical inner loops.

## 8.3 Open-source FaaS

Lambda is proprietary and AWS-only. This is a clear case of the lock-in concern from section
2, since your code plugs into provider-specific triggers. The open-source world
reimplemented the model (usually on top of *Kubernetes*), so a private or community cloud can
offer it too:

- **OpenFaaS**: a fairly approachable option. Any container (or a simple handler file) becomes
  a function with a CLI (`faas-cli up`); a good first FaaS to self-host.
- **Knative**: a Kubernetes-native platform that powers several commercial offerings
  (including Google Cloud Run).
- **Apache OpenWhisk**: a longer-established project, which underpins IBM's cloud functions.

## Recap

- FaaS/lambda = **rent the execution of a function, per event, per millisecond**. It is the
  end point of the day's progression towards renting ever less, and the most fine-grained form
  of elasticity (down *to zero*).
- The shape: an **event handler** plus a **trigger** (bucket upload, HTTP, schedule, queue).
  The constraints: **stateless, short, cold starts**; state belongs in object storage or a
  database.
- Suited to **small, bursty, event-driven** work (per-file processing, glue, schedules); not
  suited to long, stateful, or hardware-hungry jobs.
- Open-source implementations exist (**OpenFaaS**, **Knative**, **Nuclio**, **OpenWhisk**),
  all running on Kubernetes, so FaaS does not necessarily mean lock-in.

## Check yourself

1. A colleague wants to run a 6-hour patient-cohort simulation "as a lambda, so it scales".
   What do you tell them, and which *part* of their pipeline does belong in a function?
2. Why does the statelessness constraint exist? (Think about what the platform must be free
   to do to deliver per-event scaling.)

:::{dropdown} Solutions
**1.** It violates both core constraints: 6 hours exceeds any platform's runtime cap, and a
simulation typically carries heavy in-memory state.
What does fit functions are the edges of the pipeline: triggering the run when input
data lands, validating inputs beforehand, registering/announcing results when output files
appear in the bucket, notifying on failure. The functions act as glue around the pipeline,
while the heavy computation runs on VMs or Kubernetes.

**2.** To scale per event, the platform must be free to run your code as **many parallel
copies**, on **any machine**, starting **fresh** each time, and to **destroy instances as
soon as they finish** (that destruction is what makes scale-to-zero, and the billing model,
possible). Each of these would silently lose in-memory or on-disk state, so statelessness is
what makes the model implementable in the first place.
:::

---

Next: [Containers overview](09-containers.md), the other lightweight unit of computing.

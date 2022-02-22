I have used a lot of instrumentation libraries before, so I'm going to approach
Glean the way I normally would: slightly haphazardly.

(This is very stream-of-consciousnessy, going to try to just liveblog what I do as I do it.)

I'm going to try to add Glean to a project and get all the way to writing a query against it.

## Getting things started
I set up a project, and pip installed glean_sdk.

I imported glean into a repl and proceeded to mess around.

I found `help(glean.Glean)` and this shows me how to get started, I think.

I added this to the demo's init, which is taken from what was in the doc,
```python
Glean.initialize(
    application_id='mcfunley-asteroids',
    application_version='0.0.1',
    upload_enabled=True)
```

That results in `TypeError: data_dir must be provided`. So I `mkdir .glean` and
provide that. The app runs again! After that I went to see what happened in the
directory. There's a `db` in there which got me momentarily excited, imagining
that there'd be a sqlite database there or something, but that doesn't seem to
be what it is. There's opaque data there which I don't know how to inspect right now,
which I guess I'll come back to.

## Adding a first event

I decide that I should try to add an event, I'll just try to do `app_start`. Back to
the docs: https://mozilla.github.io/glean/book/reference/metrics/event.html

This looks like I'll need to make a yaml file first. Clicked around and found:
https://mozilla.github.io/glean/book/reference/yaml/metrics.html

Copied the contents of the sample file into metrics.yaml and edit to the stuff
I need to fill out:
```yaml
---
$schema: moz://mozilla.org/schemas/glean/metrics/2-0-0

$tags:
  - frontend

app:
  start:
    type: event
    description: The app started
```

It says: `Missing required properties: bugs, data_reviews, expires, notification_emails`

Thoughts here:
* Why would each metric have to have its own bug link? (Same question for data review?)
* Can I just turn these off?
* Default expiration should be "never"
* Notification emails probably don't make sense for metrics that never expire? Although sure we could have more kinds of notifications, e.g. "it broke"

Some of this is ok overhead (like it goes away after initial setup), and is generally useful for a real project.
I feel like I've contacted a big wall of bureaucracy in this context though.

I paste the placeholders back in to make progress (replacing the `expires` thing with "never", which is me
guessing at the right syntax:
```yaml
---
$schema: moz://mozilla.org/schemas/glean/metrics/2-0-0

$tags:
  - frontend

app:
  start:
    type: event
    description: The app started
    notification_emails:
      - CHANGE-ME@example.com
    bugs:
      - https://bugzilla.mozilla.org/123456789/
    data_reviews:
      - http://example.com/path/to/data-review
    expires: never
```

That seems like it worked and the app works again. Now I'm wondering if I can just specify those
things I don't want at the app level,
```yaml
---
$schema: moz://mozilla.org/schemas/glean/metrics/2-0-0

$tags:
  - frontend

app:
  notification_emails:
    - CHANGE-ME@example.com
  bugs:
    - https://bugzilla.mozilla.org/123456789/
  data_reviews:
    - http://example.com/path/to/data-review
  expires: never

  start:
    type: event
    description: The app started
```

No dice, so I revert it.

Ok back to python. I add some code to record the event: `metrics.app.start.record()`. This runs,
is it working? I look in the `.glean` directory. The `.glean/events/events` has data now. Very nice.

## Trying to submit data
I am assuming, but don't really know for certain, that this is just writing files to disk. Let's try
to get it to the end of the pipeline now so I can see the stuff I'm trying to actually create here
(namely: tables I can query).

I go skim the Pings section of the docs. Looks like I need to make a pings.yaml file. I take the sample
and hack it up. I don't remove stuff that feels irrelevant this time.

When I go back to section 4.1, I see that there's a `default` ping mentioned. I don't really care about
pings. I try this, under `app.start`:
```yaml
    send_in_pings:
      - default
```

Idle questions: am I going to have to specify that every event needs to be included in the default ping? Again
why am I concerning myself with pings already? The library should be just shipping my stuff to the server on a
background thread.

I `rm -rf pings.yaml` since I didn't have my own opinions about pings in the first place.

Now the app says: `Attempted to submit unknown ping 'events'` when I close it. Are there more implicit pings than
`default`? After scrolling up in my terminal I realize that it's been saying that for a while, but I didn't
notice because I hadn't been trying to submit data yet.

Ok, I guess I don't need that `send_in_pings` entry, so I remove it. Now how do I make the `events` ping not unknown?

Is something happening? Is this trying to ship data to ingestion? Maybe? At this point it'd be useful to know
what server dingus to look at. Is there a debug glean backend I'm supposed to be looking at? This would be
my expectation for any real product at this point. I search the docs kind of randomly for "dataset," "bigquery,"
etc, and don't find anything right away. But then I get to the Debugging section in the docs and find
https://debug-ping-preview.firebaseapp.com/ - hooray!

My app isn't there. I guess it's probably still the `unknown ping 'events'` thing. I need more information. The
debugging docs suggest I set `log_level=logging.DEBUG`, so I do and rerun. A bit more info now but not helpful,
```
ERROR:glean:Attempted to submit unknown ping 'events'
INFO:glean:No RLB symbol found. Not trying to flush the RLB dispatcher.
```

I switch gears a bit and search the book for "events ping" and find,
https://mozilla.github.io/glean/book/user/pings/events.html#the-events-ping

Nothing there makes me understand how the events ping would be unknown. I'm gonna take a stab in the dark here
and try to add back the pings.yaml file and define the events ping.

```yaml
---
$schema: moz://mozilla.org/schemas/glean/pings/2-0-0

events:
  description: >
    can has events?
  include_client_id: true
  notification_emails:
    - CHANGE-ME@example.com
  bugs:
    - http://bugzilla.mozilla.org/123456789/
  data_reviews:
    - http://example.com/path/to/data-review
```

No dice, it still says `Attempted to submit unknown ping 'events'`. I give up on the docs at this point and go
try to read the source: https://github.com/mozilla/glean/blob/d6fcde322a99fe556eb2d94c7929062b25d0e66a/glean-core/src/lib.rs#L719

I click around there for a bit.

Oh, wait, I probably need to tell glean about the `pings.yaml`, since I needed to tell it about `metrics.yaml`. Ok,
I find this: https://mozilla.github.io/glean/book/reference/pings/index.html

I add my pings.yaml to it. It complains: `Ping uses a reserved name (['baseline', 'metrics', 'events', 'deletion-request', 'default'])`
so I remove my own `events` ping. I leave the schema in there. It fails. I empty the file. It does not like
empty ping files.

Why doesn't it know about its own built-in ping? Why do I need to care about pings at all?

Is there no "here's an end to end working example of glean in an app" in the docs?

At this point I put it down for the day

## The Next Day

I went and read through the Mozilla VPN source a bunch to see if I was missing anything. After a while I noticed
that they were assigning all of their events to a custom `main` ping that they had defined, so I rewrote my
`pings.yaml` file and added

```yaml
send_in_pings:
  - main
```

Back to my event. I ran the code and it still didn't work. Then I realized that I had commented out my call
to `load_pings`, so I uncommented it and huzzah:

```
INFO:glean:Ping c996b31d-86b2-4584-8639-17424193d328 successfully sent 200.
INFO:glean:File was deleted .glean/pending_pings/c996b31d-86b2-4584-8639-17424193d328
INFO:glean:No more pings to upload! You are done.
```

Some observations,
* It didn't complain when I was trying to record an event on a ping that didn't exist.
* Whatever the default `events` ping is, it doesn't work.
* Seems like you can't have events without worrying about pings.

## Trying to validate the data
I don't see my app in the glean debug pings viewer. Did I just send production data? How would I find out?

I search the Glean book for things like "BigQuery," "dataset," "table," etc. There's nothing here that
would tell me how to look for the data I just submitted. (I suspect there's a batch ETL to contend with,
but let's see if I can figure out where I'm supposed to be waiting, at least.)

I leave the Glean book and go to DTMO and find:
https://docs.telemetry.mozilla.org/cookbooks/accessing_glean_data.html

(Why are the Glean data docs separate from the Glean book? This is the whole point of Glean!)

Ok, from this it looks like there are assumed to be datasets like: `org_mozilla_fenix`. (This page doesn't
specify which GCP project, but I know it's probably `mozdata`.)

I don't have a dataset for my app here, which I can agree is a good thing, since I'm just some yahoo
with a python app. So I take a look at the definition of one of the views in `org_mozilla_fenix` and
try to start working my way back. I go to `moz-fx-data-shared-prod`, and hit a dead end when I get
to a real table.

I switch gears and read through `bigquery-etl` and `gcp-ingestion` source for a while, but at this
point I think that

I switch gears and go look at `bigquery-etl`. Too much complexity there for me to figure out I think.
I decide to go read the ingestion code instead. I think what this is telling me is that the payload
is going to wind up on a pubsub topic.

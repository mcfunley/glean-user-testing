I have used a lot of instrumentation libraries before, so I'm going to approach
Glean the way I normally would: slightly haphazardly.

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

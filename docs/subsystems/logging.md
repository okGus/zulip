# Logging and error reporting

Having a good system for logging error reporting is essential to
making a large project like Zulip successful. Without reliable error
reporting, one has to rely solely on bug reports from users in order
to produce a working product.

Our goal as a project is to have zero known 500 errors on the backend
and zero known JavaScript exceptions on the frontend. While there
will always be new bugs being introduced, that goal is impossible
without an efficient and effective error reporting framework.

We provide integration with [Sentry][sentry] to make it easier for
very large installations like zulip.com to manage their exceptions and
ensure they are all tracked down, but our default email-based system
is great for small installations.

## Backend error reporting

The [Django][django-errors] framework provides much of the
infrastructure needed by our error reporting system:

- The ability to send emails to the server's administrators with any
  500 errors, using `django.utils.log.AdminEmailHandler`.
- The ability to rate-limit certain errors to avoid sending hundreds
  of emails in an outage (see `_RateLimitFilter` in
  `zerver/lib/logging_util.py`)
- A nice framework for filtering passwords and other important user
  data from the exception details, which we use in
  `zerver/filters.py`.
- Middleware for handling `JsonableError`, which is our standard
  system for API code to return a JSON-format HTTP error response.

Since 500 errors in any Zulip server are usually a problem the server
administrator should investigate and/or report upstream, we have this
email reporting system configured to report errors by default.

[django-errors]: https://docs.djangoproject.com/en/5.0/howto/error-reporting/

### Sentry error logging

Zulip's optional backend [Sentry][sentry] integration will aggregate
errors to show which users and realms are affected, any logging which
happened prior to the exception, local variables in each frame of the
exception, and the full request headers which triggered it.

You can enable it by:

1.  Create a [project][sentry-project] in your Sentry organization
    with a platform of "Django."
2.  Copy your [Sentry DSN][sentry-dsn] into `/etc/zulip/settings.py`
    as `SENTRY_DSN`:
    ```python3
    ## Controls the DSN used to report errors to Sentry.io
    SENTRY_DSN = "https://bbb@bbb.ingest.sentry.io/1234"
    ```
3.  As the `zulip` user, restart Zulip by running:
    ```shell
    /home/zulip/deployments/current/scripts/restart-server
    ```

You may also want to enable Zulip's [Sentry deploy
hook][sentry-deploy-hook].

[sentry-project]: https://docs.sentry.io/product/projects/
[sentry-dsn]: https://docs.sentry.io/product/sentry-basics/dsn-explainer/
[sentry-deploy-hook]: ../production/deployment.md#sentry-deploy-hook

### Backend logging

[Django's logging system][django-logging] uses the standard
[Python logging infrastructure][python-logging]. We have configured
them so that `logging.exception` and `logging.error` get emailed to
the server maintainer, while `logging.warning` will just appear in
`/var/log/zulip/errors.log`. Lower log levels just appear in the main
server log (as well as in the log for corresponding process, be it
`django.log` for the main Django processes or the appropriate
`events_*` log file for a queue worker).

#### Backend logging format

The main Zulip server log contains a line for each backend request.
It also contains warnings, errors, and the full tracebacks for any
Python exceptions. In production, it goes to
`/var/log/zulip/server.log`; in development, it goes to the terminal
where you run `run-dev`.

In development, it's good to keep an eye on the `run-dev` console
as you work on backend changes, since it's a great way to notice bugs
you just introduced.

In production, one usually wants to look at `errors.log` for errors
since the main server log can be very verbose, but the main server log
can be extremely valuable for investigating performance problems.

```text
2016-05-20 14:50:22.056 INFO [zr] 127.0.0.1       GET     302 528ms (db: 1ms/1q) (+start: 123ms) / (unauth@zulip via ?)
[20/May/2016 14:50:22]"GET / HTTP/1.0" 302 0
2016-05-20 14:50:22.272 INFO [zr] 127.0.0.1       GET     200 124ms (db: 3ms/2q) /login/ (unauth@zulip via ?)
2016-05-20 14:50:26.333 INFO [zr] 127.0.0.1       POST    302  37ms (db: 6ms/7q) /accounts/login/local/ (unauth@zulip via ?)
[20/May/2016 14:50:26]"POST /accounts/login/local/ HTTP/1.0" 302 0
2016-05-20 14:50:26.538 INFO [zr] 127.0.0.1       POST    200  12ms (db: 1ms/2q) (+start: 53ms) /api/v1/events/internal [1463769771:0/0] (8@zulip via internal)
2016-05-20 14:50:26.657 INFO [zr] 127.0.0.1       POST    200  10ms (+start: 8ms) /api/v1/events/internal [1463769771:0/0] (8@zulip via internal)
2016-05-20 14:50:26.959 INFO [zr] 127.0.0.1       GET     200 588ms (db: 26ms/21q) / [1463769771:0] (8@zulip via website)
```

The format of this output is:

- Timestamp
- Log level
- Logger name, abbreviated as "zr" for these Zulip request logs
- IP address
- HTTP method
- HTTP status code
- Time to process
- (Optional perf data details, e.g., database time/queries, memcached
  time/queries, Django process startup time, Markdown processing time,
  etc.)
- Endpoint/URL from zproject/urls.py
- "email via client" showing user account involved (if logged in) and
  the type of client they used ("web", "Android", etc.).

The performance data details are particularly useful for investigating
performance problems, since one can see at a glance whether a slow
request was caused by delays in the database, in the Markdown
processor, in memcached, or in other Python code.

One useful thing to note, however, is that the database time is only
the time spent connecting to and receiving a response from the
database. Especially when response are large, there can often be a
great deal of Python processing overhead to marshall the data from the
database into Django objects that is not accounted for in these
numbers.

#### Searching backend log files

Zulip comes with a tool, `./scripts/log-search`, to quickly search
through the main `server.log` log file based on a number of different
axes -- including client IP address, client user-id, request path,
response code. It can also search the NGINX logs, which provide
similar information.

Because often the requests to static resources, or to things like user
avatars, are not as important, they are also filtered out of the
output by default.

The output shows timestamp, request duration, client IP address,
response code, and request method, hostname, and path; any property
which is limited by the tool is not displayed, for conciseness:

```
zulip@example-prod:~/deployments/current$ ./scripts/log-search realm-name
22:30:36.593     1ms         2606:2800:220:1:248:1893:25c8:1946       302 GET    /
22:30:42.508   366ms         2606:2800:220:1:248:1893:25c8:1946       200 GET    /login/
23:18:30.977     1ms         93.184.216.34                            302 GET    /
23:18:31.286   132ms         93.184.216.34                            200 GET    /login/
23:18:48.094    20ms         93.184.216.34                            200 GET    /accounts/password/reset/
23:18:51.520   149ms         93.184.216.34                            200 GET    /login/
23:18:59.420    20ms         93.184.216.34                            200 GET    /accounts/password/reset/
23:19:02.929  1300ms         93.184.216.34                            302 POST   /accounts/password/reset/
23:19:03.056    93ms         93.184.216.34                            200 GET    /accounts/password/reset/done/
23:19:08.911    26ms         93.184.216.34                            302 GET    /accounts/password/reset/OA/b56jfp-bd80ee99b98e703456b3bdcd91892be2/
23:19:09.189   116ms         93.184.216.34                            200 GET    /accounts/password/reset/OA/set-password/
23:19:18.743   215ms         93.184.216.34                            302 POST   /accounts/password/reset/OA/set-password/
23:19:18.779    12ms         93.184.216.34                            200 GET    /accounts/password/done/
23:19:20.796    12ms         93.184.216.34                            200 GET    /accounts/login/
23:19:29.323   295ms 8@      93.184.216.34                            302 POST   /accounts/login/
23:19:29.704   362ms 8@      93.184.216.34                            200 GET    /
23:20:04.980   110ms 8@      93.184.216.34                            200 DELETE /json/users/me/subscriptions

zulip@example-prod:~/deployments/current$ ./scripts/log-search 2606:2800:220:1:248:1893:25c8:1946
22:30:36.593     1ms          302 GET    https://realm-one.example-prod.example.com/
22:30:42.508   366ms          200 GET    https://realm-one.example-prod.example.com/login/
```

See `./scripts/log-search --help` for complete documentation.

## Blueslip frontend error reporting

We have a custom library, called `blueslip` (named after the form used at MIT to
report problems with the facilities), that logs and reports JavaScript errors in
the browser. In production, this uses Sentry (if configured) to report and
aggregate errors. In development, this means displaying a highly visible overlay
over the message view area, to make exceptions in testing a new feature hard to
miss.

- Blueslip is implemented in `web/src/blueslip.ts`.
- In order to capture essentially any error occurring in the browser, in
  development mode Blueslip listens for the `error` event on `window`.
- It also has methods for being manually triggered by Zulip JavaScript code for
  warnings and assertion failures. Explicit `blueslip.error` calls are sent to
  Sentry, if configured.
- Blueslip keeps a log of all the notices it has received during a browser
  session, so that one can see cases where exceptions chained together. You can
  print this log from the browser console using
  `blueslip = require("./src/blueslip"); blueslip.get_log()`.

Blueslip supports several error levels:

- `throw new Error(…)`: For fatal errors that cannot be easily recovered
  from. We try to avoid using it, since it kills the current JS thread, rather
  than returning execution to the caller.
- `blueslip.error`: For logging of events that are definitely caused by a bug
  and thus sufficiently important to be reported, but where we can handle the
  error without creating major user-facing problems (e.g., an exception when
  handling a presence update).
- `blueslip.warn`: For logging of events that are a problem but not important
  enough to log an error to Sentry in production. They are, however, highlighted
  in the JS console in development, and appear as breadcrumb logs in Sentry in
  production.
- `blueslip.log` (and `blueslip.info`): Logged to the JS console in development
  and also in the Sentry breadcrumb log in production. Useful for data that
  might help discern what state the browser was in during an error (e.g., whether
  the user was in a narrow).
- `blueslip.debug`: Similar to `blueslip.log`, but are not printed to
  the JS console in development.

### Sentry JavaScript error logging

Zulip's optional JavaScript [Sentry][sentry] integration will aggregate errors
to show which users and realms are affected, any logging which happened prior to
the exception, and any DOM interactions which happened prior to the error.

You can enable it by:

1.  Create a [project][sentry-project] in your Sentry organization
    with a platform of "JavaScript."
2.  Copy your [Sentry DSN][sentry-dsn] into `/etc/zulip/settings.py`
    as `SENTRY_FRONTEND_DSN`:
    ```python3
    ## Controls the DSN used to report JavaScript errors to Sentry.io
    SENTRY_FRONTEND_DSN = "https://bbb@bbb.ingest.sentry.io/1234"
    ```
3.  If you wish to [sample][sentry-sample] some fraction of the errors, you
    should adjust `SENTRY_FRONTEND_SAMPLE_RATE` down from `1.0`.
4.  As the `zulip` user, restart Zulip by running:
    ```shell
    /home/zulip/deployments/current/scripts/restart-server
    ```

You may also want to enable Zulip's [Sentry deploy
hook][sentry-deploy-hook].

[sentry-sample]: https://docs.sentry.io/platforms/javascript/configuration/sampling/

## Frontend performance reporting

In order to make it easier to debug potential performance problems in
the critically latency-sensitive message sending code pathway, we log
and report to the server the following whenever a message is sent:

- The time the user triggered the message (aka the start time).
- The time the `send_message` response returned from the server.
- The time the message was received by the browser from the
  `get_events` protocol (these last two race with each other).
- Whether the message was locally echoed.
- If so, whether there was a disparity between the echoed content and
  the server-rendered content, which can be used for statistics on how
  effective our [local echo system](markdown.md) is.

The code is all in `zerver/lib/report.py` and `web/src/sent_messages.ts`.

We have similar reporting for the time it takes to narrow / switch to
a new view:

- The time the action was initiated
- The time when the updated message feed was visible to the user
- The time when the browser was idle again after switching views
  (intended to catch issues where we generate a lot of deferred work).

[python-logging]: https://docs.python.org/3/library/logging.html
[django-logging]: https://docs.djangoproject.com/en/5.0/topics/logging/
[sentry]: https://sentry.io

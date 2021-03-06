# New Relic for Node.js

**NOTE:** If you've used previous versions of New Relic for Node.js, you should
read the section on [transactions and request
naming](#transactions-and-request-naming), and some important things have
changed in the [configuration](#configuring-the-agent), so you'll want to read
that section again as well.

This package instruments your application for performance monitoring
with [New Relic](http://newrelic.com).

This is a **beta release**. It has known issues that may affect your
application's stability. You should try it in your staging or development
environment first to verify it works for you.

Make sure you have a [New Relic account](http://newrelic.com) before
starting. To see all the features, such as slow transaction traces, you will
need a New Relic Pro subscription. Contact your New Relic representative to
request a Pro Trial subscription during your beta testing.

## Table of contents

* [Support](#support)
* [Getting started](#getting-started)
* [Transactions and request naming](#transactions-and-request-naming)
* [Configuration](#configuring-the-agent)
* [Known issues](#known-issues)

## Support

New Relic for Node.js is currently in beta and has only limited support.
Customers who tried it out during the open beta period are welcome to continue
using it and will receive support, but any new deployments may not receive
technical support, or receive only limited assistance. We're very close to a
wider public release, but we're not quite there yet!

We're just as eager as you are to see Node and New Relic live happily ever
after and we're 100% committed to it. We expect to open up a new beta very
soon. If you'd like to know when the agent is ready for release, please [sign
up](http://try.newrelic.com/nodejs) to be notified.

## Getting started

1. [Install node](http://nodejs.org/#download). For now, at least 0.8 is
   required. Some features (e.g. error tracing) depend in whole or in
   part on features in 0.10 and above. Development work on the agent is
   being done against the latest released non-development version of Node.
2. Install this module via `npm install newrelic` for the application you
   want to monitor.
3. Copy `newrelic.js` from `node_modules/newrelic` into the root directory of
   your application.
4. Edit `newrelic.js` and replace `license_key`'s value with the license key
   for your account.
5. Add `require('newrelic');` as the first line of the app's main module.

If you wish to keep the configuration for the agent separate from your
application, the agent will look for newrelic.js in the directory referenced
by the environment variable `NEW_RELIC_HOME` if it's set.

When you start your app, the agent should start up with it and start
reporting data that will appear within [the New Relic
UI](https://rpm.newrelic.com/) after a few minutes. Because the agent
minimizes the amount of bandwidth it consumes, it only reports data once a
minute, so if you add the agent to tests that take less than a minute to run,
the agent won't have time to report data to New Relic. The agent will write
its log to a file named `newrelic_agent.log` in the application directory. If
the agent doesn't send data or crashes your app, the log can help New Relic
determine what went wrong, so be sure to send it along with any bug reports
or support requests.

## Transactions and request naming

In order to get the most value out of the New Relic agent for Node.js, you may
have to do a little work to help us figure out how your application is
structured. New Relic works on the assumption that it can group requests to
your application into transactions, which are defined by giving one or more
request paths a name. These names are used to visualize where your app is
spending its time (in transaction breakdowns), to identify slow requests, and
to group scoped metrics, to tell you which portions of your application are,
for example, suffering from slow database performance.

If you're using Express or Restify with their default routers and are satisfied
with your application being represented by those frameworks' route names, you
may not need to do anything. However, if you want more specific names than are
provided by your framework, you may want to use one or more of the tools
described further on.

The simplest way to tell that you need to read further in this document is if
you feel too many of your requests are being lumped together under the
catch-all name `/*`. All requests that aren't otherwise named by the agent will
end up grouped under `/*`.

### Background

If you've been working with Node for a while, you're probably accustomed to
thinking of your application's requests in terms of raw URLs. One of the great
things about Node is that it makes it so easy and simple to work with HTTP, and
that extends to things like parsing URLs and creating your own strategies for
naming and routing requests for services like RESTful APIs. This presents a
challenge for us, because we need to keep the number of names we're tracking
small enough that we can keep the New Relic user experience snappy, and also so
we don't overwhelm you with so much data that it's difficult for you to see the
problem spots in your applications. URLs are not a good fit for how New Relic
sees performance.

Another of Node's great strengths is that it provides a lot of tools that build
on top of the `http` module to simplify writing web services. Unfortunately,
that variety greatly complicates things for us, with our limited resources, and
so we offer a few different tools to help you give us the information we need
to provide you useful metrics about your application:

* we can read the route names from the Express and Restify routers, if you're
  using them (and as said above, for many of you, this may be all you need)
* we offer an API for naming the current request, either with simple names or,
  if you prefer, grouped into controllers with actions
* and we support rules stored in your agent's configuration that can mark
  requests to be renamed or ignored based on regular expressions matched
  against the request's raw URLs (also available as API calls)

Let's go through those tools one at a time.

### Router introspection

Express is the most popular web framework in use within the Node community, and
a number of important services are also using Restify. Both frameworks map
routes to handlers, and both use a similar pattern to do so: they match one or
more HTTP methods (e.g. `GET` or the ever-popular `OPTIONS` – let's hear it
for CORS) along with a potentially parameterized path (e.g. `/user/:id`) or a
regular expression (e.g.  `/^/user/([-0-9a-f]+)$/`). The New Relic agent will
capture both those pieces of information in the request name. If you have
support for slow transaction traces and have enabled `capture_params`, the
transaction trace will also have the request's parameters and their values
attached to it.

The only important thing to know about New Relic's support for Express and
Restify is that if you're dissatisfied with the names it comes up with, you can
use the API calls described below to come up with more descriptive names. Also,
if you use a different web framework or router and would like to see support
for it added, please let us know.

### The request naming API

The API is what's handed back from `require('newrelic')`, so

```javascript
var newrelic = require('newrelic');
```

is all you need. Please note that you still need to ensure that loading the New
Relic module is the first thing your application does, as it needs to bootstrap
itself before the rest of your application loads, but you can safely require
the module from multiple modules in your application – it will only initialize
itself once.

#### newrelic.setTransactionName(name)

Name the current request. You can call this function anywhere within the
context of an HTTP request handler, at any time after handling of the request
has started, but before the request has finished. A good rule of thumb is that
if the request and response objects are in scope, you can set the name.

Explicitly calling `newrelic.setTransactionName()` will override any names set
by Express or Restify routes. Calls to `newrelic.setTransactionName()` and
`newrelic.setControllerName()` will overwrite each other. The last one to run
before the request ends wins.

**VERY IMPORTANT NOTE:** Do not include highly variable information like GUIDs,
numerical IDs, or timestamps in the request names you create. If your request
is slow enough to generate a transaction trace, that trace will contain the
original URL. If you enable parameter capture, the parameters will also be
attached to the trace. The request names are used to group requests for New
Relic's many charts and tables, and those visualizations' value drops as the
number of different request names increases. If you have 50 or so different
transaction names, you're probably pushing it. If you have more than a couple
hundred, you need to rethink your naming strategy.

#### newrelic.setControllerName(name, [action])

Name the current request using a controller-style pattern, optionally including
the current controller action. If the action is omitted, New Relic will include
the HTTP method (e.g. `GET`, `POST`) as the action. The rules for when you can
call `newrelic.setControllerName()` are the same as they are for
`newrelic.setTransactionName()`.

Explicitly calling `newrelic.setControllerName()` will override any names set
by Express or Restify routes. Calls to `newrelic.setTransactionName()` and
`newrelic.setControllerName()` will overwrite each other. The last one to run
before the request ends wins.

See the above note on `newrelic.setTransactionName()`, which also applies to
this function.

### Rules for naming and ignoring requests

If you don't feel like putting calls to the New Relic module directly into your
application code, you can use pattern-based rules to name requests. There are
two sets of rules: one for renaming requests, and one to mark requests to be
ignored by New Relic's instrumentation.

If you're using socket.io, you will have a use case for ignoring rules right
out of the box. You'll probably want to add a rule like the following:

```javascript
// newrelic.js
exports.config = {
  // other configuration
  rules : {
    ignore : [
      '^/socket.io/\*/xhr-polling'
    }
  }
};
```

This will keep socket.io long-polling from dominating your response-time
metrics and blowing out the apdex metrics for your application.

#### rules.name

A list of rules of the format `{pattern : "pattern", name : "name"}` for
matching incoming request URLs to `pattern` and naming the matching New Relic
transactions `name`. The pattern can be set as either a string or a JavaScript
regular expression literal. Both pattern and name are required. Additional
attributes are ignored.

Can also be set via the environment variable `NEW_RELIC_NAMING_RULES`, with
multiple rules passed in as a list of comma-delimited JSON object literals:
`NEW_RELIC_NAMING_RULES='{"pattern":"^t","name":"u"},{"pattern":"^u","name":"t"}'`

#### rules.ignore

A list of patterns for matching incoming request URLs to be ignored. Patterns
may be strings or regular expressions.

Can also be set via the environment variable `NEW_RELIC_IGNORING_RULES`, with
multiple rules passed in as a list of comma-delimited patterns:
`NEW_RELIC_IGNORING_RULES='^/socket\.io/\*/xhr-polling,ignore_me'` Note that
currently there is no way to escape commas in patterns.

### API for adding naming and ignoring rules

#### newrelic.addNamingRule(pattern, name)

Programmatic version of `rules.name` above. Naming rules can not be removed
once added. They can also be added via the agent's configuration. Both
parameters are mandatory.

#### newrelic.addIgnoringRule(pattern)

Programmatic version of `rules.ignore` above. Ignoring rules can not be removed
once added. They can also be added via the agent's configuration. Both
parameters are mandatory.

### The fine print

This is the Node-specific version of New Relic's transaction naming API
documentation. The naming API exists to help us deal with the very real problem
that trying to handle too many metrics will make New Relic slow for everybody,
not just the account with too many metrics. If, in conversation with New Relic
Support, you see discussion of "metric explosion", this is what they're talking
about.

While we have a variety of strategies for dealing with these issues, the most
severe is simply to blacklist offending applications. The main reason for you
to be careful in using our request-naming tools is to prevent that from
happening to your applications. We will do everything in our power to ensure
that you have a good experience with New Relic even if your application is
causing us trouble, but sometimes this will require manual intervention on the
part of our team, and this can take a little while.

## Configuring the agent

The agent can be tailored to your app's requirements, both from the server and
via the newrelic.js configuration file you created. For complete details on
what can be configured, refer to
[`lib/config.default.js`](https://github.com/newrelic/node-newrelic/blob/master/lib/config.default.js),
which documents the available variables and their default values.

In addition, for those of you running in PaaS environments like Heroku or
Microsoft Azure, all of the configuration variables in `newrelic.js` have
counterparts that can be set via environment variables. You can mix and match
variables in the configuration file and environment variables freely;
environment variables take precedence.

Here's the list of the most important variables and their values:

* `NEW_RELIC_LICENSE_KEY`: Your New Relic license key. This is a required
  setting with no default value.
* `NEW_RELIC_APP_NAME`: The name of this application, for reporting to
  New Relic's servers. This value can be also be a comma-delimited list of
  names. This is a required setting with no default value.
* `NEW_RELIC_NO_CONFIG_FILE`: Inhibit loading of the configuration file
  altogether. Use with care. This presumes that all important configuration
  will be available via environment variables, and some log messages
  assume that a config file exists.
* `NEW_RELIC_HOME`: path to the directory in which you've placed newrelic.js.
* `NEW_RELIC_LOG`: Complete path to the New Relic agent log, including
  the filename. The agent will shut down the process if it can't create
  this file, and it creates the log file with the same umask of the
  process. Setting this to `stdout` will write all logging to stdout, and
  `stderr` will write all logging to stderr.
* `NEW_RELIC_LOG_LEVEL`: Logging priority for the New Relic agent. Can be one of
  `error`, `warn`, `info`, `debug`, or `trace`. `debug` and `trace` are
  pretty chatty; unless you're helping New Relic figure out irregularities
  with the agent, you're probably best off using `info` or higher.

For completeness, here's the rest of the list:

* `NEW_RELIC_ENABLED`: Whether or not the agent should run. Good for
  temporarily disabling the agent while debugging other issues with your
  code. It doesn't prevent the agent from bootstrapping its instrumentation
  or setting up all its pieces, it just prevents it from starting up or
  connecting to New Relic's servers. Defaults to true.
* `NEW_RELIC_ERROR_COLLECTOR_ENABLED`: Whether or not to trace errors within
  your application. Values are `true` or `false`. Defaults to true.
* `NEW_RELIC_ERROR_COLLECTOR_IGNORE_ERROR_CODES`: Comma-delimited list of HTTP
  status codes to ignore. Maybe you don't care if payment is required? Defaults
  to ignoring 404.
* `NEW_RELIC_IGNORE_SERVER_CONFIGURATION`: Whether to ignore server-side
  configuration for this application. Defaults to false.
* `NEW_RELIC_TRACER_ENABLED`: Whether to collect and submit slow
  transaction traces to New Relic. Values are `true` or `false`. Defaults to
  true.
* `NEW_RELIC_TRACER_THRESHOLD`: Duration (in seconds) at which a transaction
  trace will count as slow and be sent to New Relic. Can also be set to
  `apdex_f`, at which point it will set the trace threshold to 4 times the
  current ApdexT.
* `NEW_RELIC_APDEX`: Set the initial Apdex tolerating / threshold value.
  This is more often than not set from the server. Defaults to 0.5.
* `NEW_RELIC_CAPTURE_PARAMS`: Whether to capture request parameters on
  slow transaction or error traces. Defaults to false.
* `NEW_RELIC_IGNORED_PARAMS`: Some parameters may contain sensitive
  values you don't want being sent out of your application. This setting
  is a comma-delimited list of names of parameters to ignore. Defaults to
  empty.
* `NEW_RELIC_NAMING_RULES`: A list of comma-delimited JSON object literals:
  `NEW_RELIC_NAMING_RULES='{"pattern":"^t","name":"u"},{"pattern":"^u","name":"t"}'`
  See the section on request and transaction naming for details. Defaults to
  empty.
* `NEW_RELIC_IGNORING_RULES`: A list of comma-delimited patterns:
  `NEW_RELIC_IGNORING_RULES='^/socket\.io/\*/xhr-polling,ignore_me'` Note that
  currently there is no way to escape commas in patterns. Defaults to empty.
* `NEW_RELIC_TRACER_TOP_N`: How many different named requests to track for
  transaction tracing. See the description in `lib/config.default.js`, as this
  feature is exceedingly hard to summarize.
* `NEW_RELIC_HOST`: Hostname for the New Relic collector proxy. You
  shouldn't need to change this.
* `NEW_RELIC_PORT`: Port number on which the New Relic collector proxy
  will be listening. You shouldn't need to change this either.
* `NEW_RELIC_DEBUG_METRICS`: Whether to collect internal supportability
  metrics for the agent. Don't mess with this unless New Relic asks you to.
* `NEW_RELIC_DEBUG_TRACER`: Whether to dump traces of the transaction tracer's
  internal operation. You're welcome to enable it, but it's unlikely to be
  edifying unless you're a New Relic Node.js engineer.

## Recent changes

Information about changes to the agent are in NEWS.md.

### Known issues:

* The agent works only with Node.js 0.6 and newer ( **IMPORTANT**: newer betas
  depend on Node 0.8, and support for 0.6 may or may not come back by the time
  version 1.0 of the New Relic agent is released). Certain features rely on
  Node 0.8. Some features may behave differently between 0.8 and 0.10. The
  agent is optimized for newer versions of Node.
* There are irregularities around transaction trace capture and display.
  If you notice missing or incorrect information from transaction traces,
  let us know.
* There are <del>over 20,000</del> <del>30,000</del> <del>40,000</del> <ins>*A
  LOT* of</ins> modules on npm. We can only instrument a tiny number of them.
  Even for the modules we support, there are a very large number of ways to use
  them. If you see data you don't expect on New Relic and have the time to
  produce a reduced version of the code that is producing the strange data, it
  will be used to improve the agent and you will have the Node team's gratitude.
* There is an error tracer in the Node agent, but it's a work in progress.
  In particular, it still does not intercept errors that may already be
  handled by frameworks. Also, parts of it depend on the
  [domain](http://nodejs.org/api/domain.html) API added in Node 0.8, and
  domain-specific functionality will not work in apps running in
  Node 0.6.x.
* The CPU and memory overhead incurred by the Node agent is relatively minor
  (~1-10%, depending on how much of the instrumentation your apps end up
  using).  GC activity is significantly increased while the agent is active,
  due to the large number of ephemeral objects created by metrics gathering.
* When using Node's included clustering support, each worker process will
  open its own connection to New Relic's servers, and will incur its own
  overhead costs.

### New Relic features available for other platforms not yet in Node.js

* Real User Monitoring (RUM)
* custom instrumentation APIs
* slow SQL traces and explain plans
* custom parameters
* garbage collector instrumentation
* capacity planning
* thread profiling

## LICENSE

The New Relic Node.js agent uses code from the following open source projects
under the following licenses:

    bunyan                      http://opensource.org/licenses/MIT
    continuation-local-storage  http://opensource.org/licenses/BSD-3-Clause

The New Relic Node.js agent itself is free-to-use, proprietary software.
Please see the full license (found in LICENSE in this distribution) for
details.

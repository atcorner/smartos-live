# bunyan Changelog

Known issues:

- [issue #58] Can't install to a dir with spaces. This is [this node-gyp
  bug](https://github.com/TooTallNate/node-gyp/issues/65).


## bunyan 0.18.1

- Get the `bunyan` CLI to **not** automatically page (i.e. pipe to `less`)
  if stdin isn't a TTY, or if following dtrace probe output (via `-p PID`),
  or if not given log file arguments.


## bunyan 0.18.0

- Automatic paging support in the `bunyan` CLI (similar to `git log` et al).
  IOW, `bunyan` will open your pager (by default `less`) and pipe rendered
  log output through it. A main benefit of this is getting colored logs with
  a pager without the pain. Before you had to explicit use `--color` to tell
  bunyan to color output when the output was not a TTY:

        bunyan foo.log --color | less -R        # before
        bunyan foo.log                          # now

  Disable with the `--no-pager` option or the `BUNYAN_NO_PAGER=1` environment
  variable.

  Limitations: Only supported for node >=0.8. Windows is not supported (at
  least not yet).

- Switch test suite to nodeunit (still using a node-tap'ish API via
  a helper).


## bunyan 0.17.0

- [issue #33] Log rotation support:

        var bunyan = require('bunyan');
        var log = bunyan.createLogger({
            name: 'myapp',
            streams: [{
                type: 'rotating-file',
                path: '/var/log/myapp.log',
                count: 7,
                period: 'daily'
            }]
        });


- Tweak to CLI default pretty output: don't special case "latency" field.
  The special casing was perhaps nice, but less self-explanatory.
  Before:

        [2012-12-27T21:17:38.218Z]  INFO: audit/45769 on myserver: handled: 200 (15ms, audit=true, bar=baz)
          GET /foo
          ...

  After:

        [2012-12-27T21:17:38.218Z]  INFO: audit/45769 on myserver: handled: 200 (audit=true, bar=baz, latency=15)
          GET /foo
          ...

- *Exit* CLI on EPIPE, otherwise we sit there useless processing a huge log
  file with, e.g.  `bunyan huge.log | head`.


## bunyan 0.16.8

- Guards on `-c CONDITION` usage to attempt to be more user friendly.
  Bogus JS code will result in this:

        $ bunyan portal.log -c 'this.req.username==boo@foo'
        bunyan: error: illegal CONDITION code: SyntaxError: Unexpected token ILLEGAL
          CONDITION script:
            Object.prototype.TRACE = 10;
            Object.prototype.DEBUG = 20;
            Object.prototype.INFO = 30;
            Object.prototype.WARN = 40;
            Object.prototype.ERROR = 50;
            Object.prototype.FATAL = 60;
            this.req.username==boo@foo
          Error:
            SyntaxError: Unexpected token ILLEGAL
                at new Script (vm.js:32:12)
                at Function.Script.createScript (vm.js:48:10)
                at parseArgv (/Users/trentm/tm/node-bunyan-0.x/bin/bunyan:465:27)
                at main (/Users/trentm/tm/node-bunyan-0.x/bin/bunyan:1252:16)
                at Object.<anonymous> (/Users/trentm/tm/node-bunyan-0.x/bin/bunyan:1330:3)
                at Module._compile (module.js:449:26)
                at Object.Module._extensions..js (module.js:467:10)
                at Module.load (module.js:356:32)
                at Function.Module._load (module.js:312:12)
                at Module.runMain (module.js:492:10)

  And all CONDITION scripts will be run against a minimal valid Bunyan
  log record to ensure they properly guard against undefined values
  (at least as much as can reasonably be checked). For example:

        $ bunyan portal.log -c 'this.req.username=="bob"'
        bunyan: error: CONDITION code cannot safely filter a minimal Bunyan log record
          CONDITION script:
            Object.prototype.TRACE = 10;
            Object.prototype.DEBUG = 20;
            Object.prototype.INFO = 30;
            Object.prototype.WARN = 40;
            Object.prototype.ERROR = 50;
            Object.prototype.FATAL = 60;
            this.req.username=="bob"
          Minimal Bunyan log record:
            {
              "v": 0,
              "level": 30,
              "name": "name",
              "hostname": "hostname",
              "pid": 123,
              "time": 1355514346206,
              "msg": "msg"
            }
          Filter error:
            TypeError: Cannot read property 'username' of undefined
                at bunyan-condition-0:7:9
                at Script.Object.keys.forEach.(anonymous function) [as runInNewContext] (vm.js:41:22)
                at parseArgv (/Users/trentm/tm/node-bunyan-0.x/bin/bunyan:477:18)
                at main (/Users/trentm/tm/node-bunyan-0.x/bin/bunyan:1252:16)
                at Object.<anonymous> (/Users/trentm/tm/node-bunyan-0.x/bin/bunyan:1330:3)
                at Module._compile (module.js:449:26)
                at Object.Module._extensions..js (module.js:467:10)
                at Module.load (module.js:356:32)
                at Function.Module._load (module.js:312:12)
                at Module.runMain (module.js:492:10)

  A proper way to do that condition would be:

        $ bunyan portal.log -c 'this.req && this.req.username=="bob"'



## bunyan 0.16.7

- [issue #59] Clear a possibly interrupted ANSI color code on signal
  termination.


## bunyan 0.16.6

- [issue #56] Support `bunyan -p NAME` to dtrace all PIDs matching 'NAME' in
  their command and args (using `ps -A -o pid,command | grep NAME` or, on SunOS
  `pgrep -lf NAME`). E.g.:

        bunyan -p myappname

  This is useful for usage of node's [cluster
  module](http://nodejs.org/docs/latest/api/all.html#all_cluster) where you'll
  have multiple worker processes.


## bunyan 0.16.5

- Allow `bunyan -p '*'` to capture bunyan dtrace probes from **all** processes.
- issue #55: Add support for `BUNYAN_NO_COLOR` environment variable to
  turn off all output coloring. This is still overridden by the `--color`
  and `--no-color` options.


## bunyan 0.16.4

- issue #54: Ensure (again, see 0.16.2) that stderr from the dtrace child
  process (when using `bunyan -p PID`) gets through. There had been a race
  between exiting bunyan and the flushing of the dtrace process' stderr.


## bunyan 0.16.3

- Drop 'trentm-dtrace-provider' fork dep now that
  <https://github.com/chrisa/node-dtrace-provider/pull/24> has been resolved.
  Back to dtrace-provider.


## bunyan 0.16.2

- Ensure that stderr from the dtrace child process (when using `bunyan -p PID`)
  gets through. The `pipe` usage wasn't working on SmartOS. This is important
  to show the user if they need to 'sudo'.


## bunyan 0.16.1

- Ensure that a possible dtrace child process (with using `bunyan -p PID`) is
  terminated on signal termination of the bunyan CLI (at least for SIGINT,
  SIGQUIT, SIGTERM, SIGHUP).


## bunyan 0.16.0

- Add `bunyan -p PID` support. This is a convenience wrapper that effectively
  calls:

        dtrace -x strsize=4k -qn 'bunyan$PID:::log-*{printf("%s", copyinstr(arg0))}' | bunyan


## bunyan 0.15.0

- issue #48: Dtrace support! The elevator pitch is you can watch all logging
  from all Bunyan-using process with something like this:

        dtrace -x strsize=4k -qn 'bunyan*:::log-*{printf("%d: %s: %s", pid, probefunc, copyinstr(arg0))}'

  And this can include log levels *below* what the service is actually configured
  to log. E.g. if the service is only logging at INFO level and you need to see
  DEBUG log messages, with this you can. Obviously this only works on dtrace-y
  platforms: Illumos derivatives of SunOS (e.g. SmartOS, OmniOS), Mac, FreeBSD.

  Or get the bunyan CLI to render logs nicely:

        dtrace -x strsize=4k -qn 'bunyan*:::log-*{printf("%s", copyinstr(arg0))}' | bunyan

  See <https://github.com/trentm/node-bunyan#dtrace-support> for details. By
  Bryan Cantrill.


## bunyan 0.14.6

- Export `bunyan.safeCycles()`. This may be useful for custom `type == "raw"`
  streams that may do JSON stringification of log records themselves. Usage:

        var str = JSON.stringify(rec, bunyan.safeCycles());

- [issue #49] Allow a `log.child()` to specify the level of inherited streams.
  For example:

        # Before
        var childLog = log.child({...});
        childLog.level('debug');

        # After
        var childLog = log.child({..., level: 'debug'});

- Improve the Bunyan CLI crash message to make it easier to provide relevant
  details in a bug report.


## bunyan 0.14.5

- Fix a bug in the long-stack-trace error serialization added in 0.14.4. The
  symptom:

        bunyan@0.14.4: .../node_modules/bunyan/lib/bunyan.js:1002
          var ret = ex.stack || ex.toString();
                      ^
        TypeError: Cannot read property 'stack' of undefined
            at getFullErrorStack (.../node_modules/bunyan/lib/bunyan.js:1002:15)
            ...


## bunyan 0.14.4

- **Bad release. Use 0.14.5 instead.**
- Improve error serialization to walk the chain of `.cause()` errors
  from the likes of `WError` or `VError` error classes from
  [verror](https://github.com/davepacheco/node-verror) and
  [restify v2.0](https://github.com/mcavage/node-restify). Example:

        [2012-10-11T00:30:21.871Z] ERROR: imgapi/99612 on 0525989e-2086-4270-b960-41dd661ebd7d: my-message
            ValidationFailedError: my-message; caused by TypeError: cause-error-message
                at Server.apiPing (/opt/smartdc/imgapi/lib/app.js:45:23)
                at next (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:550:50)
                at Server.setupReq (/opt/smartdc/imgapi/lib/app.js:178:9)
                at next (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:550:50)
                at Server.parseBody (/opt/smartdc/imgapi/node_modules/restify/lib/plugins/body_parser.js:15:33)
                at next (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:550:50)
                at Server.parseQueryString (/opt/smartdc/imgapi/node_modules/restify/lib/plugins/query.js:40:25)
                at next (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:550:50)
                at Server._run (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:579:17)
                at Server._handle.log.trace.req (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:480:38)
            Caused by: TypeError: cause-error-message
                at Server.apiPing (/opt/smartdc/imgapi/lib/app.js:40:25)
                at next (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:550:50)
                at Server.setupReq (/opt/smartdc/imgapi/lib/app.js:178:9)
                at next (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:550:50)
                at Server.parseBody (/opt/smartdc/imgapi/node_modules/restify/lib/plugins/body_parser.js:15:33)
                at next (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:550:50)
                at Server.parseQueryString (/opt/smartdc/imgapi/node_modules/restify/lib/plugins/query.js:40:25)
                at next (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:550:50)
                at Server._run (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:579:17)
                at Server._handle.log.trace.req (/opt/smartdc/imgapi/node_modules/restify/lib/server.js:480:38)


## bunyan 0.14.2

- [issue #45] Fix bunyan CLI (default output mode) to not crash on a 'res'
  field that isn't a response object, but a string.


## bunyan 0.14.1

- [issue #44] Fix the default `bunyan` CLI output of a `res.body` that is an
  object instead of a string. See issue#38 for the same with `req.body`.


## bunyan 0.14.0

- [pull #41] Safe `JSON.stringify`ing of emitted log records to avoid blowing
  up on circular objects (by Isaac Schlueter).


## bunyan 0.13.5

- [issue #39] Fix a bug with `client_req` handling in the default output
  of the `bunyan` CLI.


## bunyan 0.13.4

- [issue #38] Fix the default `bunyan` CLI output of a `req.body` that is an
  object instead of a string.


## bunyan 0.13.3

- Export `bunyan.resolveLevel(NAME-OR-NUM)` to resolve a level name or number
  to its log level number value:

        > bunyan.resolveLevel('INFO')
        30
        > bunyan.resolveLevel('debug')
        20

  A side-effect of this change is that the uppercase level name is now allowed
  in the logger constructor.


## bunyan 0.13.2

- [issue #35] Ensure that an accidental `log.info(BUFFER)`, where BUFFER is
  a node.js Buffer object, doesn't blow up.


## bunyan 0.13.1

- [issue #34] Ensure `req.body`, `res.body` and other request/response fields
  are emitted by the `bunyan` CLI (mostly by Rob Gulewich).



## bunyan 0.13.0

- [issue #31] Re-instate defines for the (uppercase) log level names (TRACE,
  DEBUG, etc.) in `bunyan -c "..."` filtering condition code. E.g.:

        $ ... | bunyan -c 'level >= ERROR'


## bunyan 0.12.0

- [pull #32] `bunyan -o short` for more concise output (by Dave Pacheco). E.g.:

        22:56:52.856Z  INFO myservice: My message

  instead of:

        [2012-02-08T22:56:52.856Z]  INFO: myservice/123 on example.com: My message


## bunyan 0.11.3

- Add '--strict' option to `bunyan` CLI to suppress all but legal Bunyan JSON
  log lines. By default non-JSON, and non-Bunyan lines are passed through.


## bunyan 0.11.2

- [issue #30] Robust handling of 'req' field without a 'headers' subfield
  in `bunyan` CLI.
- [issue #31] Pull the TRACE, DEBUG, et al defines from `bunyan -c "..."`
  filtering code. This was added in v0.11.1, but has a significant adverse
  affect.


## bunyan 0.11.1

- **Bad release. The TRACE et al names are bleeding into the log records
  when using '-c'.**
- Add defines for the (uppercase) log level names (TRACE, DEBUG, etc.) in
  `bunyan -c "..."` filtering condition code. E.g.:

        $ ... | bunyan -c 'level >= ERROR'


## bunyan 0.11.0

- [pull #29] Add -l/--level for level filtering, and -c/--condition for
  arbitrary conditional filtering (by github.com/isaacs):

        $ ... | bunyan -l error   # filter out log records below error
        $ ... | bunyan -l 50      # numeric value works too
        $ ... | bunyan -c 'level===50'              # equiv with -c filtering
        $ ... | bunyan -c 'pid===123'               # filter on any field
        $ ... | bunyan -c 'pid===123' -c '_audit'   # multiple filters


## bunyan 0.10.0

- [pull #24] Support for gzip'ed log files in the bunyan CLI (by
  github.com/mhart):

        $ bunyan foo.log.gz
        ...


## bunyan 0.9.0

- [pull #16] Bullet proof the `bunyan.stdSerializers` (by github.com/rlidwka).

- [pull #15] The `bunyan` CLI will now chronologically merge multiple log
  streams when it is given multiple file arguments. (by github.com/davepacheco)

        $ bunyan foo.log bar.log
        ... merged log records ...

- [pull #15] A new `bunyan.RingBuffer` stream class that is useful for
  keeping the last N log messages in memory. This can be a fast way to keep
  recent, and thus hopefully relevant, log messages. (by @dapsays,
  github.com/davepacheco)

  Potential uses: Live debugging if a running process could inspect those
  messages. One could dump recent log messages at a finer log level than is
  typically logged on
  [`uncaughtException`](http://nodejs.org/docs/latest/api/all.html#all_event_uncaughtexception).

        var ringbuffer = new bunyan.RingBuffer({ limit: 100 });
        var log = new bunyan({
            name: 'foo',
            streams: [{
                type: 'raw',
                stream: ringbuffer,
                level: 'debug'
            }]
        });

        log.info('hello world');
        console.log(ringbuffer.records);

- Add support for "raw" streams. This is a logging stream that is given
  raw log record objects instead of a JSON-stringified string.

        function Collector() {
            this.records = [];
        }
        Collector.prototype.write = function (rec) {
            this.records.push(rec);
        }
        var log = new Logger({
            name: 'mylog',
            streams: [{
                type: 'raw',
                stream: new Collector()
            }]
        });

  See "examples/raw-stream.js". I expect raw streams to be useful for
  piping Bunyan logging to separate services (e.g. <http://www.loggly.com/>,
  <https://github.com/etsy/statsd>) or to separate in-process handling.

- Add test/corpus/*.log files (accidentally excluded) so the test suite
  actually works(!).


## bunyan 0.8.0

- [pull #21] Bunyan loggers now re-emit `fs.createWriteStream` error events.
  By github.com/EvanOxfeld. See "examples/handle-fs-error.js" and
  "test/error-event.js" for details.

        var log = new Logger({name: 'mylog', streams: [{path: FILENAME}]});
        log.on('error', function (err, stream) {
            // Handle error writing to or creating FILENAME.
        });

- jsstyle'ing (via `make check`)


## bunyan 0.7.0

- [issue #12] Add `bunyan.createLogger(OPTIONS)` form, as is more typical in
  node.js APIs.  This'll eventually become the preferred form.


## bunyan 0.6.9

- Change `bunyan` CLI default output to color "src" info red. Before the "src"
  information was uncolored. The "src" info is the filename, line number and
  function name resulting from using `src: true` in `Logger` creation. I.e.,
  the `(/Users/trentm/tm/node-bunyan/examples/hi.js:10)` in:

        [2012-04-10T22:28:58.237Z]  INFO: myapp/39339 on banana.local (/Users/trentm/tm/node-bunyan/examples/hi.js:10): hi

- Tweak `bunyan` CLI default output to still show an "err" field if it doesn't
  have a "stack" attribute.


## bunyan 0.6.8

- Fix bad bug in `log.child({...}, true);` where the added child fields **would
  be added to the parent's fields**. This bug only existed for the "fast child"
  path (that second `true` argument). A side-effect of fixing this is that
  the "fast child" path is only 5 times as fast as the regular `log.child`,
  instead of 10 times faster.


## bunyan 0.6.7

- [issue #6] Fix bleeding 'type' var to global namespace. (Thanks Mike!)


## bunyan 0.6.6

- Add support to the `bunyan` CLI taking log file path args, `bunyan foo.log`,
  in addition to the usual `cat foo.log | bunyan`.
- Improve reliability of the default output formatting of the `bunyan` CLI.
  Before it could blow up processing log records missing some expected
  fields.


## bunyan 0.6.5

- ANSI coloring output from `bunyan` CLI tool (for the default output mode/style).
  Also add the '--color' option to force coloring if the output stream is not
  a TTY, e.g. `cat my.log | bunyan --color | less -R`. Use `--no-color` to
  disable coloring, e.g. if your terminal doesn't support ANSI codes.
- Add 'level' field to log record before custom fields for that record. This
  just means that the raw record JSON will show the 'level' field earlier,
  which is a bit nicer for raw reading.


## bunyan 0.6.4

- [issue #5] Fix `log.info() -> boolean` to work properly. Previous all were
  returning false. Ditto all trace/debug/.../fatal methods.


## bunyan 0.6.3

- Allow an optional `msg` and arguments to the `log.info(<Error> err)` logging
  form. For example, before:

        log.debug(my_error_instance)            // good
        log.debug(my_error_instance, "boom!")   // wasn't allowed

  Now the latter is allowed if you want to expliciting set the log msg. Of course
  this applies to all the `log.{trace|debug|info...}()` methods.

- `bunyan` cli output: clarify extra fields with quoting if empty or have
  spaces. E.g. 'cmd' and 'stderr' in the following:

        [2012-02-12T00:30:43.736Z] INFO: mo-docs/43194 on banana.local: buildDocs results (req_id=185edca2-2886-43dc-911c-fe41c09ec0f5, route=PutDocset, error=null, stderr="", cmd="make docs")


## bunyan 0.6.2

- Fix/guard against unintended inclusion of some files in npm published package
  due to <https://github.com/isaacs/npm/issues/2144>


## bunyan 0.6.1

- Internal: starting jsstyle usage.
- Internal: add .npmignore. Previous packages had reams of bunyan crud in them.


## bunyan 0.6.0

- Add 'pid' automatic log record field.


## bunyan 0.5.3

- Add 'client_req' (HTTP client request) standard formatting in `bunyan` CLI
  default output.
- Improve `bunyan` CLI default output to include *all* log record keys. Unknown keys
  are either included in the first line parenthetical (if short) or in the indented
  subsequent block (if long or multiline).


## bunyan 0.5.2

- [issue #3] More type checking of `new Logger(...)` and `log.child(...)`
  options.
- Start a test suite.


## bunyan 0.5.1

- [issue #2] Add guard on `JSON.stringify`ing of log records before emission.
  This will prevent `log.info` et al throwing on record fields that cannot be
  represented as JSON. An error will be printed on stderr and a clipped log
  record emitted with a 'bunyanMsg' key including error details. E.g.:

        bunyan: ERROR: could not stringify log record from /Users/trentm/tm/node-bunyan/examples/unstringifyable.js:12: TypeError: Converting circular structure to JSON
        {
          "name": "foo",
          "hostname": "banana.local",
          "bunyanMsg": "bunyan: ERROR: could not stringify log record from /Users/trentm/tm/node-bunyan/examples/unstringifyable.js:12: TypeError: Converting circular structure to JSON",
        ...

  Some timing shows this does effect log speed:

        $ node tools/timeguard.js     # before
        Time try/catch-guard on JSON.stringify:
         - log.info:  0.07365ms per iteration
        $ node tools/timeguard.js     # after
        Time try/catch-guard on JSON.stringify:
         - log.info:  0.07368ms per iteration


## bunyan 0.5.0

- Use 10/20/... instead of 1/2/... for level constant values. Ostensibly this
  allows for intermediary levels from the defined "trace/debug/..." set.
  However, that is discouraged. I'd need a strong user argument to add
  support for easily using alternative levels. Consider using a separate
  JSON field instead.
- s/service/name/ for Logger name field. "service" is unnecessarily tied
  to usage for a service. No need to differ from log4j Logger "name".
- Add `log.level(...)` and `log.levels(...)` API for changing logger stream
  levels.
- Add `TRACE|DEBUG|INFO|WARN|ERROR|FATAL` level constants to exports.
- Add `log.info(err)` special case for logging an `Error` instance. For
  example `log.info(new TypeError("boom")` will produce:

        ...
        "err": {
          "message": "boom",
          "name": "TypeError",
          "stack": "TypeError: boom\n    at Object.<anonymous> ..."
        },
        "msg": "boom",
        ...


## bunyan 0.4.0

- Add `new Logger({src: true})` config option to have a 'src' attribute be
  automatically added to log records with the log call source info. Example:

        "src": {
          "file": "/Users/trentm/tm/node-bunyan/examples/src.js",
          "line": 20,
          "func": "Wuzzle.woos"
        },


## bunyan 0.3.0

- `log.child(options[, simple])` Added `simple` boolean arg. Set `true` to
  assert that options only add fields (no config changes). Results in a 10x
  speed increase in child creation. See "tools/timechild.js". On my Mac,
  "fast child" creation takes about 0.001ms. IOW, if your app is dishing
  10,000 req/s, then creating a log child for each request will take
  about 1% of the request time.
- `log.clone` -> `log.child` to better reflect the relationship: streams and
  serializers are inherited. Streams can't be removed as part of the child
  creation. The child doesn't own the parent's streams (so can't close them).
- Clean up Logger creation. The goal here was to ensure `log.child` usage
  is fast. TODO: measure that.
- Add `Logger.stdSerializers.err` serializer which is necessary to get good
  Error object logging with node 0.6 (where core Error object properties
  are non-enumerable).


## bunyan 0.2.0

- Spec'ing core/recommended log record fields.
- Add `LOG_VERSION` to exports.
- Improvements to request/response serializations.


## bunyan 0.1.0

First release.
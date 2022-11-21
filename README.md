# ftlog

[![Build Status](https://github.com/nonconvextech/ftlog/workflows/CI%20%28Linux%29/badge.svg?branch=main)](https://github.com/nonconvextech/ftlog/actions)
![License](https://img.shields.io/crates/l/ftlog.svg)
[![Latest Version](https://img.shields.io/crates/v/ftlog.svg)](https://crates.io/crates/ftlog)
[![ftlog](https://docs.rs/ftlog/badge.svg)](https://docs.rs/ftlog)

Logging is affected by disk IO and pipe system call.
Sequencial log call can a bottleneck in scenario where low
latency is critical (e.g. high frequency trading).

`ftlog` alleviates this bootleneck by sending message to dedicated logger
thread, and computing as little as possible in main/worker thread.

`ftlog` can boost log performance by **times** in main/worker thread. See
performance for details.

**CAUTION**: this crate use `unchecked_math` unstable feature and `unsafe`
code. Only use this crate in rust `nightly` channel.


## Usage

Add to your `Cargo.toml`:

```toml
ftlog = "0.2.0"
```

Configure and initialize ftlog at the start of your `main` function:
```rust
// ftlog re-export `log`'s macros, so no need to add `log` to dependencies
use ftlog::{debug, trace};
use log::{error, info, warn};

fn main() {
    // minimal configuration with default setting
    // define root appender, pass None would write to stderr
    let dest = FileAppender::new("./current.log");
    ftlog::builder().root(dest).build().unwrap().init().unwrap();

    trace!("Hello world!");
    debug!("Hello world!");
    info!("Hello world!");
    warn!("Hello world!");
    error!("Hello world!");

    // when main thread is done, logging thread may be busy printing messages
    // wait for log output to flush, otherwise messages in memory yet might lost
    ftlog::logger().flush();
}
```

A more complicated but feature rich usage:

```rust
use ftlog::{appender::{Period, FileAppender}, LevelFilter, Record, FtLogFormat};

// Custom formatter
// A formatter defines how to build a message.
// Since Formatting message into string can slow down the log macro call,
// the idomatic way is to send required field as is to log thread, and build message in log thread.
struct MyFormatter;
impl FtLogFormat for MyFormatter {
    fn msg(&self, record: &Record) -> Box<dyn Send + Sync + std::fmt::Display> {
        Box::new(Msg {
            level: record.level(),
            thread: std::thread::current().name().map(|n| n.to_string()),
            file: record.file_static(),
            line: record.line(),
            args: format!("{}", record.args()),
            module_path: record.module_path_static(),
        })
    }
}

// Store necessary field, define how to build into string with `Display` trait.
struct Msg {
    level: Level,
    thread: Option<String>,
    file: Option<&'static str>,
    line: Option<u32>,
    args: String,
    module_path: Option<&'static str>,
}

impl Display for Msg {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str(&format!(
            "{}@{}||{}:{}[{}] {}",
            self.thread.as_ref().map(|x| x.as_str()).unwrap_or(""),
            self.module_path.unwrap_or(""),
            self.file.unwrap_or(""),
            self.line.unwrap_or(0),
            self.level,
            self.args
        ))
    }
}

// configurate logger
let logger = ftlog::builder()
    // use custom fotmat
    // datetime format is not configurable for performance
    .format(StringFormatter)
    // global max log level
    .max_log_level(LevelFilter::Info)
    // define root appender, pass None would write to stderr
    .root(FileAppender::rotate_with_expire(
        "./current.log",
        Period::Minute,
        Duration::seconds(30),
    ))
    // ---------- configure additional filter ----------
    // write to "ftlog-appender" appender, with different level filter
    .filter("ftlog::appender", "ftlog-appender", LevelFilter::Error)
    // write to root appender, but with different level filter
    .filter("ftlog", None, LevelFilter::Trace)
    // write to "ftlog" appender, with default level filter
    .filter("ftlog::appender::file", "ftlog", None)
    // ----------  configure additional appender ----------
    // new appender
    .appender("ftlog-appender", FileAppender::new("ftlog-appender.log"))
    // new appender, rotate to new file every Day
    .appender("ftlog", FileAppender::rotate("ftlog.log", Period::Day))
    .build()
    .expect("logger build failed");
// init global logger
logger.init().expect("set logger failed");
```

See `examples/complex.rs`.

### Default Log Format

Datetime format is fixed for performance.

> 2022-04-08 19:20:48.190+08 **298ms** INFO main/src/ftlog.rs:14 My log
> message

Here `298ms` denotes the latency between log macros call (e.g.
`log::info!("msg")`) and actual printing in log thread. Normally this will
be 0ms.

A large delay indicates log thread might be blocked by excessive log
messages.

> 2022-04-10 21:27:15.996+08 0ms **2** INFO main/src/main.rs:29 limit
> running3 !

The number **2** above shows how many log messages have been discarded.
Only shown when limiting logging interval for a single log call (e.g.
`log::info!(limit=3000;"msg")`).

### Log with interval

`ftlog` allows restriction on writing frequency for single log call.

If the above line is called multiple times within 3000ms, then it will only
log once with a added number donates the number of discarded log message.

Each log call owns independent interval, so we can set different interval
for different log call. Internally, `ftlog` records last print time by a
combination of (module name, file name, line of code).

#### Example

```rust
info!(limit=3000; "limit running{} !", 3);
info!(limit=1000; "limit running{} !", 1);
```
The minimal interval of the the above log call is 3000ms.

```plain
2022-04-10 21:27:15.996+08 0ms 2 INFO main/src/main.rs:29 limit running1 !
```
The number **2** above shows how many log messages have been discarded.

### Log rotation
`ftlog` supports log rotation in local timezone. The available rotation
periods are:

- minute `Period::Minute`
- hour `Period::Hour`
- day `Period::Day`
- month `Period::Month`
- year `Period::Year`

Log rotation is configured in `FileAppender`, and datetime will be added to
the end of filename:

```rust
use ftlog::appender::{FileAppender, Period};

let logger = ftlog::builder()
    .root(FileAppender::rotate("./mylog.log", Period::Minute))
    .build()
    .unwrap();
logger.init().unwrap();
```

When configured to divide log file by minutes,
the file name of log file is in the format of
`mylog-{MMMM}{YY}{DD}T{hh}{mm}.log`. When by days, the log file names is
something like `mylog-{MMMM}{YY}{DD}.log`.

Log filename examples:
```sh
$ ls
# by minute
current-20221026T1351.log
current-20221026T1352.log
current-20221026T1353.log
# by hour
current-20221026T13.log
current-20221026T14.log
# by day
current-20221026.log
current-20221027.log
# by month
current-202210.log
current-202211.log
# by year
current-2022.log
current-2023.log

# omitting extension (e.g. "./log") will add datetime to the end of log filename
log-20221026T1351
log-20221026T1352
log-20221026T1353
```

#### Clean outdated logs

With log rotation enabled, it is possible to clean outdated logs to free up
disk space with `FileAppender::rotate_with_expire` method.

`ftlog` first finds files generated by `ftlog` and cleans outdated logs by
last modified time. `ftlog` find generated logs by filename matched by file
stem and added datetime.

**ATTENTION**: Any files that matchs the pattern will be deleted.

```rust
use ftlog::{appender::{Period, FileAppender, Duration}};

// clean files named like `current-\d{8}T\d{4}.log`.
// files like `another-\d{8}T\d{4}.log` or `current-\d{8}T\d{4}` will not be deleted, since the filenames' stem do not match.
// files like `current-\d{8}.log` will remains either, since the rotation durations do not match.
let appender = FileAppender::rotate_with_expire("./current.log", Period::Minute, Duration::seconds(180));
let logger = ftlog::builder()
    .root(appender)
    .build()
    .unwrap();
logger.init().unwrap();
```

## Performance

> Rust：1.67.0-nightly

|                                                   |  message type | Apple M1 Pro, 3.2GHz  | AMD EPYC 7T83, 3.2GHz |
| ------------------------------------------------- | ------------- | --------------------- | --------------------- |
| `ftlog`                                           | static string |   89 ns/iter (±22)    | 197 ns/iter (±232)    |
| `ftlog`                                           | with i32      |   123 ns/iter (±31)   | 263 ns/iter (±124)    |
| `env_logger` <br/> output to file                 | static string | 1,674 ns/iter (±123)  | 1,142 ns/iter (±56)   |
| `env_logger` <br/> output to file                 | with i32      | 1,681 ns/iter (±59)   | 1,179 ns/iter (±46)   |
| `env_logger` <br/> output to file with `BufWriter`| static string | 279 ns/iter (±43)     | 550 ns/iter (±96)     |
| `env_logger` <br/> output to file with `BufWriter`| with i32      | 278 ns/iter (±53)     | 565 ns/iter (±95)     |

License: MIT OR Apache-2.0

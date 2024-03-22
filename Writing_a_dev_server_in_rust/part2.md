<meta name="daria:article_id" content="writing_a_dev_server_in_rust_part_2">
<meta name="daria:title" content="Part 2">
<meta name="daria:title_slug" content="part_2">
<meta name="daria:order" content="1">
<meta name="daria:created_on" content="2022-07-05">
<meta name="daria:tags" content="rust,html/css,javascript">

# Writing a dev server in rust - Logging

In this part I will look at setting up logging for use throughout the app.

## Introduction

Logging is not strictly needed for this project but it does make debugging problems a lot easier.

I am going to have a thread dedicated to it and use channels to send log items to which will be printed to the console for now.

The logger is based on one I have created previously so is more complex than it probably needed to be.

First of all I added a directory `src/logging` and 3 files: `mod.rs`, `common.rs` and `logger.rs`.

All common components are kept in `common.rs`:

```rust
pub struct LogItem {
    pub(crate) from: String,
    pub(crate) message: String,
    pub(crate) item_type: LogItemType,
}

pub enum LogItemType {
    Information,
    Success,
    Error,
    Warning,
    Trace,
    Debug,
}

pub enum ConsoleColor {
    Black,
    BlackBright,
    Red,
    RedBright,
    Green,
    GreenBright,
    Yellow,
    YellowBright,
    Blue,
    BlueBright,
    Magenta,
    MagentaBright,
    Cyan,
    CyanBright,
    White,
    WhiteBright,
}

pub fn create_item(from: String, message: String, item_type: LogItemType) -> LogItem {
    LogItem {
        from,
        message,
        item_type,
    }
}

impl LogItem {
    pub fn create(from: String, message: String, item_type: LogItemType) -> LogItem {
        LogItem {
            from,
            message,
            item_type,
        }
    }

    pub fn info(from: String, message: String) -> LogItem {
        LogItem::create(from, message, LogItemType::Information)
    }

    pub fn success(from: String, message: String) -> LogItem {
        LogItem::create(from, message, LogItemType::Success)
    }

    pub fn error(from: String, message: String) -> LogItem {
        LogItem::create(from, message, LogItemType::Error)
    }

    pub fn warning(from: String, message: String) -> LogItem {
        LogItem::create(from, message, LogItemType::Warning)
    }

    pub fn trace(from: String, message: String) -> LogItem {
        LogItem::create(from, message, LogItemType::Trace)
    }

    pub fn debug(from: String, message: String) -> LogItem {
        LogItem::create(from, message, LogItemType::Debug)
    }
}

impl ConsoleColor {
    pub fn set_foreground(&self) {
        print!("{}", self.get_foreground_color())
    }

    pub fn set_background(&self) {
        print!("{}", self.get_background_color())
    }

    pub fn reset() {
        print!("\x1B[0m")
    }

    pub fn get_foreground_color(&self) -> &'static str {
        match self {
            ConsoleColor::Black => "\x1B[30m",
            ConsoleColor::BlackBright => "\x1B[30;1m",
            ConsoleColor::Red => "\x1B[31m",
            ConsoleColor::RedBright => "\x1B[31;1m",
            ConsoleColor::Green => "\x1B[32m",
            ConsoleColor::GreenBright => "\x1B[32;1m",
            ConsoleColor::Yellow => "\x1B[33m",
            ConsoleColor::YellowBright => "\x1B[33;1m",
            ConsoleColor::Blue => "\x1B[34m",
            ConsoleColor::BlueBright => "\x1B[34;1m",
            ConsoleColor::Magenta => "\x1B[35m",
            ConsoleColor::MagentaBright => "\x1B[35;1m",
            ConsoleColor::Cyan => "\x1B[36m",
            ConsoleColor::CyanBright => "\x1B[36m;1m",
            ConsoleColor::White => "\x1B[37m",
            ConsoleColor::WhiteBright => "\x1B[37;1m",
            //ConsoleColor::Custom(id) => format!("\x1B[38;5;${}m", id).as_str(),
        }
    }

    pub fn get_background_color(&self) -> &'static str {
        match &self {
            ConsoleColor::Black => "\x1B[40m",
            ConsoleColor::BlackBright => "\x1B[40;1m",
            ConsoleColor::Red => "\x1B[41m",
            ConsoleColor::RedBright => "\x1B[41;1m",
            ConsoleColor::Green => "\x1B[42m",
            ConsoleColor::GreenBright => "\x1B[42;1m",
            ConsoleColor::Yellow => "\x1B[43m",
            ConsoleColor::YellowBright => "\x1B[43;1m",
            ConsoleColor::Blue => "\x1B[44m",
            ConsoleColor::BlueBright => "\x1B[44;1m",
            ConsoleColor::Magenta => "\x1B[45m",
            ConsoleColor::MagentaBright => "\x1B[45;1m",
            ConsoleColor::Cyan => "\x1B[46m",
            ConsoleColor::CyanBright => "\x1B[46m;1m",
            ConsoleColor::White => "\x1B[47m",
            ConsoleColor::WhiteBright => "\x1B[47;1m",
            //ConsoleColor::Custom(id) => format!("\x1b[48;5;${}m", id).as_str(),
        }
    }
}
```

The actual logger is stored in `logger.rs`:

```rust
use crate::logging::common::{ConsoleColor, LogItem, LogItemType};

use chrono::UTC;
use std::sync::mpsc;
use std::sync::mpsc::{Receiver, Sender};
use std::thread;
use std::thread::JoinHandle;

pub struct Logger {
    name: String,
    sender: Sender<LogItem>,
}

pub struct Log {
    handler: JoinHandle<()>,
    sender: Sender<LogItem>,
}

impl Logger {
    pub fn create(name: String, sender: Sender<LogItem>) -> Logger {
        Logger { name, sender }
    }

    pub fn create_from(&self, name: String) -> Logger {
        Logger {
            name,
            sender: self.sender.clone(),
        }
    }

    pub fn log(&self, item: LogItem) -> Result<(), &'static str> {
        match self.sender.send(item) {
            Ok(_) => Ok(()),
            Err(_) => Err("Could not write to log."),
        }
    }

    pub fn log_info(&self, message: String) -> Result<(), &'static str> {
        self.log(LogItem::info(self.name.clone(), message))
    }

    pub fn log_success(&self, message: String) -> Result<(), &'static str> {
        self.log(LogItem::success(self.name.clone(), message))
    }

    pub fn log_error(&self, message: String) -> Result<(), &'static str> {
        self.log(LogItem::error(self.name.clone(), message))
    }

    pub fn log_warning(&self, message: String) -> Result<(), &'static str> {
        self.log(LogItem::warning(self.name.clone(), message))
    }

    pub fn log_debug(&self, message: String) -> Result<(), &'static str> {
        self.log(LogItem::debug(self.name.clone(), message))
    }
}

impl Log {
    pub fn start() -> Result<Log, &'static str> {
        let (sender, receiver) = mpsc::channel::<LogItem>();

        let _ = sender.send(LogItem::info(
            "Logger".to_string(),
            "Starting log".to_string(),
        ));

        let handler = thread::spawn(move || loop {
            let item = receiver.recv().unwrap();
            Log::print(item);
        });

        let _ = sender.send(LogItem::success(
            "Log".to_string(),
            "Log started".to_string(),
        ));

        Ok(Log { handler, sender })
    }

    pub fn get_logger(&self, name: String) -> Logger {
        Logger {
            name,
            sender: self.sender.clone(),
        }
    }

    fn print(item: LogItem) {
        let (color, name) = match item.item_type {
            LogItemType::Information => (ConsoleColor::WhiteBright, "info  "), //{}
            LogItemType::Success => (ConsoleColor::Green, "ok    "),
            LogItemType::Error => (ConsoleColor::Red, "error "),
            LogItemType::Warning => (ConsoleColor::Yellow, "warn  "),
            LogItemType::Trace => (ConsoleColor::BlackBright, "debug "),
            LogItemType::Debug => (ConsoleColor::Magenta, "trace "),
        };

        color.set_foreground();
        println!(
            "[{} {}] {} - {}",
            UTC::now().format("%F %H:%M:%S%.3f"),
            name,
            item.from,
            item.message
        );
        ConsoleColor::reset();
    }
}
```

These are simply referenced in `mod.rs`:

```rust
pub mod common;
pub mod logger;
```

An example of setting up the log and getting a logger:

```rust
use crate::logging::logger::Log;

fn main() {
    let log = Log::start().unwrap();

    let logger = log.get_logger("test".to_string());

    logger.log_info("Hello, World!".to_string()).unwrap();
}
```

## A few improvments

There are a lot of calls to `.to_string()`, this could be improved but supporting `String` and `&str` types.

Also there are a lot of `.unwrap()` calls, this could be improved by introduction functions like `try_log_info()` and changing exisiting ones to handle the unwrap (or error).

It could also be improved further but adding the option for different sinks and implementing rust's log traits.

But generally, this does the job and the output looks good enough for now.

## Summary

In this part I went over setting up a basic logger and using it. Though not necessary for this project, it does give a good level of visibility to what the app is doing and makes debugging problems much easier.

<meta name="daria:article_id" content="writing_a_dev_server_in_rust_part_4">
<meta name="daria:title" content="Part 4">
<meta name="daria:title_slug" content="part_4">
<meta name="daria:order" content="3">
<meta name="daria:created_on" content="2022-07-05">
<meta name="daria:tags" content="rust,html/css,javascript">
<meta name="daria:image_id" content="cranes-2">

# Writing a dev server in rust - File watcher

In this part I will look at implementing the file watcher, which will send out notifications when a file is altered in the target directory.

## Overview

With logging and messaging covered it is time to start looking at the key components. I am going to start with file watcher. 
This will use [notify](https://docs.rs/notify/latest/notify/) rather than be implemented from scratch for cross platform compatibility.

It will look for new files or changes to existing files and send a notification to the message hub with details.

## Implementation

I started by adding a `files` directory and a `mod.rs` file in there. As with messaging,  this will be implemented in one file for now:

```rust
use std::{
    path::PathBuf,
    sync::mpsc::{self, Sender},
    thread::{self, JoinHandle},
    time::Duration,
};

use notify::{watcher, RecursiveMode, Watcher};

use crate::{logging::logger::Log, messaging::Notification};

pub struct FileWatcher {
    thread: JoinHandle<()>,
}

impl FileWatcher {
    /// Start the file watcher. This will return a FileWatcher with the related thread's
    /// JoinHandle.
    ///
    /// # Panics
    ///
    /// Panics if an DebouncedEvent::Error is returned.
    pub fn start(sender: Sender<Notification>, base_path: String, log: &Log) -> FileWatcher {
        let (tx, rx) = mpsc::channel();
        let logger = log.get_logger("file_watcher".to_string());

        let thread = thread::spawn(move || {
            let mut watcher = watcher(tx, Duration::from_secs(1)).unwrap();

            watcher.watch(base_path, RecursiveMode::Recursive).unwrap();

            loop {
                match rx.recv() {
                    Ok(event) => {
                        match event {
                            notify::DebouncedEvent::NoticeWrite(_) => {}
                            notify::DebouncedEvent::NoticeRemove(_) => {}
                            notify::DebouncedEvent::Create(e) => send_message(
                                &sender,
                                Notification::FileCreated(path_buf_to_string(e)),
                            ),
                            notify::DebouncedEvent::Write(e) => send_message(
                                &sender,
                                Notification::FileUpdated(path_buf_to_string(e)),
                            ),
                            notify::DebouncedEvent::Chmod(_) => {}
                            notify::DebouncedEvent::Remove(e) => send_message(
                                &sender,
                                Notification::FileRemoved(path_buf_to_string(e)),
                            ),
                            notify::DebouncedEvent::Rename(o, n) => send_message(
                                &sender,
                                Notification::FileRenamed(
                                    path_buf_to_string(o),
                                    path_buf_to_string(n),
                                ),
                            ),
                            notify::DebouncedEvent::Rescan => {}
                            notify::DebouncedEvent::Error(_, _) => todo!(),
                        };
                    }
                    Err(_) => {
                        logger.log_error("Watcher error.".to_string()).unwrap();
                    }
                }
            }
        });

        FileWatcher { thread }
    }
}

/// Convert a PathBuf to a String.
///
/// # Panics
///
/// Panics if the PathBuf contains non-UTF8 characters.
fn path_buf_to_string(path_buf: PathBuf) -> String {
    path_buf.to_str().unwrap().to_string()
}

/// Send a notification message.
fn send_message(sender: &Sender<Notification>, notification: Notification) {
    let r = sender.send(notification);

    match r {
        Ok(_) => {}
        Err(e) => println!("{}", e),
    };
}
```

This needs some cleaning up and the `todo!()` to be removed. Most of the heavy lifting is done by the `notify` crate, the reason is really just a wrapper to make it work in the context of this application.

Also the `println!()` in `send_message` should be cleaned up.

## Summary

With the file watcher set it one of the key components of the complete. The project is starting to come together now. 
As with other parts this could be improved but it has been a good learning experience so far. 

With the file watcher implemented the last major component is the http server.
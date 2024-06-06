<meta name="daria:article_id" content="writing_a_dev_server_in_rust_part_3">
<meta name="daria:title" content="Part 3">
<meta name="daria:title_slug" content="part_3">
<meta name="daria:order" content="2">
<meta name="daria:created_on" content="2022-07-05">
<meta name="daria:tags" content="rust,html/css,javascript">
<meta name="daria:image_id" content="cranes-2">

# Writing a dev server in rust - Messaging

In this part I will look one of the core components - messaging. Messages will be passed from the file watcher to an subscribers to trigger a page reload in the browser.

## Overview

Messaging is a common component used through the application. A the core is a message hub (for want of a better name). 
Subscribers register with the it by sending a `Sender`. Then the file watch can send notifications to the message hub.
These notifications are then cloned and send to each subscriber.

## Implementation

I added a directory `messaging` and a file in that called `mod.rs`. For now the implementation will be kept in there:

```rust
use std::{
    sync::mpsc::{Receiver, Sender},
    thread::{self, JoinHandle},
    time::Duration,
};

use crate::logging::logger::Log;

#[derive(Clone)]
pub enum Notification {
    FileCreated(String),
    FileUpdated(String),
    FileRemoved(String),
    FileRenamed(String, String),
}

pub struct Subscription {
    sender: Sender<Notification>,
}

pub struct MessageHub {
    thread: JoinHandle<()>,
}

impl MessageHub {
    /// Start the [`MessageHub`].
    ///
    /// # Panics
    ///
    /// Panics if there is an issue with the logger.
    pub fn start(
        receiver: Receiver<Subscription>,
        notifications: Receiver<Notification>,
        log: &Log,
    ) -> MessageHub {
        let mut subscribers: Vec<Sender<Notification>> = Vec::new();

        // A vec to the index of any broken subscriptions, so the can be dropped.
        let mut dead_subs: Vec<usize> = Vec::new();
        let logger = log.get_logger("message_hub".to_string());

        let thread = thread::spawn(move || loop {
            // Check for new subscribers
            match receiver.try_recv() {
                Ok(sub) => {
                    logger
                        .log_info("Subscription received".to_string())
                        .unwrap();
                    subscribers.push(sub.sender);
                }
                Err(_) => {}
            };

            // Check for notifications and send them to subscribers.
            match notifications.recv_timeout(Duration::from_secs(1)) {
                Ok(notification) => {
                    logger
                        .log_info("Notification received".to_string())
                        .unwrap();
                    for (i, sub) in &mut subscribers.iter().enumerate() {
                        match sub.send(notification.clone()) {
                            Ok(_) => logger
                                .log_info("Notification sent to subscriber".to_string())
                                .unwrap(),
                            Err(e) => {
                                // Subscriber pipe broken. Drop subscriber.
                                logger.log_warning(format!("Failure sending to subscriber, subscription to be dropped. Error: {}", e)).unwrap();
                                dead_subs.push(i);
                            }
                        };
                    }

                    if dead_subs.len() > 0 {
                        // Reverse so subs with a highest index are removed first.
                        // Example:
                        // 0, 1*, 2, 3* (* = remove).
                        // 3 will be removed leaving 0, 1, 2.
                        // Then 1 will be removed. To avoid calculating new next etc.
                        // This is the indexes are preserved as dead subs are removed.
                        // If not, removing the item at index 1 will mean there is no longer and item at index 3 for example.
                        dead_subs.reverse();

                        for i in &dead_subs {
                            subscribers.remove(*i);
                        }

                        dead_subs.clear();
                    };
                }
                Err(_) => {
                    // No notifications after timeout. Do nothing.
                }
            };
        });

        MessageHub { thread }
    }
}

impl Subscription {
    /// Creates a new [`Subscription`].
    pub fn new(sender: Sender<Notification>) -> Subscription {
        Subscription { sender }
    }
}
```

The implementation will do for now. It could be improved and cleaned up a bit, but it does the job.

## Summary

With messaging added it is possible to get the key components to talk to each other. 
It could be implemented directly in the file watcher, but for now I perfer to have it separated out.
This is a reasonable simple way to take one notification and send it to multiple subscribers.

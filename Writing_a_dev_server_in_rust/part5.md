# Writing a dev server in rust - WebSockets

In this part I will look at a bare bones WebSockets server implentation needed for the application. 
Currently it will only support server-to-client messaging, but that is enough.

## Overview

Before I talk about the http server it is important to go over the WebSockets server. 
This will be the bare minimum needed for the project, it will only support server-to-client messaging and short messages. 

In the future I might write a series on properly implmenting WebSockets but wanted to keep things simple for this application.

## Implementation

I used 2 creates for the implementation - `sha1` and `base64`. This is used for generating the handshake.

I started by adding a directory `ws` and file `mod.rs` into that. Because I am keeping it simple I added all the code in one file:

```rust
use sha1::{Digest, Sha1};

/// Handle the WebSockets handshake and return a WebSockets key for use in the Sec-WebSocket-Accept
//  http header.
pub fn handle_handshake(key: &String) -> String {
    let mut hasher = Sha1::new();

    // Combine the key and standard websocket uuid.
    hasher.update(format!("{}{}", key, "258EAFA5-E914-47DA-95CA-C5AB0DC85B11").as_bytes());

    // Sha1 hash and then base64 encode.
    base64::encode(hasher.finalize())
}

/// Handle creating a short WebSocket message to be send to a client.
pub fn handle_write(data: &mut Vec<u8>, length: u8) -> Vec<u8> {
    let mut response = Vec::with_capacity(length as usize + 2);

    // Fin byte
    let fin: u8 = 0x80;
    let byte1 = fin | 1;

    // 0 used because this is from the server.
    let byte2: u8 = 0 | length;

    response.push(byte1);
    response.push(byte2);

    response.append(data);
    response
}
```

This is very basic, but enough for this project. 

## Summary

With WebSockets set up the last component is building a http server to handle connections and serve files.

One day I hope to write a more indepth article/series on implementing WebSockets. It is a powerful technology and very useful.

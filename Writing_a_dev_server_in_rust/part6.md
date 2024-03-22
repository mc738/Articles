<meta name="daria:article_id" content="writing_a_dev_server_in_rust_part_6">
<meta name="daria:title" content="Part 6">
<meta name="daria:title_slug" content="part_6">
<meta name="daria:order" content="5">
<meta name="daria:created_on" content="2022-07-05">
<meta name="daria:tags" content="rust,html/css,javascript">

# Writing a dev server in rust - Http server

In this part I will look at implementing a http server in rust. This is the last component of the dev server and once complete the basic project will be usable.

## Overview

I have experiment a few times with writing http servers in rust so there will be so extras that will not be used in this project.

Personally I find http server to be a very rewarding project to create on their own, there is something mmensely satisfying about being able to server a web page you created on a server you built.
This is not meant to be a production ready server but still a good insight into how server work.

## Implementation

To start I added a directory `http`. In there I added 3 files: `common.rs`, `server.rs` and `mod.rs`. 

The first file, `common.rs` contains basic http types such as requests, responses, verbs and status:

```rust
use crate::logging::logger::Logger;
use std::collections::HashMap;
use std::io::Read;
use std::net::TcpStream;

#[derive(Clone, Copy)]
pub enum HttpVerb {
    GET,
    HEAD,
    POST,
    PUT,
    DELETE,
    CONNECT,
    OPTIONS,
    TRACE,
    PATCH,
}

pub enum HttpStatus {
    SwitchingProtocols,
    Ok,
    BadRequest,
    Unauthorized,
    NotFound,
    MethodNotAllowed,
    InternalError,
}

pub struct HttpRequest {
    pub header: HttpRequestHeader,
    pub body: Option<Vec<u8>>,
}

pub struct HttpRequestHeader {
    pub route: String,
    pub verb: HttpVerb,
    pub content_length: usize,
    pub headers: HashMap<String, String>,
    pub http_version: String,
}

pub struct HttpResponse {
    pub header: HttpResponseHeader,
    pub body: Option<Vec<u8>>,
}

pub struct HttpResponseHeader {
    pub http_version: String,
    pub status: HttpStatus,
    pub content_length: usize,
    //pub content_type: String,
    pub headers: HashMap<String, String>,
}

impl HttpVerb {
    /// Create a HttpVerb from a name.
    ///
    /// # Errors
    ///
    /// This function will return an error if the name is unknown.
    pub fn from_str(data: &str) -> Result<HttpVerb, &'static str> {
        match data.to_uppercase().as_str() {
            "GET" => Ok(HttpVerb::GET),
            "HEAD" => Ok(HttpVerb::HEAD),
            "POST" => Ok(HttpVerb::POST),
            "PUT" => Ok(HttpVerb::PUT),
            "DELETE" => Ok(HttpVerb::DELETE),
            "CONNECT" => Ok(HttpVerb::CONNECT),
            "PATCH" => Ok(HttpVerb::PATCH),
            "OPTIONS" => Ok(HttpVerb::OPTIONS),
            "TRACE" => Ok(HttpVerb::TRACE),
            _ => Err("Unknown http verb"),
        }
    }

    /// Returns a reference to the name of this [`HttpVerb`].
    pub fn get_str(&self) -> &'static str {
        match self {
            HttpVerb::GET => "GET",
            HttpVerb::HEAD => "HEAD",
            HttpVerb::POST => "POST",
            HttpVerb::PUT => "PUT",
            HttpVerb::DELETE => "DELETE",
            HttpVerb::CONNECT => "CONNECT",
            HttpVerb::OPTIONS => "OPTIONS",
            HttpVerb::TRACE => "TRACE",
            HttpVerb::PATCH => "PATCH",
        }
    }
}

impl HttpStatus {
    /// Create a HttpStatus from a status code.
    ///
    /// # Errors
    ///
    /// This function will return an error if the status code is unknown.
    pub fn from_code(code: i16) -> Result<HttpStatus, &'static str> {
        match code {
            101 => Ok(HttpStatus::SwitchingProtocols),
            200 => Ok(HttpStatus::Ok),
            400 => Ok(HttpStatus::BadRequest),
            401 => Ok(HttpStatus::Unauthorized),
            404 => Ok(HttpStatus::NotFound),
            405 => Ok(HttpStatus::MethodNotAllowed),
            500 => Ok(HttpStatus::InternalError),
            _ => Err("Unknown response type code"),
        }
    }

    /// Returns the status code of this [`HttpStatus`].
    pub fn get_code(&self) -> i16 {
        match self {
            HttpStatus::SwitchingProtocols => 101,
            HttpStatus::Ok => 200,
            HttpStatus::BadRequest => 400,
            HttpStatus::Unauthorized => 401,
            HttpStatus::NotFound => 404,
            HttpStatus::MethodNotAllowed => 405,
            HttpStatus::InternalError => 500,
        }
    }

    /// Returns a reference to the name of this [`HttpStatus`].
    pub fn get_str(&self) -> &'static str {
        match self {
            HttpStatus::SwitchingProtocols => "Switching Protocols",
            HttpStatus::Ok => "OK",
            HttpStatus::BadRequest => "Bad Request",
            HttpStatus::Unauthorized => "Unauthorized",
            HttpStatus::NotFound => "Not Found",
            HttpStatus::MethodNotAllowed => "Method Not Allowed",
            HttpStatus::InternalError => "Internal Error",
        }
    }
}

impl HttpRequest {
    /// Create a new HttpRequest.
    pub fn create(
        route: String,
        verb: HttpVerb,
        content_type: String,
        addition_headers: HashMap<String, String>,
        body: Option<Vec<u8>>,
    ) -> HttpRequest {
        let len = match &body {
            None => 0,
            Some(b) => b.len(),
        };

        HttpRequest {
            header: HttpRequestHeader::create(route, verb, content_type, addition_headers, len),
            body,
        }
    }

    /// Create a HttpRequest from a TcpStream.
    ///
    /// # Panics
    ///
    /// Panics if the stream can not be read or there is an issue with the logger.
    ///
    /// # Errors
    ///
    /// Does not currently error, however it should error instead of panic.
    pub fn from_stream(
        mut stream: &TcpStream,
        logger: &Logger,
    ) -> Result<HttpRequest, &'static str> {
        let mut buffer = [0; 4096];
        let mut body: Vec<u8> = Vec::new();
        logger
            .log_debug(format!("Parsing http request header."))
            .unwrap();
        stream.read(&mut buffer).unwrap();
        logger.log_debug(format!("Read to buffer.")).unwrap();
        let (header, body_start_index) = HttpRequestHeader::create_from_buffer(buffer)?;
        let body = match (
            header.content_length > 0,
            body_start_index + header.content_length as usize > 4096,
        ) {
            // Short cut -> content length is 0 so no body
            (false, _) => None,
            // If the body_start_index + content length
            // the request of the body is bigger than the buffer and more reads needed
            (true, true) => {
                // TODO handle!
                None
            }
            // If the body_start_index + content length < 2048,
            // the body is in the initial buffer and no more reading is needed.
            (true, false) => {
                let end = body_start_index + header.content_length as usize;

                let body = buffer[body_start_index..end].to_vec();

                Some(body)
            }
        };

        Ok(HttpRequest { header, body })
    }

    /// Get the bytes of this [`HttpRequest`].
    pub fn to_bytes(&mut self) -> Vec<u8> {
        // Get the bytes for the header and append the response body.
        let mut bytes = self.header.to_bytes();

        if let Some(b) = &self.body {
            let mut body = b.clone();

            bytes.append(&mut body);
        }

        bytes
    }
}

impl HttpRequestHeader {
    /// Create a new HttpRequestHeader.
    pub fn create(
        route: String,
        verb: HttpVerb,
        content_type: String,
        addition_headers: HashMap<String, String>,
        content_length: usize,
    ) -> HttpRequestHeader {
        let http_version = String::from("HTTP/1.1");

        // Map the headers.
        let mut headers: HashMap<String, String> = HashMap::new();

        // Add any standardized headers.
        headers.insert("Server".to_string(), "Psionic 0.0.1".to_string());
        headers.insert("Content-Length".to_string(), format!("{}", content_length));
        headers.insert("Connection".to_string(), "Closed".to_string());
        headers.insert("Content-Type".to_string(), content_type);

        for (k, v) in addition_headers {
            headers.insert(k, v);
        }

        HttpRequestHeader {
            route,
            verb,
            content_length,
            headers,
            http_version,
        }
    }

    /// Create a new HttpRequestHeader from a buffer.
    ///
    /// # Errors
    ///
    /// This function will return an error if the request header is larger than the buffer.
    pub fn create_from_buffer(
        buffer: [u8; 4096],
    ) -> Result<(HttpRequestHeader, usize), &'static str> {
        for i in 0..buffer.len() {
            if i > 4
                && buffer[i] == 10
                && buffer[i - 1] == 13
                && buffer[i - 2] == 10
                && buffer[i - 3] == 13
            {
                // \r\n\r\n found, after this its the body.
                let header = String::from_utf8_lossy(&buffer[0..i]).into_owned();

                //println!("{}", header);

                let request = HttpRequestHeader::parse_from_string(header)?;

                return Ok((request, i + 1));
            }
        }

        Err("Request header larger than buffer")
    }

    /// Parse a HttpRequestHeader from a string.
    ///
    /// # Errors
    ///
    /// This function will not return an error if the HttpVerb can not be created.
    pub fn parse_from_string(data: String) -> Result<HttpRequestHeader, &'static str> {
        let split_header: Vec<&str> = data.split("\r\n").collect();

        let mut headers = HashMap::new();

        let mut content_length: usize = 0;

        let split_status_line: Vec<&str> = split_header[0].split(" ").collect();

        let verb = HttpVerb::from_str(split_status_line[0])?;
        let route = String::from(split_status_line[1]);
        let http_version = String::from(split_status_line[2]);

        for i in 1..split_header.len() {
            let split_item: Vec<&str> = split_header[i].split(": ").collect();

            // If the split item has more than 1 item, add a header.
            if split_item.len() > 1 {
                let k = String::from(split_item[0]).to_uppercase();
                let v = String::from(split_item[1]);

                // If the header item is `Content-Length` set it as such.
                if k == "CONTENT-LENGTH" {
                    match v.parse::<usize>() {
                        Ok(i) => content_length = i,
                        Err(_) => {}
                    }
                }

                headers.insert(k, v);
            }
        }

        Ok(HttpRequestHeader {
            route,
            verb,
            content_length,
            headers,
            http_version,
        })
    }

    /// Returns the string of this [`HttpRequestHeader`].
    pub fn get_string(&self) -> String {
        let mut header_string = String::new();

        header_string.push_str(&self.verb.get_str());
        header_string.push(' ');
        header_string.push_str(&self.route);
        header_string.push(' ');
        header_string.push_str(&self.http_version);

        header_string.push_str("\r\n");

        for header in &self.headers {
            header_string.push_str(&header.0);
            header_string.push_str(": ");
            header_string.push_str(&header.1);
            header_string.push_str("\r\n");
        }

        header_string.push_str("\r\n");
        header_string
    }

    /// Get the bytes of this [`HttpRequestHeader`].
    pub fn to_bytes(&mut self) -> Vec<u8> {
        let mut bytes = Vec::from(self.get_string().as_bytes());

        bytes
    }
}

impl HttpResponse {
    /// Create a new HttpResponse.
    pub fn create(
        status: HttpStatus,
        content_type: String,
        addition_headers: HashMap<String, String>,
        body: Option<Vec<u8>>,
    ) -> HttpResponse {
        let len = match &body {
            None => 0,
            Some(b) => b.len(),
        };

        HttpResponse {
            header: HttpResponseHeader::create(status, content_type, addition_headers, len),
            body,
        }
    }

    /// Create a new HttpResponse from a TcpStream.
    ///
    /// # Panics
    ///
    /// Panics if the stream can not be read.
    ///
    /// # Errors
    ///
    /// This function will return an error if the HttpResponseHeader can not be created.
    pub fn from_stream(
        mut stream: &TcpStream, /*, logger: &Logger*/
    ) -> Result<HttpResponse, &'static str> {
        let mut buffer = [0; 4096];
        let mut body: Vec<u8> = Vec::new();
        //logger.log_debug( format!("Parsing http response header.")).unwrap();
        let read = stream.read(&mut buffer).unwrap();
        //logger.log_debug(format!("Read to buffer.")).unwrap();
        let (header, body_start_index) = HttpResponseHeader::create_from_buffer(buffer)?;
        let body = match (
            header.content_length > 0,
            body_start_index + header.content_length as usize > 4096,
        ) {
            // Short cut -> content length is 0 so no body
            (false, _) => None,
            // If the body_start_index + content length
            // the request of the body is bigger than the buffer and more reads needed
            (true, true) => {
                // TODO handle!
                None
            }
            // If the body_start_index + content length < 2048,
            // the body is in the initial buffer and no more reading is needed.
            (true, false) => {
                if read == body_start_index {
                    // Only head was send (might be general.
                    // Therefore clear the array
                    buffer.fill(0);
                    stream.read(&mut buffer).unwrap();
                    body = buffer[0..header.content_length].to_vec();
                } else {
                    let end = body_start_index + header.content_length as usize;
                    body = buffer[body_start_index..end].to_vec();
                }

                Some(body)
            }
        };

        Ok(HttpResponse { header, body })
    }

    /// Returns the bytes of this [`HttpResponse`].
    pub fn to_bytes(&mut self) -> Vec<u8> {
        // Get the bytes for the header and append the response body.
        let mut bytes = self.header.to_bytes();

        if let Some(b) = &self.body {
            let mut body = b.clone();

            bytes.append(&mut body);
        }

        bytes
    }
}

impl HttpResponseHeader {
    /// Create a new HttpResponseHeader.
    pub fn create(
        status: HttpStatus,
        content_type: String,
        addition_headers: HashMap<String, String>,
        content_length: usize,
    ) -> HttpResponseHeader {
        let http_version = String::from("HTTP/1.1");

        // Map the headers.
        let mut headers: HashMap<String, String> = HashMap::new();

        // Add any standardized headers.
        headers.insert("Server".to_string(), "Psionic 0.0.1".to_string());
        headers.insert("Content-Length".to_string(), format!("{}", content_length));
        headers.insert("Connection".to_string(), "Closed".to_string());
        headers.insert("Content-Type".to_string(), content_type);

        for (k, v) in addition_headers {
            headers.insert(k, v);
        }

        HttpResponseHeader {
            http_version,
            status,
            content_length,
            headers,
        }
    }

    /// Create a new HttpResponseHeader from a bufffer.
    ///
    /// # Errors
    ///
    /// This function will return an error if the header is bigger than the buffer.
    pub fn create_from_buffer(
        buffer: [u8; 4096],
    ) -> Result<(HttpResponseHeader, usize), &'static str> {
        for i in 0..buffer.len() {
            if i > 4
                && buffer[i] == 10
                && buffer[i - 1] == 13
                && buffer[i - 2] == 10
                && buffer[i - 3] == 13
            {
                // \r\n\r\n found, after this its the body.
                let header = String::from_utf8_lossy(&buffer[0..i]).into_owned();

                //println!("{}", header);

                let response = HttpResponseHeader::parse_from_string(header)?;

                return Ok((response, i + 1));
            }
        }

        Err("Request header larger than buffer")
    }

    /// Parse a HttpResponseHeader from a string.
    ///
    /// # Errors
    ///
    /// This function will return an error if HttpStatus can not be created.
    pub fn parse_from_string(data: String) -> Result<HttpResponseHeader, &'static str> {
        let split_header: Vec<&str> = data.split("\r\n").collect();

        let mut headers = HashMap::new();

        let mut content_length: usize = 0;

        let split_status_line: Vec<&str> = split_header[0].split(" ").collect();

        //let verb = HttpVerb::from_str(split_status_line[0])?;
        //let route = String::from(split_status_line[1]);
        let http_version = String::from(split_status_line[0]);
        //let response = split_status_line[1].parse::<i32>();

        let status = match split_status_line[1].parse::<i16>() {
            Ok(status_code) => HttpStatus::from_code(status_code),
            Err(_) => Err("Failed to parse status code"),
        }?;

        for i in 1..split_header.len() {
            //println!("Head: {}", split_header[i]);

            let split_item: Vec<&str> = split_header[i].split(": ").collect();

            // If the split item has more than 1 item, add a header.
            if split_item.len() > 1 {
                let k = String::from(split_item[0]).to_uppercase();
                let v = String::from(split_item[1]);

                // If the header item is `Content-Length` set it as such.
                if k == "CONTENT-LENGTH" {
                    match v.parse::<usize>() {
                        Ok(i) => content_length = i,
                        Err(_) => {}
                    }
                }

                headers.insert(k, v);
            }
        }

        Ok(HttpResponseHeader {
            headers,
            http_version,
            status,
            content_length,
        })
    }

    /// Returns the string of this [`HttpResponseHeader`].
    pub fn get_string(&self) -> String {
        // Create the header.
        let mut header_string = String::new();

        header_string.push_str(&self.http_version);
        header_string.push(' ');
        header_string.push_str(&self.status.get_code().to_string());
        header_string.push(' ');
        header_string.push_str(self.status.get_str());

        header_string.push_str("\r\n");

        for header in &self.headers {
            header_string.push_str(&header.0);
            header_string.push_str(": ");
            header_string.push_str(&header.1);
            header_string.push_str("\r\n");
        }

        header_string.push_str("\r\n");
        header_string
    }

    /// Returns the bytes of this [`HttpResponseHeader`].
    pub fn to_bytes(&mut self) -> Vec<u8> {
        // Get the bytes for the header and append the response body.
        let bytes = Vec::from(self.get_string());

        bytes
    }
}
```

Because I copied this code from another project of mine it contains some bits that are not used in this application.

In `server.rs` I added the server implementation. This is based of the [rust example](https://doc.rust-lang.org/book/ch20-00-final-project-a-web-server.html).
It could use some refactoring but works as an initial version:

```rust
use std::{
    collections::HashMap,
    fs::File,
    io::{Read, Write},
    net::{TcpListener, TcpStream},
    sync::{
        mpsc::{self, Receiver, Sender},
        Arc, Mutex,
    },
    thread::{self, JoinHandle},
};

use regex::Regex;

use crate::{
    http::common::{HttpRequest, HttpResponse, HttpStatus},
    logging::logger::{Log, Logger},
    messaging::Subscription,
    ws,
};

pub(crate) struct Server {
    thread: JoinHandle<()>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;

struct ConnectionPool {
    sender: Sender<Job>,
    workers: Vec<Worker>,
}

struct Worker {
    id: usize,
    thread: JoinHandle<()>,
}

impl Server {
    /// Start the http server.
    ///
    /// # Errors
    ///
    /// This function will return an error if TcpListener can not be bond to the address.
    pub fn start(
        address: String,
        log: &Log,
        sub_sender: Sender<Subscription>,
        base_path: String,
    ) -> Result<Server, &'static str> {
        let logger = log.get_logger("server".to_string());
        let connection_pool = ConnectionPool::new(4);

        match TcpListener::bind(address) {
            Ok(listener) => {
                let thread = thread::spawn(move || loop {
                    for stream in listener.incoming() {
                        match stream {
                            Ok(stream) => {
                                let request_logger = logger.create_from("connection".to_string());
                                let ss = sub_sender.clone();
                                let bp = base_path.clone();
                                connection_pool
                                    .execute(|| handle_connection(stream, request_logger, ss, bp));
                            }
                            Err(_) => todo!(),
                        };
                    }
                });

                Ok(Server { thread })
            }
            Err(_) => Err("Could not start server."),
        }
    }
}

impl ConnectionPool {
    /// Creates a new [`ConnectionPool`].
    fn new(size: usize) -> ConnectionPool {
        let mut workers = Vec::with_capacity(size);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));

        for id in 0..size {
            let name = format!("worker_{}", id);
            workers.push(Worker::new(id, receiver.clone()));
        }

        ConnectionPool { sender, workers }
    }

    fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);
        self.sender.send(job).unwrap();
    }
}

impl Worker {
    /// Creates a new [`Worker`].
    ///
    /// # Panics
    ///
    /// Panics if a lock can ot be gained on the receiver or a job received.
    fn new(id: usize, receiver: Arc<Mutex<Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let job = receiver.lock().unwrap().recv().unwrap();
            job();
        });

        Worker { id, thread }
    }
}

/// Handle a connection from a client.
///
/// # Panics
///
/// Panics if an issue with the logger, a file can not be read or a failure to write to the stream.
fn handle_connection(
    mut stream: TcpStream,
    logger: Logger,
    sub_sender: Sender<Subscription>,
    base_path: String,
) {
    match HttpRequest::from_stream(&stream, &logger) {
        Ok(request) => match request.header.route.as_str() {
            "/ws/notify" => {
                logger
                    .log_info(format!("Update notification requested"))
                    .unwrap();
                handle_ws_connection(request, stream, sub_sender, logger);
            }
            route if route == "/" || route == "/index" || route == "/index.html" => {
                match File::open(format!("{}/index.html", base_path)) {
                    Ok(mut file) => {
                        let mut buf = Vec::new();

                        let mut doc = String::new();

                        file.read_to_string(&mut doc).unwrap();

                        file.read_to_end(&mut buf).unwrap();

                        let mut response = HttpResponse::create(
                            HttpStatus::Ok,
                            "text/html".to_string(),
                            HashMap::new(),
                            Some(inject_script(&doc).as_bytes().to_vec()),
                        );

                        stream.write(&response.to_bytes()).unwrap();
                    }
                    Err(_) => todo!(),
                }
            }
            _ => match File::open(get_path(format!(
                "{}{}",
                base_path,
                request.header.route.clone()
            ))) {
                Ok(mut file) => {
                    let mut buf = Vec::new();

                    file.read_to_end(&mut buf).unwrap();

                    let mut response = HttpResponse::create(
                        HttpStatus::Ok,
                        get_content_type(request.header.route.clone()),
                        HashMap::new(),
                        Some(buf),
                    );

                    stream.write(&&response.to_bytes()).unwrap();

                    logger
                        .log_info(format!("Request received. Route: {}", request.header.route))
                        .unwrap();
                }
                Err(_) => {
                    let mut response = HttpResponse::create(
                        HttpStatus::NotFound,
                        "text/plain".to_string(),
                        HashMap::new(),
                        Some(b"Not found".to_vec()),
                    );

                    stream.write(&response.to_bytes()).unwrap();
                }
            },
        },
        Err(_) => todo!(),
    };
}

/// Get a file path from a route.
fn get_path(route: String) -> String {
    route
}

/// Handle a WebSocket connection.
///
/// # Panics
///
/// Panics if a failure with the logger.
fn handle_ws_connection(
    request: HttpRequest,
    mut stream: TcpStream,
    sub_sender: Sender<Subscription>,
    logger: Logger,
) {
    logger.log_debug("WS connection".to_string()).unwrap();

    match request.header.headers.get("SEC-WEBSOCKET-KEY") {
        Some(key) => {
            logger.log_info(format!("Key: {}", key)).unwrap();
            let ws_handshake = ws::handle_handshake(key);
            logger
                .log_debug(format!("Handshake: {}", ws_handshake))
                .unwrap();

            let mut addition_headers = HashMap::new();

            addition_headers.insert("Upgrade".to_string(), "websocket".to_string());
            addition_headers.insert("Connection".to_string(), "Upgrade".to_string());
            addition_headers.insert("Sec-WebSocket-Accept".to_string(), ws_handshake);
            addition_headers.insert("Sec-WebSocket-Version".to_string(), "13".to_string());

            let mut response = HttpResponse::create(
                HttpStatus::SwitchingProtocols,
                "text/plain".to_string(),
                addition_headers,
                None,
            );

            match stream.write(&mut response.to_bytes()) {
                Ok(_) => {
                    // Handle web socket connection
                    let (tx, rx) = mpsc::channel();

                    let thread = thread::spawn(move || loop {
                        sub_sender.send(Subscription::new(tx.clone())).unwrap();

                        match rx.recv() {
                            Ok(notification) => {
                                let (data, len) = match notification {
                                    crate::messaging::Notification::FileCreated(_) => {
                                        (b"File created", 12)
                                    }
                                    crate::messaging::Notification::FileUpdated(_) => {
                                        (b"File updated", 12)
                                    }
                                    crate::messaging::Notification::FileRemoved(_) => {
                                        (b"File removed", 12)
                                    }
                                    crate::messaging::Notification::FileRenamed(_, _) => {
                                        (b"File renamed", 12)
                                    }
                                };

                                let result =
                                    stream.write(&ws::handle_write(&mut data.to_vec(), len));

                                match result {
                                    Ok(_) => {}
                                    Err(e) => {
                                        logger
                                            .log_error(format!(
                                                "Failed sending to client, Error {}",
                                                e
                                            ))
                                            .unwrap();
                                        break;
                                    }
                                };
                            }
                            Err(_) => (),
                        };
                    });
                }
                Err(_) => todo!(),
            }
        }

        None => todo!(),
    };
}

/// Get the content type from a path based on it's file extension.
fn get_content_type(path: String) -> String {
    match path {
        _ if path.ends_with(".css") => "text/css".to_string(),
        _ if path.ends_with(".js") => "application/javascript".to_string(),
        _ if path.ends_with(".png") => "image/png".to_string(),
        _ if path.ends_with(".jpg") || path.ends_with(".jpeg") => "image/jpeg".to_string(),
        _ => "text/plain".to_string(),
    }
}

/// Inject the handler script into a html document.
///
/// # Panics
///
/// Panics if the regex can not be created.
fn inject_script(document: &String) -> String {
    let re = Regex::new("</body>").unwrap();

    let replace = "<script>var ws = new WebSocket('ws://127.0.0.1:8080/ws/notify'); ws.onopen = function(evt) { console.log('Connected'); };  ws.onmessage = function (evt) { location.reload();  };</script>\n</body>";

    re.replace(document, replace).to_string()
}
```

Lastly, `mod.rs` just references the 2 other files: 

```rust
pub mod common;
pub mod server;
```

There is quite a lot of code here, it is the most complex part of the project.

## Summary

With the http server in place all the components of the application are complete. 
There are a lot of improvements that can be made, the whole project needs some refacotring.

However, it's not bad (even if I do say so myself) for an initial version and a couple of days work.
It has helped me get a bit more comfortable with rust (and vim), which was one of the main goals.

As I learn more I would like to refactor and improve the application to be a bit cleaner and remove the panics.
Also I would like to pull the http and logging bits out to separate crates. 
Both of these are parts I have used in other projects but for this series I wanted to keep things as self contained as possible.

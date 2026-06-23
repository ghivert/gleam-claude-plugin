---
name: gleam-wisp-mist
description: >-
  Wisp + Mist — building Gleam web apps: routing, request/response handling,
  middleware, static files, JSON, form data, WebSockets, SSE, and server setup
user-invocable: false
---

# Wisp + Mist — Gleam Web Stack

Wisp is a practical Gleam web framework (handlers, middleware, routing). Mist is
the underlying HTTP server it runs on. Most apps use Wisp for logic and Mist
only for startup; Mist can also be used directly for WebSockets, SSE, and
chunked responses.

```sh
gleam add wisp mist
```

## Minimal server

```gleam
import gleam/erlang/process
import mist
import wisp
import wisp/wisp_mist

pub fn main() {
  wisp.configure_logger()
  let secret_key_base = wisp.random_string(64)
  let assert Ok(_) =
    wisp_mist.handler(handle_request, secret_key_base)
    |> mist.new
    |> mist.port(8000)
    |> mist.start
  process.sleep_forever()
}

pub fn handle_request(req: wisp.Request) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.rescue_crashes
  use <- wisp.serve_static(req, under: "/static", from: "/public")
  use <- wisp.handle_head(req)
  router(req)
}

fn router(req: wisp.Request) -> wisp.Response {
  case wisp.path_segments(req) {
    [] -> home(req)
    ["posts", id] -> post(req, id)
    _ -> wisp.not_found()
  }
}

fn home(req: wisp.Request) -> wisp.Response {
  use <- wisp.require_method(req, http.Get)
  wisp.ok() |> wisp.html_body("<h1>Home</h1>")
}
```

---

## Wisp

### Core types

```gleam
pub type Request = request.Request(Connection)
pub type Response = response.Response(Body)

pub type Body {
  Text(String)
  Bytes(bytes_tree.BytesTree)
  File(path: String, offset: Int, limit: option.Option(Int))
}

pub type FormData {
  FormData(
    values: List(#(String, String)),
    files: List(#(String, UploadedFile)),
  )
}

pub type UploadedFile {
  UploadedFile(file_name: String, path: String)
}

pub type Security {
  PlainText
  Signed
}

pub type Range {
  Range(offset: Int, limit: option.Option(Int))
}

pub type LogLevel {
  EmergencyLevel | AlertLevel | CriticalLevel | ErrorLevel
  WarningLevel   | NoticeLevel | InfoLevel    | DebugLevel
}
```

### Response constructors

```gleam
pub fn ok() -> Response                                                        // 200
pub fn created() -> Response                                                   // 201
pub fn accepted() -> Response                                                  // 202
pub fn no_content() -> Response                                                // 204
pub fn redirect(to url: String) -> Response                                    // 302
pub fn permanent_redirect(to url: String) -> Response                          // 301
pub fn not_found() -> Response                                                 // 404
pub fn bad_request(detail: String) -> Response                                 // 400
pub fn method_not_allowed(allowed methods: List(http.Method)) -> Response      // 405
pub fn unprocessable_content() -> Response                                     // 422
pub fn content_too_large() -> Response                                         // 413
pub fn unsupported_media_type(accept acceptable: List(String)) -> Response     // 415
pub fn internal_server_error() -> Response                                     // 500
pub fn response(status: Int) -> Response
```

### Response body / header setters

```gleam
pub fn html_body(response: Response, html: String) -> Response
pub fn html_response(html: String, status: Int) -> Response
pub fn json_body(response: Response, json: String) -> Response
pub fn json_response(json: String, status: Int) -> Response
pub fn string_body(response: Response, content: String) -> Response
pub fn string_tree_body(response: Response, content: string_tree.StringTree) -> Response
pub fn set_body(response: Response, body: Body) -> Response
pub fn set_header(response: Response, name: String, value: String) -> Response
pub fn file_download(response: Response, named name: String, from path: String) -> Response
pub fn file_download_from_memory(response: Response, named name: String, containing data: bytes_tree.BytesTree) -> Response
```

### Middleware

All middleware uses `use <- middleware(...)` syntax.

```gleam
pub fn log_request(req: Request, handler: fn() -> Response) -> Response
pub fn rescue_crashes(handler: fn() -> Response) -> Response
pub fn serve_static(req: Request, under prefix: String, from directory: String, next handler: fn() -> Response) -> Response
pub fn handle_head(req: Request, next handler: fn(Request) -> Response) -> Response
pub fn method_override(request: Request(a)) -> Request(a)
pub fn require_method(request: Request(t), method: http.Method, next: fn() -> Response) -> Response
pub fn require_content_type(request: Request, expected: String, next: fn() -> Response) -> Response
pub fn content_security_policy_protection(handle_request: fn(String) -> Response) -> Response
pub fn csrf_known_header_protection(request: Request, next: fn(Request) -> Response) -> Response
```

### Request body parsing

```gleam
pub fn require_string_body(request: Request, next: fn(String) -> Response) -> Response
pub fn require_bit_array_body(request: Request, next: fn(BitArray) -> Response) -> Response
pub fn require_json(request: Request, next: fn(dynamic.Dynamic) -> Response) -> Response
pub fn require_form(request: Request, next: fn(FormData) -> Response) -> Response
pub fn read_body_bits(request: Request) -> Result(BitArray, Nil)
```

### Request introspection & configuration

```gleam
pub const path_segments: fn(Request(a)) -> List(String)
pub fn get_query(request: Request) -> List(#(String, String))
pub fn get_secret_key_base(request: Request) -> String
pub fn get_max_body_size(request: Request) -> Int
pub fn get_max_files_size(request: Request) -> Int
pub fn get_read_chunk_size(request: Request) -> Int

pub fn set_max_body_size(request: Request, size: Int) -> Request
pub fn set_max_files_size(request: Request, size: Int) -> Request
pub fn set_read_chunk_size(request: Request, size: Int) -> Request
pub fn set_secret_key_base(request: Request, key: String) -> Request
```

### Cookies

```gleam
pub fn get_cookie(request: Request, name name: String, security security: Security) -> Result(String, Nil)
pub fn set_cookie(
  response response: Response,
  request request: Request,
  name name: String,
  value value: String,
  security security: Security,
  max_age max_age: Int,
) -> Response
```

`PlainText` stores value as-is; `Signed` signs with secret key base.

### Signing

```gleam
pub fn sign_message(request: Request, message: BitArray, algorithm: crypto.HashAlgorithm) -> String
pub fn verify_signed_message(request: Request, message: String) -> Result(BitArray, Nil)
```

### Temporary files (uploads)

```gleam
pub fn new_temporary_file(request: Request) -> Result(String, simplifile.FileError)
pub fn delete_temporary_files(request: Request) -> Result(Nil, simplifile.FileError)
```

### Utilities

```gleam
pub fn escape_html(content: String) -> String
pub fn random_string(length: Int) -> String
pub fn parse_range_header(range_header: String) -> Result(Range, Nil)
pub const priv_directory: fn(String) -> Result(String, Nil)
```

### Logging

```gleam
pub fn configure_logger() -> Nil          // call once at startup
pub fn set_logger_level(log_level: LogLevel) -> Nil
pub fn log_emergency(message: String) -> Nil
pub fn log_alert(message: String) -> Nil
pub fn log_critical(message: String) -> Nil
pub fn log_error(message: String) -> Nil
pub fn log_warning(message: String) -> Nil
pub fn log_notice(message: String) -> Nil
pub fn log_info(message: String) -> Nil
pub fn log_debug(message: String) -> Nil
```

---

## `wisp/wisp_mist`

Bridges a Wisp handler to Mist.

```gleam
pub fn handler(
  handler: fn(request.Request(wisp.Connection)) -> response.Response(wisp.Body),
  secret_key_base: String,
) -> fn(request.Request(mist.Connection)) -> response.Response(mist.ResponseData)
```

---

## `wisp/simulate`

Test helpers — no running server needed.

```gleam
pub type FileUpload {
  FileUpload(file_name: String, content_type: String, content: BitArray)
}

pub const default_secret_key_base: String
pub const default_host: String
pub const default_headers: List(#(String, String))
pub const default_browser_headers: List(#(String, String))

pub fn request(method: http.Method, path: String) -> Request
pub fn browser_request(method: http.Method, path: String) -> Request
pub fn session(next_request: Request, previous_request: Request, previous_response: response.Response(wisp.Body)) -> Request

pub fn string_body(request: Request, text: String) -> Request
pub fn bit_array_body(request: Request, data: BitArray) -> Request
pub fn html_body(request: Request, html: String) -> Request
pub fn form_body(request: Request, data: List(#(String, String))) -> Request
pub fn json_body(request: Request, data: json.Json) -> Request
pub fn multipart_body(request: Request, values values: List(#(String, String)), files files: List(#(String, FileUpload))) -> Request

pub fn header(request: Request, name: String, value: String) -> Request
pub fn cookie(request: Request, name: String, value: String, security: wisp.Security) -> Request

pub fn read_body(response: response.Response(wisp.Body)) -> String
pub fn read_body_bits(response: response.Response(wisp.Body)) -> BitArray
```

---

## Mist (direct usage)

Use Mist directly for WebSockets, SSE, chunked streaming, or when bypassing Wisp
entirely.

### Core types

```gleam
pub opaque type Builder(request_body, response_body)
pub opaque type Connection       // HTTP request body type; holds client IP
pub opaque type WebsocketConnection
pub opaque type SSEConnection
pub opaque type SSEEvent
pub opaque type Next(state, user_message)

pub type ConnectionInfo { ConnectionInfo(port: Int, ip_address: IpAddress) }
pub type IpAddress {
  IpV4(Int, Int, Int, Int)
  IpV6(Int, Int, Int, Int, Int, Int, Int, Int)
}
pub type ReadError { ExcessBody | MalformedBody }
pub type FileError { IsDir | NoAccess | NoEntry | UnknownFileError }

pub type ResponseData {
  Websocket
  Bytes(bytes_tree.BytesTree)
  Chunked
  File(descriptor, offset: Int, length: Int)
  ServerSentEvents
}

pub type Chunk {
  Chunk(data: BitArray, consume: fn(Int) -> Result(Chunk, ReadError))
  Done
}

pub type ChunkNext(state) {
  ChunkContinue(state: state)
  ChunkStop
  ChunkAbort(reason: String)
}

pub type WebsocketMessage(custom) {
  Text(String)
  Binary(BitArray)
  Closed
  Shutdown
  Custom(custom)
}
```

### Server builder

```gleam
pub fn new(handler: fn(request.Request(in)) -> response.Response(out)) -> Builder(in, out)
pub fn port(builder: Builder(in, out), port: Int) -> Builder(in, out)
pub fn bind(builder: Builder(in, out), interface: String) -> Builder(in, out)
pub fn with_ipv6(builder: Builder(in, out)) -> Builder(in, out)
pub fn with_tls(builder: Builder(in, out), certfile cert: String, keyfile key: String) -> Builder(in, out)
pub fn after_start(builder: Builder(in, out), after_start: fn(Int, http.Scheme, IpAddress) -> Nil) -> Builder(in, out)
pub fn read_request_body(builder: Builder(BitArray, out), bytes_limit bytes_limit: Int, failure_response failure_response: response.Response(out)) -> Builder(Connection, out)
pub fn start(builder: Builder(Connection, ResponseData)) -> Result(actor.Started(static_supervisor.Supervisor), actor.StartError)
pub fn supervised(builder: Builder(Connection, ResponseData)) -> supervision.ChildSpecification(static_supervisor.Supervisor)
```

### Request body

```gleam
pub fn read_body(req: request.Request(Connection), max_body_limit max_body_limit: Int) -> Result(request.Request(BitArray), ReadError)
pub fn stream(req: request.Request(Connection)) -> Result(fn(Int) -> Result(Chunk, ReadError), ReadError)
```

### Connection info

```gleam
pub fn get_connection_info(conn: Connection) -> Result(ConnectionInfo, Nil)
pub fn ip_address_to_string(address: IpAddress) -> String
pub fn connection_info_to_string(connection_info: ConnectionInfo) -> String
```

### File serving

```gleam
pub fn send_file(path: String, offset offset: Int, limit limit: option.Option(Int)) -> Result(ResponseData, FileError)
```

### WebSockets

```gleam
pub fn websocket(
  request request: request.Request(Connection),
  handler handler: fn(state, WebsocketMessage(message), WebsocketConnection) -> Next(state, message),
  on_init on_init: fn(WebsocketConnection) -> #(state, option.Option(process.Selector(message))),
  on_close on_close: fn(state) -> Nil,
) -> response.Response(ResponseData)

pub fn send_text_frame(connection: WebsocketConnection, frame: String) -> Result(Nil, socket.SocketReason)
pub fn send_binary_frame(connection: WebsocketConnection, frame: BitArray) -> Result(Nil, socket.SocketReason)
```

WebSocket example:

```gleam
fn handle_ws(req) {
  mist.websocket(
    request: req,
    on_init: fn(_conn) { #(Nil, option.None) },
    on_close: fn(_state) { Nil },
    handler: fn(_state, msg, conn) {
      case msg {
        mist.Text(text) -> {
          let assert Ok(_) = mist.send_text_frame(conn, "echo: " <> text)
          mist.continue(Nil)
        }
        mist.Closed | mist.Shutdown -> mist.stop()
        _ -> mist.continue(Nil)
      }
    },
  )
}
```

### Actor loop control

```gleam
pub fn continue(state: state) -> Next(state, user_message)
pub fn stop() -> Next(state, user_message)
pub fn stop_abnormal(reason: String) -> Next(state, user_message)
pub fn with_selector(next: Next(state, user_message), selector: process.Selector(user_message)) -> Next(state, user_message)
```

### Server-Sent Events

```gleam
pub fn server_sent_events(
  request req: request.Request(Connection),
  initial_response resp: response.Response(discard),
  init init: fn(process.Subject(message)) -> state,
  loop loop: fn(state, message, SSEConnection) -> actor.Next(state, message),
) -> response.Response(ResponseData)

pub fn event(data: string_tree.StringTree) -> SSEEvent
pub fn event_id(event: SSEEvent, id: String) -> SSEEvent
pub fn event_name(event: SSEEvent, name: String) -> SSEEvent
pub fn event_retry(event: SSEEvent, retry: Int) -> SSEEvent
pub fn send_event(conn: SSEConnection, event: SSEEvent) -> Result(Nil, Nil)
```

### Chunked responses

```gleam
pub fn chunked(
  request req: request.Request(Connection),
  response response: response.Response(discard),
  init init: fn(process.Subject(message)) -> state,
  loop loop: fn(state, message, Connection) -> ChunkNext(state),
) -> response.Response(ResponseData)

pub fn send_chunk(connection: Connection, data: BitArray) -> Result(Nil, Nil)
pub fn chunk_continue(state: state) -> ChunkNext(state)
pub fn chunk_stop() -> ChunkNext(state)
pub fn chunk_stop_abnormal(reason: String) -> ChunkNext(state)
```

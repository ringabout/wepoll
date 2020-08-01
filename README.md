# wepoll
Windows epoll wrapper based on [wepoll](https://github.com/piscisaureus/wepoll)


## API

Docs from https://github.com/piscisaureus/wepoll

### General remarks

* The epoll port is a `HANDLE`, not a file descriptor.
* All functions set both `errno` and `GetLastError()` on failure.
* For more extensive documentation, see the [epoll(7) man page][man epoll],
  and the per-function man pages that are linked below.

### epoll_create/epoll_create1

```nim
proc epoll_create*(size: cint): HANDLE
proc epoll_create1*(flags: cint): HANDLE
```

* Create a new epoll instance (port).
* `size` is ignored but most be greater than zero.
* `flags` must be zero as there are no supported flags.
* Returns `NULL` on failure.
* [Linux man page][man epoll_create]

### epoll_close

```nim
proc epoll_close*(ephnd: HANDLE): cint
```

* Close an epoll port.
* Do not attempt to close the epoll port with `close()`,
  `CloseHandle()` or `closesocket()`.

### epoll_ctl

```nim
proc epoll_ctl*(ephnd: HANDLE; op: cint; 
                sock: SOCKET; event: ptr epoll_event): cint
```

* Control which socket events are monitored by an epoll port.
* `ephnd` must be a HANDLE created by
  [`epoll_create()`](#epoll_createepoll_create1) or
  [`epoll_create1()`](#epoll_createepoll_create1).
* `op` must be one of `EPOLL_CTL_ADD`, `EPOLL_CTL_MOD`, `EPOLL_CTL_DEL`.
* `sock` must be a valid socket created by [`socket()`][msdn socket],
  [`WSASocket()`][msdn wsasocket], or [`accept()`][msdn accept].
* `event` should be a pointer to a [`struct epoll_event`](#struct-epoll_event).<br>
  If `op` is `EPOLL_CTL_DEL` then the `event` parameter is ignored, and it
  may be `NULL`.
* Returns 0 on success, -1 on failure.
* It is recommended to always explicitly remove a socket from its epoll
  set using `EPOLL_CTL_DEL` *before* closing it.<br>
  As on Linux, closed sockets are automatically removed from the epoll set, but
  wepoll may not be able to detect that a socket was closed until the next call
  to [`epoll_wait()`](#epoll_wait).
* [Linux man page][man epoll_ctl]

### epoll_wait

```c
proc epoll_wait*(ephnd: HANDLE; 
                 events: ptr epoll_event; 
                 maxevents: cint; 
                 timeout: cint): cint
```

* Receive socket events from an epoll port.
* `events` should point to a caller-allocated array of
  [`epoll_event`](#-epoll_event) structs, which will receive the
  reported events.
* `maxevents` is the maximum number of events that will be written to the
  `events` array, and must be greater than zero.
* `timeout` specifies whether to block when no events are immediately available.
  - `<0` block indefinitely
  - `0`  report any events that are already waiting, but don't block
  - `≥1` block for at most N milliseconds
* Return value:
  - `-1` an error occurred
  - `0`  timed out without any events to report
  - `≥1` the number of events stored in the `events` buffer
* [Linux man page][man epoll_wait]

### object epoll_event

```nim
type
  epoll_data_t* {.bycopy, union.} = object
    p*: pointer
    fd*: cint
    u32*: uint32_t
    u64*: uint64_t
    sock*: SOCKET              ##  Windows specific
    hnd*: HANDLE               ##  Windows specific
```

```nim
type
  epoll_event* {.bycopy.} = object
    events*: uint32_t          ##  Epoll events and flags
    data*: epoll_data_t        ##  User data variable
```

* The `events` field is a bit mask containing the events being
  monitored/reported, and optional flags.<br>
  Flags are accepted by [`epoll_ctl()`](#epoll_ctl), but they are not reported
  back by [`epoll_wait()`](#epoll_wait).
* The `data` field can be used to associate application-specific information
  with a socket; its value will be returned unmodified by
  [`epoll_wait()`](#epoll_wait).
* [Linux man page][man epoll_ctl]

| Event         | Description                                                          |
|---------------|----------------------------------------------------------------------|
| `EPOLLIN`     | incoming data available, or incoming connection ready to be accepted |
| `EPOLLOUT`    | ready to send data, or outgoing connection successfully established  |
| `EPOLLRDHUP`  | remote peer initiated graceful socket shutdown                       |
| `EPOLLPRI`    | out-of-band data available for reading                               |
| `EPOLLERR`    | socket error<sup>1</sup>                                             |
| `EPOLLHUP`    | socket hang-up<sup>1</sup>                                           |
| `EPOLLRDNORM` | same as `EPOLLIN`                                                    |
| `EPOLLRDBAND` | same as `EPOLLPRI`                                                   |
| `EPOLLWRNORM` | same as `EPOLLOUT`                                                   |
| `EPOLLWRBAND` | same as `EPOLLOUT`                                                   |
| `EPOLLMSG`    | never reported                                                       |

| Flag             | Description               |
|------------------|---------------------------|
| `EPOLLONESHOT`   | report event(s) only once |
| `EPOLLET`        | not supported by wepoll   |
| `EPOLLEXCLUSIVE` | not supported by wepoll   |
| `EPOLLWAKEUP`    | not supported by wepoll   |

<sup>1</sup>: the `EPOLLERR` and `EPOLLHUP` events may always be reported by
[`epoll_wait()`](#epoll_wait), regardless of the event mask that was passed to
[`epoll_ctl()`](#epoll_ctl).

# Stream & Message Buffers

Queues copy fixed-size items. That's the wrong shape for two very common
jobs: a UART/DMA ISR handing off a raw byte stream, or a parser producing
variable-length frames. Forcing either through a queue means padding every
item to a worst-case size (wasteful) or building your own ring buffer with
manual locking (error-prone). FreeRTOS ships two purpose-built primitives
for exactly this: **stream buffers** for continuous byte streams and
**message buffers** for discrete, variable-length messages built on top of
the same lock-free core.

## Stream buffers: bytes in, bytes out

A stream buffer is a single-producer/single-consumer byte pipe. It has no
concept of "messages" — what comes out is just the next N bytes available,
regardless of how they were sent.

```cpp
#include "stream_buffer.h"

StreamBufferHandle_t uartStream;

void setup_stream() {
  // capacity 256 bytes, notify the reader once >= 8 bytes are available
  uartStream = xStreamBufferCreate(256, 8);
}

// ISR context — e.g. UART RX interrupt with a small local buffer
void UART_ISR(void) {
  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
  uint8_t byte = UART_READ_REG();
  xStreamBufferSendFromISR(uartStream, &byte, 1, &xHigherPriorityTaskWoken);
  portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

void parserTask(void *pv) {
  uint8_t buf[64];
  for (;;) {
    // blocks until >= trigger level bytes are available, or timeout
    size_t n = xStreamBufferReceive(uartStream, buf, sizeof(buf),
                                     pdMS_TO_TICKS(100));
    if (n > 0) {
      feedParser(buf, n);
    }
  }
}
```

The **trigger level** (the second argument to `xStreamBufferCreate`) is the
whole point: it decouples "data arrived" from "wake the reader." Set it to
1 and every byte wakes the task — usually wasteful. Set it to your typical
frame size and the reader wakes once per frame's worth of bytes instead of
once per byte, at the cost of a little added latency for the last few bytes
of a short burst. `xStreamBufferSetTriggerLevel()` can retune it at runtime.

## Message buffers: framing is included

A message buffer is a stream buffer with a length prefix bolted on. Each
send is one atomic "message"; each receive returns exactly one message
(or the receiver's buffer must be large enough for the biggest expected
message, or the excess is silently discarded — check your header size).

```cpp
#include "message_buffer.h"

MessageBufferHandle_t frames;

void setup_frames() {
  frames = xMessageBufferCreate(512);   // total buffer capacity in bytes
}

void radioTask(void *pv) {
  uint8_t packet[64];
  for (;;) {
    size_t len = receiveFromRadio(packet, sizeof(packet));
    // each call is one discrete message — no manual length prefixing needed
    xMessageBufferSend(frames, packet, len, pdMS_TO_TICKS(50));
  }
}

void protocolTask(void *pv) {
  uint8_t msg[64];
  for (;;) {
    size_t len = xMessageBufferReceive(frames, msg, sizeof(msg),
                                        portMAX_DELAY);
    if (len > 0) {
      handlePacket(msg, len);
    }
  }
}
```

Each `xMessageBufferReceive` call returns one complete message that was
handed to `xMessageBufferSend` — the boundary is preserved even if multiple
sends happened back-to-back. That's the entire value proposition over a
stream buffer: you get "one send = one receive" for free.

## Why not just use a queue of bytes, or a queue of pointers?

- **Queue of single bytes**: works, but every byte pays full queue
  bookkeeping overhead (a mutex-guarded copy in and out per byte) — orders
  of magnitude slower than the lock-free stream buffer for bulk transfer.
- **Queue of pointers to variable-length buffers**: fast, but now *you* own
  buffer lifetime — who allocates, who frees, what happens if the queue is
  full and the sender needs to reuse the buffer immediately? Stream/message
  buffers copy data into their own internal storage, sidestepping ownership
  entirely, at the cost of an extra copy.

Pointer queues remain the right choice for large, expensive-to-copy buffers
(a full camera frame); stream/message buffers win for small, frequent,
telemetry-style traffic where copy cost is negligible and you'd rather not
manage allocation.

## ISR-safe API and the single-reader/single-writer rule

Every stream/message buffer API has an `...FromISR` variant
(`xStreamBufferSendFromISR`, `xMessageBufferReceiveFromISR`, etc.) that
takes a `pxHigherPriorityTaskWoken` out-parameter instead of blocking —
exactly the same pattern as queues and semaphores. Never call the
non-ISR variant from an interrupt handler; it can attempt to block the CPU
forever inside an ISR.

The hard constraint that's easy to violate: **stream and message buffers
are single-producer/single-consumer**. Unlike a queue, there is no internal
mutual exclusion between multiple senders or multiple receivers. If two
tasks both call `xStreamBufferSend()` on the same buffer, or the ISR and a
task both send to it, the writes can interleave and corrupt the stream.
(The one documented exception: one task and one ISR may safely share a
buffer as writer and reader respectively — that's the intended ISR→task
handoff shown above.) If you need fan-in from multiple producers, funnel
them through a queue into a single relay task that owns the stream buffer.

## Traps

- **Multiple writers**: the single-producer rule above is silently
  violated more often than any other rule in this module — always trace
  who calls `Send` on a given handle before assuming it's safe.
- **Message too big for the receiver's buffer**: `xMessageBufferReceive`
  truncates to `xBufferLengthBytes` and the remainder of that message is
  lost — size the receive buffer to your largest possible message, not the
  average.
- **Stream buffer starvation**: if the trigger level is higher than what
  a slow producer ever sends, the reader blocks forever (or until timeout)
  even though bytes are sitting in the buffer. Match the trigger level to
  actual traffic patterns, and always pass a bounded timeout, not
  `portMAX_DELAY`, on links that can go quiet.
- **Static vs dynamic**: both offer a `...CreateStatic()` variant (Module 2)
  that takes a `uint8_t` storage array and a `StaticStreamBuffer_t` /
  `StaticMessageBuffer_t` control struct — required if `configSUPPORT_STATIC_ALLOCATION`
  is your only enabled allocation mode, and generally preferred for buffers
  whose size is known at compile time so a fragmented heap can't fail the
  create call at runtime.
- **Confusing capacity with message count**: `xMessageBufferCreate(512)`
  allocates 512 *bytes* total, shared by all messages currently queued
  (each message also costs a small internal length header) — it is not
  "room for 512 messages."

## Cheat sheet

| API / concept | Stream buffer | Message buffer |
|---|---|---|
| Create | `xStreamBufferCreate(size, triggerLevel)` | `xMessageBufferCreate(size)` |
| Static create | `xStreamBufferCreateStatic(...)` | `xMessageBufferCreateStatic(...)` |
| Send | `xStreamBufferSend(h, data, len, timeout)` | `xMessageBufferSend(h, data, len, timeout)` |
| Receive | `xStreamBufferReceive(h, buf, bufLen, timeout)` | `xMessageBufferReceive(h, buf, bufLen, timeout)` |
| ISR variants | `...FromISR(..., &xHigherPriorityTaskWoken)` | same pattern |
| Boundary semantics | None — arbitrary byte chunks | Preserved — one send = one receive |
| Writers/readers | Exactly one of each | Exactly one of each |
| Best for | Continuous byte streams (UART, DMA) | Discrete variable-length messages (packets, logs) |
| Wake tuning | `xStreamBufferSetTriggerLevel()` | N/A — wakes per message |

## Exercise

1. Build the UART ISR → `parserTask` pipeline above using a real (or
   simulated with a timer ISR feeding fake bytes) interrupt source. Try
   trigger levels of 1, 8, and 32 and measure how often `parserTask` wakes
   under a steady 100-bytes/second load.
2. Convert a producer/consumer pair from Level 1's queue-based pipeline
   (Module 4) to use a message buffer instead, where each "reading" is
   serialized to a variable-length string. Confirm message boundaries are
   preserved even when you burst-send three readings back to back before
   the consumer runs.
3. Deliberately create two tasks that both call `xStreamBufferSend()` on
   the same stream buffer under load. Observe (or reason through) how the
   data gets corrupted, then fix it by routing both through a single relay
   task fed by a queue.

# Networking — FreeRTOS-Plus-TCP

**Verification note for this module:** FreeRTOS-Plus-TCP requires a network
interface driver bound to real or emulated hardware (a MAC/PHY, or a
WinPCap/libpcap-backed interface for its own Windows/Linux simulation
targets) — there is no network stack wired into the POSIX-port build used
elsewhere in this course, and setting one up was out of scope for this pass.
Content here is a careful review of the FreeRTOS-Plus-TCP API and its own
published architecture documentation, not code executed on this machine.

## Why a separate TCP/IP stack, and why it looks like more FreeRTOS

FreeRTOS-Plus-TCP is Amazon/the FreeRTOS project's own lwIP-alternative
TCP/IP stack, built from the ground up around FreeRTOS primitives rather
than ported from a different RTOS's networking code. This shows up directly
in its API: the stack's internal state machine runs as its own task
(the **IP task**, `prvIPTask`), and application code interacts with sockets
using an API deliberately shaped like BSD sockets but implemented entirely
on top of FreeRTOS queues and semaphores underneath — consistent with every
other subsystem this course has covered rather than a foreign layer bolted
on top.

```c
#include "FreeRTOS_IP.h"
#include "FreeRTOS_Sockets.h"

static const uint8_t ucIPAddress[4] = {192, 168, 1, 50};
static const uint8_t ucNetMask[4]   = {255, 255, 255, 0};
static const uint8_t ucGatewayAddress[4] = {192, 168, 1, 1};
static const uint8_t ucDNSServerAddress[4] = {8, 8, 8, 8};
static const uint8_t ucMACAddress[6] = {0x00,0x11,0x22,0x33,0x44,0x55};

void vStartNetworking(void) {
    FreeRTOS_IPInit(ucIPAddress, ucNetMask, ucGatewayAddress,
                     ucDNSServerAddress, ucMACAddress);
    // network driver's own task/interrupt path calls xSendEventStructToIPTask()
    // to hand received frames to the IP task from here on
}
```

## The IP task: everything serialized through one place

A deliberate architectural choice: **all stack state (routing, socket
buffers, ARP cache) is only ever touched by the single IP task**, and every
other task's socket call (`FreeRTOS_send`, `FreeRTOS_recv`, `FreeRTOS_connect`)
is translated into a message sent to the IP task's own queue, processed
serially. This is precisely Level 2 Module 8's gatekeeper pattern, applied
at network-stack scale — no stack-internal mutex forest to get wrong, because
only one task ever mutates stack state:

```c
Socket_t xSocket = FreeRTOS_socket(FREERTOS_AF_INET, FREERTOS_SOCK_STREAM,
                                     FREERTOS_IPPROTO_TCP);
struct freertos_sockaddr xRemote = {
    .sin_family = FREERTOS_AF_INET,
    .sin_port = FreeRTOS_htons(80),
    .sin_addr = FreeRTOS_inet_addr("192.168.1.1"),
};
FreeRTOS_connect(xSocket, &xRemote, sizeof(xRemote));
FreeRTOS_send(xSocket, "GET / HTTP/1.0\r\n\r\n", 19, 0);

char buf[512];
BaseType_t received = FreeRTOS_recv(xSocket, buf, sizeof(buf), 0);
```

Application code never touches IP-task-owned state directly — every one of
these calls blocks the calling task (if configured to) while the IP task
does the real work, exactly mirroring how a gatekeeper task's queue
decouples callers from the resource owner.

## Zero-copy buffers and driver integration

`NetworkBufferDescriptor_t` structures are the stack's currency for received
and to-be-sent Ethernet frames — designed so a driver's receive-interrupt
handler can hand a buffer straight to the IP task without an extra memcpy,
and so the IP task can hand a transmit buffer straight to the driver's send
path the same way. This buffer-passing-by-reference design is why
`ipconfigZERO_COPY_TX_DRIVER`/`ipconfigZERO_COPY_RX_DRIVER` exist as explicit
configuration knobs — whether a given MAC driver can genuinely avoid the
copy depends on whether its DMA descriptors can point directly at
FreeRTOS-Plus-TCP's own buffer pool memory.

## TCP window, retransmission, and the RTOS timing connection

FreeRTOS-Plus-TCP implements its own TCP state machine (windowing,
retransmission timers, `ipconfigTCP_WINDOW_SIZE` for RAM/throughput
trade-offs) rather than delegating to any OS network stack — which means
every one of this course's earlier lessons about task priorities and ISR
latency (Module 7) applies directly to network performance: if the IP task
runs at too low a priority relative to application tasks that hog the CPU,
TCP retransmission timers and ACK generation can be delayed enough to tank
throughput, exactly the same mechanism as any other priority-starvation
scenario covered earlier in this course, just manifesting as "the network is
slow" instead of "a sensor reading is late."

## Traps

- **Setting the IP task's priority too low.** Because all socket operations
  route through it, a starved IP task doesn't just slow networking — it can
  make application tasks blocked in `FreeRTOS_send`/`recv` wait far longer
  than the network round-trip time alone would suggest, an easy
  misdiagnosis (blaming "the network" for what's actually a priority
  configuration problem).
- **Calling FreeRTOS-Plus-TCP socket functions from an ISR.** Like every
  other kernel API this course has covered, socket calls are task-context
  only — a driver's receive ISR must hand frames off via the documented
  `xSendEventStructToIPTask`/buffer-descriptor path, not call socket
  functions directly.
- **Undersizing `ipconfigTCP_WINDOW_SIZE` or buffer counts** on a
  RAM-constrained target and then being surprised by poor throughput —
  networking RAM budgets trade directly against every other subsystem's RAM
  needs (task stacks, queues) covered throughout this course, and need to be
  planned as part of the same overall budget, not as an afterthought.
- **Assuming zero-copy is automatic.** It requires a driver written to hand
  DMA descriptors directly into the stack's buffer pool — a driver that
  internally copies into its own buffers first defeats the purpose even with
  `ipconfigZERO_COPY_*` enabled.
- **Treating this module's coverage as sufficient for a real network
  security posture.** FreeRTOS-Plus-TCP handles the transport; TLS (usually
  via mbedTLS integration) and the broader secure-firmware concerns of
  Level 4 Module 5 are separate, additional layers this module does not
  cover.

## Cheat sheet

| Concept | Standard FreeRTOS pattern it mirrors | Purpose |
|---|---|---|
| IP task | Gatekeeper task (Level 2 Module 8) | Serializes all stack state access |
| Socket API calls | Queue send/receive to the gatekeeper | Decouples callers from stack internals |
| `NetworkBufferDescriptor_t` | Pass-by-reference buffer, like a stream buffer chunk | Avoids extra copies between driver and stack |
| IP task priority | Same priority-starvation rules as Module 7 | Network throughput is a scheduling problem too |
| `ipconfigTCP_WINDOW_SIZE` | Same RAM-budget trade-off as queue/stack sizing | Throughput vs. RAM |

## Exercise

1. Read the FreeRTOS-Plus-TCP source for `FreeRTOS_send` and trace exactly
   how it hands off to the IP task's queue — identify the specific point
   where a calling task's priority stops mattering and the IP task's
   priority takes over.
2. On a real target with FreeRTOS-Plus-TCP integrated, deliberately set the
   IP task's priority below several CPU-hogging application tasks and
   measure the throughput/latency impact — then correct the priority and
   confirm the improvement, directly connecting this module's warning to a
   measured result.
3. Design a buffer/window-size budget for a target with 64KB total RAM
   running three FreeRTOS-Plus-TCP sockets simultaneously, alongside the
   task stacks and queues from a prior Level 2 project — show your full RAM
   accounting, not just the networking portion.

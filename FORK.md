# arngory/picoquic

This is a fork of [private-octopus/picoquic](https://github.com/private-octopus/picoquic) for
[slipstream](https://github.com/arngory/slipstream), which runs QUIC over DNS. It replaces the old fork,
EndPositive/slipstream-picoquic. That fork was based on upstream `4bdf7c71` (2024-11-25), and it lacked
upstream's later security fixes.

Base: upstream tag `v2026.11.21.92fa226` (commit `92fa226`).

## Branches

| Branch | What it holds |
|---|---|
| `master` | A mirror of upstream `master`. It carries no changes of ours. |
| `slipstream` | The base tag plus the patch series below. The slipstream server and Niko pin this line. |

## Patches

Each patch is one commit. Its message says what the patch changes and why.

1. `slipstream: build flags for embedding in the slipstream builds`: no `-O3` and no `-fno-exceptions`; the flags are exported to `PARENT_SCOPE` when picoquic is included with `add_subdirectory`.
2. `slipstream: lower the enforced initial MTU, settable at build time`: `PICOQUIC_ENFORCED_INITIAL_MTU` is 129 by default and can be set with the CMake cache variable (`PUBLIC` define); `PICOQUIC_MIN_SEGMENT_SIZE` is 32.
3. `slipstream: picoquic_set_initial_send_mtu and config fields`: `initial_send_mtu_ipv4`/`_ipv6` in the config struct and in the QUIC context. Slipstream sets the struct fields; there is no command-line option for them.
4. `slipstream: POLL frame (type 0x20)`: a frame that is not ack-eliciting, sent on `cnx->is_poll_requested`.
5. `slipstream: at most 128 segments in one GSO send`
6. `slipstream: path-id variants of the incoming, prepare and probe calls`: the `_path_ex` and `_path_id_ex` functions; the upstream signatures are kept.
7. `slipstream: hooks for slipstream's own packet loop`: `before_select` (the last value of the callback enum, so upstream's values keep their numbers), the `encode`/`decode` parameters, `picoquic_packet_loop_wait_udp`, and `picoquic_open_network_wake_up`.
8. `slipstream: answer path challenges from unknown addresses with multipath`
9. `slipstream: congestion windows in units of the path MTU, cubic tuning`
10. `slipstream: ack and loss-recovery tuning`
11. `slipstream: split retransmitted stream frames down to 8 bytes`: `PICOQUIC_MIN_STREAM_DATA_FRAGMENT` is 8.
12. `slipstream: reorder memory factor 64 (upstream 6)`
13. `slipstream: name the POLL frame in logs and qlog; strerror on send errors`
14. `slipstream: pthread_kill fallback under _POSIX_THREAD_PROCESS_SHARED`: reverted on top (see below). Drop it at the next rebase.
15. `slipstream: retry_token_length test buffer sized by the max packet size`: a test-only change. The test overflowed its stack buffer once the MTU was lowered.

### Fixes on top of the series

These commits follow the series. Each message names the patch it belongs to, so that it can be squashed into that
patch at the next rebase.

| Fix | Squash into |
|---|---|
| Revert the `pthread_kill` fallback: glibc defines `_POSIX_THREAD_PROCESS_SHARED`, so glibc Linux sent SIGTERM; Android is covered by upstream's `ANDROID`; slipstream never calls `picoquic_delete_thread` | drop 14 |
| Path creation takes a family-0 peer as IPv6 again, as upstream (the `binlog` test) | 3 |
| Drop the two config option ids that no option-table row reached; they also renumbered the later options | 3 |
| `picoquic_packet_loop_before_select` at the end of the enum, with a correct comment | 7 |
| No debug `message()` in `set_picoquic_compile_settings` (16 lines on each configure) | 1 |
| `PARENT_SCOPE` exports only when there is a parent (5 CMake warnings on each top-level configure) | 1 |
| `picoquic_sendmsg` counts a short last GSO segment | 5 |
| `PICOQUIC_ACK_DELAY_MAX`'s comment says 100 ms; no `<stdio.h>` in loss_recovery.c | 10 |
| The fragment comment reworded, and the reorder block moved below the fragment define | 11, 12 |
| Never send a 0-RTT packet without frames (a bug: see the 0-RTT section) | 2 |

Patches 9 and 10 are tuning values carried over from the old fork. They will be measured again in the soak test
through real resolvers, and any that turn out not to be needed will be dropped.

One value of patch 10 does more than its name says. `PICOQUIC_ACK_DELAY_MAX` (100 ms, upstream 10 ms) is also:
- the `max_ack_delay` transport parameter, which the peer adds to every retransmit timer, so a broken path is demoted
  later (3 retransmit timers);
- the earliest time a packet is repeated preemptively. A packet is repeated no sooner than `ACK_DELAY_MAX` after it
  was sent and no later than RTT/2, so **below an RTT of 200 ms preemptive repeat never happens**. The slipstream
  client turns preemptive repeat on (`do_preemptive_repeat`). The old fork has the same code and value, so this is
  how the deployed client behaves today. The soak test should say whether 10 ms or 100 ms is better.

## Tests

Run `picoquic_ct` one test at a time, with a time limit per test. The upstream helper of `direct_receive` loops forever
once a stream has two holes (the loop that looks for the last hole never advances), and the series' loss tuning
triggers that. Skip `direct_receive`, or give it a limit.

On this tag, the plain upstream tree fails 15 tests in a sandbox without IPv6 (the socket tests). With the series,
167 more fail, at the default enforced MTU (129) and at slipstream's 64 alike. None of them is a regression: each one
passes again when the value of a deliberate patch is set back (checked by building the series with each value
reverted in turn):

| Cause | Tests | Why |
|---|---|---|
| Patch 2, client Initials no longer padded to 1200 | 124 (70 of them fail for this alone) | The server's 3x budget holds back its first flight, so handshakes take more round trips: timing limits, reference logs (qlog, binlog, packet traces) and fixed loop budgets fail |
| Patch 2, the 0-RTT window (10 x the enforced MTU) | `zero_rtt_delay`, `zero_rtt_long` | They expect at least 3 0-RTT packets; the window holds 1 |
| Patch 2, tests built around 1200 bytes | `bad_chello`, `port_blocked`, `stateless_reset_handshake`, `transport_param`; at 64 also `wrong_source_addr_retry` | Buffers of `PICOQUIC_ENFORCED_INITIAL_MTU` bytes; an unpadded Initial that the server does not answer at once (3 × 336 bytes is less than its 1252-byte send MTU); a VN now drawn by a 256-byte unknown version; a `max_packet_size` below 1200 |
| Patch 10, ack and loss tuning | 66 (18 alone) | Mostly timing; `ec2f_second_flight`, `ec9a_preemptive_amok`, `multipath_back0` pass with `ACK_DELAY_MAX` at 10 ms alone |
| Patches 2 and 10 together | 13 at 129, 12 at 64 | Timing limits that each patch alone stays under |
| Patch 9, cubic | 7 | Tests of the cubic constants and window floor |
| Patch 11, fragments of 8 bytes | `dataqueue_packet`, `perflog` | |
| Patch 12, reorder factor 64 | `stream_data_nodes_attack`, `stream_data_nodes_limit`, `direct_receive_duplicate` | They check the bound this patch loosens |

With every one of these values set back (enforced MTU 1200 and patches 9 to 12 reverted), the result is the plain
tag's 15 failures: the other patches and the fixes on top fail no test. The tests themselves are not changed to hide
these failures.

## Upstream defenses weakened on purpose

- **Reorder memory bound (patch 12).** Upstream closes a connection when the out-of-order stream data nodes × 1536
  exceed 6 × the free receive window (`b661cd5b`, `f0b23c1a`). The fork uses 64, which allows about 10 times as
  many nodes. The reason: a DNS query carries 30 to 120 bytes and resolvers reorder queries, so a factor of 6 closed
  normal connections after about 26 KB. Memory is still bounded; the old fork had no bound at all.
- **Per-tuple address check on PATH_CHALLENGE (patch 8).** With multipath, a challenge that matches no tuple is
  answered on the path's first tuple, where upstream ignores it. Resolvers change source addresses. The challenge
  is encrypted, and the response goes to an address the path already knows, never to the unverified source.
- **The 1200-byte Initial floor of RFC 9000 (patch 2).** Datagrams down to the enforced MTU are accepted as
  Initials, and they can draw a Version Negotiation, Retry, busy reply or stateless reset from picoquic. The 3×
  amplification limit on connection data is unchanged. This is acceptable behind niko-server's DNS front:
  - The slipstream server receives QUIC only inside DNS queries, answers each query once, and sends nothing else.
  - It never sends picoquic's stateless packets: `server_drop_stateless()` deletes the whole queue (VN, Retry, busy,
    stateless reset) after each batch of queries. Checked with a debugger: a 64-byte unknown-version query makes
    picoquic build a VN, and a short header with an unknown connection ID makes it build a reset; both are deleted,
    and the query gets the server's empty answer, the same size as the query.
  - In a tagged zone (`"tag": "slipstream"`), the front passes a query on only if it carries a valid dnsauth-v2 tag
    of a known device over the QUIC packet, and only once (copies get NOERROR). Anyone else gets the front's own
    NOERROR, the size of the query; picoquic never sees it.
  - In an untagged zone, and on the server's own UDP port, anyone can start a handshake. Each answer is one QUIC
    datagram of the connection, within the 3× budget of the QUIC bytes received, and the front sends it only to the
    query that waits for it, within the requester's size. Measured on loopback: a client's first three queries carried
    405 bytes of QUIC (778 bytes of DNS); the answers carried 900 bytes of QUIC (1694 bytes of DNS). By the 14th
    answer, the queries had carried 1812 bytes of QUIC and the answers 1888. Each query carries its QUIC packet in
    base32 (1.6 bytes of name for each byte), and each answer echoes the question, so the answers' DNS bytes stay below
    about 3 times the queries' DNS bytes.
  - What the floor still protected: a sender with a spoofed address now pays 64 bytes, not 1200, for each
    handshake it opens. The server's own limits (`-m`, per device, the reaper of pending handshakes) bound that
    state.

## 0-RTT

`PICOQUIC_DEFAULT_0RTT_WINDOW` is upstream's `10 * PICOQUIC_ENFORCED_INITIAL_MTU`: 1290 bytes at 129, 640 at 64
(upstream 12000). It limits the 0-RTT data a client sends before the handshake ends. It does not matter to slipstream
today: the client makes one connection per process and keeps no ticket file, so it never has a ticket and never sends
0-RTT. The server accepts 0-RTT from any client that has a ticket (picoquic's default); slipstream could turn that off
with `picoquic_set_accept_0rtt(quic, 0)`.

Patch 2 also lowered `PICOQUIC_MIN_SEGMENT_SIZE` to 32. With the small window, the room left for a 0-RTT packet could
be more than 32 bytes but less than its header and checksum, and the client sent a 0-RTT packet without frames, which
the server closes the connection for (PROTOCOL_VIOLATION). The fix on top sends nothing then. The old fork has the
same window and segment size, so it has the same bug, which slipstream never reaches.

## Rebase policy

- Rebase only onto dated upstream tags (`vYYYY.MM.DD.<hash>`), about once a quarter.
- When upstream publishes a security fix, rebase at once.
- Rebase the whole series onto the new tag, keeping one commit per patch. Build it, then run `picoquic_ct` and
  compare the result with the plain tag. Then run slipstream's tests against it before slipstream moves its pin.
- Never rewrite or delete a commit that something pins. A rebase goes on a new commit line, and `slipstream` moves
  to it.

## Wire note: moving from the old fork

Upstream moved multipath from the draft-11 code points to the draft-20 ones (`6eb6c726`). An old client works with
a new server, but a new client does not work with an old server. So roll out in this order:

1. The slipstream server, built on this fork.
2. Then the client (Niko), built with the **same** `PICOQUIC_ENFORCED_INITIAL_MTU` as the server. A server drops
   client Initials below its own value.

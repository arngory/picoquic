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
| `slipstream-port` | The base tag plus the patch series below. The slipstream server and Niko pin this line. |

## Patches

Each patch is one commit. Its message says what the patch changes and why.

1. `slipstream: build flags for embedding in the slipstream builds`: no `-O3` and no `-fno-exceptions`; the flags are exported to `PARENT_SCOPE` when picoquic is included with `add_subdirectory`.
2. `slipstream: lower the enforced initial MTU, settable at build time`: `PICOQUIC_ENFORCED_INITIAL_MTU` is 64 by default (it was 129 until round 3), in [64, 1200], and can be set with the CMake cache variable (`PUBLIC` define); `PICOQUIC_MIN_SEGMENT_SIZE` is 32. See "The enforced MTU at build and link time".
3. `slipstream: picoquic_set_initial_send_mtu and config fields`: `initial_send_mtu_ipv4`/`_ipv6` in the config struct and in the QUIC context. Slipstream sets the struct fields; there is no command-line option for them. `picoquic_set_initial_send_mtu` returns an error for a value that is not 0 and outside [`PICOQUIC_ENFORCED_INITIAL_MTU`, `PICOQUIC_MAX_PACKET_SIZE`], and `picoquic_create_and_configure` then fails.
4. `slipstream: POLL frame (type 0x20)`: a frame that is not ack-eliciting, sent on `cnx->is_poll_requested`. 0x20 is unassigned in IANA's QUIC frame-type registry, between 0x1f (IMMEDIATE_ACK) and 0x24 (RESET_STREAM_AT); the one-byte range is assigned by Standards Action, so a future RFC may take it.
5. `slipstream: at most 128 segments in one GSO send`: since round 3, `PICOQUIC_MAX_GSO_SEGMENTS` (64, Linux's limit up to 6.6).
6. `slipstream: path-id variants of the incoming, prepare and probe calls`: the `_path_ex` and `_path_id_ex` functions; the upstream signatures are kept. A named path that cannot take a packet (demoted, abandoned, its challenge failed, no connection ID of the peer) is ignored, and picoquic chooses. A path index is valid only until the next picoquic call on the connection.
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
| Round 3: a Version Negotiation only from a server, with both connection IDs inside the datagram (B1, below) | 2 |
| Round 3: `picoquic_set_initial_send_mtu` takes 0 or [enforced MTU, max packet size], and says so | 3 |
| Round 3: no packet without frames, nor one that does not fit, of any type (B2, below) | 2 |
| Round 3: a named path is used only if a packet can go on it | 6 |
| Round 3: at most 64 GSO segments (`PICOQUIC_MAX_GSO_SEGMENTS`) | 5 |
| Round 3: an acked POLL packet counted once in `data_acked` (it was counted once more for each POLL) | 4 |
| Round 3: enforced MTU 64 by default, `#error` outside [64, 1200], the link-time tag | 2 |
| Round 3: the reorder-bound tests use `PICOQUIC_REORDER_MEMORY_FACTOR` | 12 |

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

On this tag, the plain upstream tree fails 15 tests in a sandbox without IPv6 (the socket tests). With the series
and the fixes of round 3, 163 more fail at the enforced MTU of 64 (now the default), in Release and in a Debug build
with ASan and UBSan alike, with no sanitizer report. None of them is a regression: each one passes again when the
value of a deliberate patch is set back (round 2 checked this by building the series with each value reverted in
turn; round 3 changed the set only by the four tests it fixed: `stateless_reset_handshake`, and the three reorder
tests, which now check the fork's bound):

| Cause | Tests | Why |
|---|---|---|
| Patch 2, client Initials no longer padded to 1200 | 124 (70 of them fail for this alone) | The server's 3x budget holds back its first flight, so handshakes take more round trips: timing limits, reference logs (qlog, binlog, packet traces) and fixed loop budgets fail |
| Patch 2, the 0-RTT window (10 x the enforced MTU) | `zero_rtt_delay`, `zero_rtt_long` | They expect at least 3 0-RTT packets; the window holds 1 |
| Patch 2, tests built around 1200 bytes | `bad_chello`, `port_blocked`, `transport_param`, `wrong_source_addr_retry` | Buffers of `PICOQUIC_ENFORCED_INITIAL_MTU` bytes; an unpadded Initial that the server does not answer at once (3 × 336 bytes is less than its 1252-byte send MTU); a `max_packet_size` below 1200 |
| Patch 10, ack and loss tuning | 66 (18 alone) | Mostly timing; `ec2f_second_flight`, `ec9a_preemptive_amok`, `multipath_back0` pass with `ACK_DELAY_MAX` at 10 ms alone |
| Patches 2 and 10 together | 13 at 129, 12 at 64 | Timing limits that each patch alone stays under |
| Patch 9, cubic | 7 | Tests of the cubic constants and window floor |
| Patch 11, fragments of 8 bytes | `dataqueue_packet`, `perflog` | |

With every one of these values set back (enforced MTU 1200 and patches 9 to 12 reverted), the result is the plain
tag's 15 failures: the other patches and the fixes on top fail no test. The tests themselves are not changed to hide
these failures. The one change to upstream tests: `stream_data_nodes_attack`, `stream_data_nodes_limit` and
`direct_receive_duplicate` compute their trigger with `PICOQUIC_REORDER_MEMORY_FACTOR` instead of a literal 6, so
they check the bound the fork has (with the bound doubled in `frames.c`, all three fail).

Round 3 added five tests: `vn_bounds`, `initial_send_mtu_bounds`, `small_mtu_handshake`, `path_id_override` and
`poll_ack_count`. Each fails on the code before its fix. picoquic_ct never sets small send MTUs otherwise, so
`small_mtu_handshake` is the only test of the paths that QUIC over DNS takes at small MTUs.

## Upstream defenses weakened on purpose

- **Reorder memory bound (patch 12).** Upstream closes a connection when the out-of-order stream data nodes × 1536
  exceed 6 × the free receive window (`b661cd5b`, `f0b23c1a`). The fork uses 64, which allows about 10 times as
  many nodes. The reason: a DNS query carries 30 to 120 bytes and resolvers reorder queries, so a factor of 6 closed
  normal connections after about 26 KB. Memory is still bounded; the old fork had no bound at all.
  - What the bound is in bytes: every out-of-order fragment but a packet's last takes a whole 1600-byte node
    (`sizeof(picoquic_stream_data_node_t)`), even for one byte of data. So one connection can pin about
    64 / 1536 × 1600 = 67 bytes of memory for each byte of free receive window: 70 MB at upstream's default window of
    1 MiB (43,691 nodes).
  - The window is not fixed unless the application fixes it. Without `picoquic_set_max_data_control` (the config's
    `flow_control_max`), picoquic raises MAX_DATA as data arrives: in the test simulator, 4 MB sent in order took the
    free window from 1 MiB to 7.9 MB (a bound of 524 MB), and from 256 KiB to 17.7 MB.
  - The slipstream server therefore fixes its window at 256 KiB (`flow_control_max`, and the same
    `initial_max_data` and `initial_max_stream_data_*`): at most 10,923 nodes, **17.5 MB per connection**, 1.1 GB at
    `-m 64`. Other users of the fork should do the same, or keep upstream's factor.
- **Per-tuple address check on PATH_CHALLENGE (patch 8).** With multipath, a challenge that matches no tuple is
  answered on the path's first tuple, where upstream ignores it. Resolvers change source addresses. The challenge
  is encrypted, and the response goes to an address the path already knows, never to the unverified source.
- **The 1200-byte Initial floor of RFC 9000 (patch 2).** Datagrams down to the enforced MTU are accepted as
  Initials, and they can draw a Version Negotiation, Retry, busy reply or stateless reset from picoquic. The 3×
  amplification limit on connection data is unchanged. This is acceptable behind niko-server's DNS front:
  - The slipstream server receives QUIC only inside DNS queries, answers each query once, and sends nothing else.
  - It never sends picoquic's stateless packets: `server_drop_stateless()` deletes the whole queue (VN, Retry, busy,
    stateless reset) after each batch of queries. Checked with a debugger: a 64-byte unknown-version query makes
    picoquic build a VN (if both of its connection IDs lie inside it: see B1), and a short header with an unknown
    connection ID makes it build a reset; both are deleted, and the query gets the server's empty answer, the same
    size as the query.
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
  - The floor was also the bounds check of the Version Negotiation code (B1, below). That check is now explicit.
  - For other users of the fork, who send picoquic's stateless packets: a Retry is now drawn by a 64-byte Initial and
    is about twice its size; a busy reply is smaller than its trigger, and a stateless reset always is. That Retry
    factor is the one new reflection, and a modest one. A VN is at most the two connection IDs (inside the
    datagram) and 4 bytes for each version, under 2 × a 64-byte datagram.

## Bugs of the series found in round 3

- **B1: Version Negotiation read past the datagram** (since patch 2; the old fork, floor 129, has it too). For an
  unknown version picoquic does not parse the header past the version, and the VN code read both connection ID
  lengths (up to 255 each) and copied the IDs, up to 517 bytes. Upstream's 1200-byte floor held them, the fork's
  does not: a 64-byte datagram made it read up to 453 bytes of heap past its end, and a VN that went out would have
  carried them (9× the datagram) to any address. The slipstream server deletes every stateless packet, so there it
  was an over-read on unauthenticated input without a leak. Now: no VN unless both IDs lie inside the bytes left in
  the datagram, and none at all from a client-only context (RFC 9000, 6.1). The client case mattered: the slipstream
  client sends stateless packets (`picoquic_prepare_next_packet_ex`), so a resolver could have made it send a VN
  back up the tunnel. Upstream builds that VN from a client too, inside its 1200 bytes.
- **B2: packets without frames, and packets that do not fit** (since patch 2's `PICOQUIC_MIN_SEGMENT_SIZE` of 32,
  upstream 256). The room left for a coalesced packet can be less than a header, a frame and the checksum.
  - The server's Initial and Handshake packets then went out with no frames, and the client closes the connection
    for that (PROTOCOL_VIOLATION). In the simulator, both sides at MTUs of 64 to 200 and client CIDs of 0 to 20,
    96 of 528 runs ended so; for example MTU 137 with 8-byte client CIDs. Slipstream's server MTU of 900 did not hit
    it in the sweep; any smaller MTU or a longer client CID can. The 0-RTT case was fixed earlier (see 0-RTT).
  - The handshake PINGs are written without a check of the room left. At server MTUs of 1523 to 1536 (8-byte client
    CIDs), the server's Handshake PING packet was written past the end of a 1536-byte send buffer (a heap overflow);
    at smaller MTUs such a packet goes past the path MTU, inside the buffer.
  - Now picoquic sends nothing in both cases, for every packet type (MTU probes, which are meant to exceed the path
    MTU, excepted).
- **Initial send MTU without bounds** (patch 3): below about 59 the client's Initial padding was a negative
  `memset`, from 59 to 63 the connection never started, and above 1536 packets overflowed picoquic's packet buffer.
  The setter now refuses such values.
- **The named path** (patch 6) was taken even with no connection ID of the peer (a NULL dereference in the short
  header) or when demoted. The slipstream server guards against this itself; the library now does too.
- **GSO** (patch 5): Linux takes at most 64 segments up to 6.6 (128 from 6.12); 128 got EINVAL on Debian 12 and most
  Android kernels.
- **POLL** (patch 4): an acked POLL packet's bytes went into `data_acked` once more for each POLL frame.

## The enforced MTU at build and link time

`PICOQUIC_ENFORCED_INITIAL_MTU` is a protocol constant between the peers: a server drops client Initials below its
value, and a client drops small server Initials below its own. Slipstream uses 64 on both sides, and 64 is now the
default of `picoquic_internal.h` and of the CMake cache variable, so every build path gets it unless it asks for
another value.
- A value outside [64, 1200] stops the build (`#error`, and a CMake error).
- Every file that includes `picoquic_internal.h` references a symbol named after its value,
  `picoquic_built_with_enforced_initial_mtu_<value>`, which the library defines. Code built with another value than
  the library fails to link instead of running with two floors. The reference survives `--gc-sections` with
  `-fdata-sections` (the Android NDK's defaults): it is kept with `retain` or a constructor. Checked with gcc/GNU ld
  and clang/lld; the MSVC form (a `/include` pragma) is not built here.
- The build cannot check the peers. A client's MTU must not be below the server's floor (picoquic now refuses such
  an initial MTU), and both sides of a deployment must use the same value.

## 0-RTT

`PICOQUIC_DEFAULT_0RTT_WINDOW` is upstream's `10 * PICOQUIC_ENFORCED_INITIAL_MTU`: 640 bytes at the default of 64,
1290 at 129 (upstream 12000). It limits the 0-RTT data a client sends before the handshake ends. It does not matter to slipstream
today: the client makes one connection per process and keeps no ticket file, so it never has a ticket and never sends
0-RTT. The server accepts 0-RTT from any client that has a ticket (picoquic's default); slipstream could turn that off
with `picoquic_set_accept_0rtt(quic, 0)`.

Patch 2 also lowered `PICOQUIC_MIN_SEGMENT_SIZE` to 32. With the small window, the room left for a 0-RTT packet could
be more than 32 bytes but less than its header and checksum, and the client sent a 0-RTT packet without frames, which
the server closes the connection for (PROTOCOL_VIOLATION). The fix on top sends nothing then. The old fork has the
same window and segment size, so it has the same bug, which slipstream never reaches. Round 3 found the same bug for
the other packet types (B2) and made the rule general; the 0-RTT code keeps its own check, which also stops it from
waking the sender at once when nothing fitted.

## Known, and left as they are

- `PICOQUIC_ACK_DELAY_MAX` at 100 ms (above): a question for the soak test.
- The spurious-retransmission check is off in the ready state (patch 10, `frames.c`); its comment says "1-RTT
  packets". An ack of a packet already declared lost no longer undoes the CC reduction or marks its data acked (the
  retransmitted copy carries it). Memory stays bounded: such packets leave the queue by age.
- POLL is accepted in Initial and Handshake packets, where RFC 9000 allows only PADDING, PING, ACK, CRYPTO and
  CONNECTION_CLOSE. The sender puts it only in 1-RTT packets.
- An acked POLL sets the path's last acked data frame time to the ack's time, where stream frames use the packet's
  send time.
- Upstream, not the series: the text logger reads past truncated frames (`textlog_*`; slipstream does not enable the
  text log), and `packet.c` dereferences a NULL local CID with 0-byte CIDs and multipath (also on the tag).

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

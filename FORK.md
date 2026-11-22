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

1. `slipstream: build flags for embedding in the slipstream builds`: no `-O3` and no `-fno-exceptions`; the flags are exported to `PARENT_SCOPE`.
2. `slipstream: lower the enforced initial MTU, settable at build time`: `PICOQUIC_ENFORCED_INITIAL_MTU` is 129 by default and can be set with the CMake cache variable (`PUBLIC` define); `PICOQUIC_MIN_SEGMENT_SIZE` is 32.
3. `slipstream: picoquic_set_initial_send_mtu and config fields`
4. `slipstream: POLL frame (type 0x20)`: a frame that is not ack-eliciting, sent on `cnx->is_poll_requested`.
5. `slipstream: at most 128 segments in one GSO send`
6. `slipstream: path-id variants of the incoming, prepare and probe calls`: the `_path_ex` and `_path_id_ex` functions; the upstream signatures are kept.
7. `slipstream: hooks for slipstream's own packet loop`: `before_select`, the `encode`/`decode` parameters, `picoquic_packet_loop_wait_udp`, and `picoquic_open_network_wake_up`.
8. `slipstream: answer path challenges from unknown addresses with multipath`
9. `slipstream: congestion windows in units of the path MTU, cubic tuning`
10. `slipstream: ack and loss-recovery tuning`
11. `slipstream: split retransmitted stream frames down to 8 bytes`: `PICOQUIC_MIN_STREAM_DATA_FRAGMENT` is 8.
12. `slipstream: reorder memory factor 64 (upstream 6)`
13. `slipstream: name the POLL frame in logs and qlog; strerror on send errors`
14. `slipstream: pthread_kill fallback under _POSIX_THREAD_PROCESS_SHARED`: this one is not needed. Drop it at the next rebase (see its message).
15. `slipstream: retry_token_length test buffer sized by the max packet size`: a test-only change. The test overflowed its stack buffer once the MTU was lowered.

Patches 9 and 10 are tuning values carried over from the old fork. They will be measured again in the soak test
through real resolvers, and any that turn out not to be needed will be dropped.

## Tests

With the series, `picoquic_ct` has about 175 failures that the plain tag does not have. They come from the
deliberate changes, mostly from the lower enforced MTU:
- client Initials are no longer padded to 1200 bytes;
- the 0-RTT window is 10 × the MTU;
- many tests have timing limits and reference logs.

The tests themselves are not changed to hide these failures. The upstream test helper of `direct_receive` loops
forever once a stream has two holes, and the series' loss tuning triggers that, so run the suite with a time limit
per test.

## Upstream defenses weakened on purpose

- **Reorder memory bound (patch 12).** Upstream closes a connection when the out-of-order stream data nodes × 1536
  exceed 6 × the free receive window (`b661cd5b`, `f0b23c1a`). The fork uses 64, which allows about 10 times as
  many nodes. The reason: a DNS query carries 30 to 120 bytes and resolvers reorder queries, so a factor of 6 closed
  normal connections after about 26 KB. Memory is still bounded; the old fork had no bound at all.
- **Per-tuple address check on PATH_CHALLENGE (patch 8).** With multipath, a challenge that matches no tuple is
  answered on the path's first tuple, where upstream ignores it. Resolvers change source addresses. The challenge
  is encrypted, and the response goes to an address the path already knows, never to the unverified source.
- **The 1200-byte Initial floor of RFC 9000 (patch 2).** Datagrams down to the enforced MTU are accepted as
  Initials, and they can draw a Version Negotiation, Retry or busy reply. The 3× amplification limit on connection
  data is unchanged. The slipstream server receives QUIC only inside DNS queries and sends only in answer to them.

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

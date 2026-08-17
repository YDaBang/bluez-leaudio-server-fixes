# BlueZ LE Audio fixes for the acceptor (server) role

Four patches against BlueZ 5.87, found while running BlueZ as the **acceptor**
side of a LE Audio unicast link — a router acting as the audio sink/source for a
phone, rather than the usual initiator role of a headset stack.

The acceptor path is less travelled, and these are places where it does not hold
up.  Each patch is described below with the symptom that led to it, why it
happens, and how to reproduce it.

## The patches

| # | Area | What it fixes |
|---|------|---------------|
| 220 | `src/shared/bap.c` | PAC context recompute survives an external PAC owner restart |
| 221 | `profiles/audio/media.c` | Profile services survive an owner restart |
| 222 | `src/shared/bap.c` | Assigned Generic Audio metadata is validated |
| 223 | `src/shared/bap.c` | Server-side Release settles when a transport IO is attached |

223 is the one to read first.  It is a hang, it is reproducible in the existing
unit test harness, and the code has been unchanged since BAP support was first
added.

## 223 — server-side Release never completes

### Symptom

A call ends.  The remote releases the stream.  The ASE never leaves
`RELEASING`, so the next call cannot set it up.  On Android the group is
dropped to inactive by a watchdog roughly 3.5 seconds later, and from then on
the phone will not offer a CIS again until the link is rebuilt.

Nothing logs an error.  Both sides believe they are behaving correctly.

### Cause

`stream_release()` only drives the state machine to completion when the stream
has no IO attached:

```c
if (!stream->io) {
        ...
        stream_set_state(stream, BT_BAP_STREAM_STATE_RELEASING);
        if (cache_config)
                stream_set_state(stream, BT_BAP_STREAM_STATE_CONFIG);
}
```

When an IO *is* attached, completion is left to `stream_io_disconnected()`.
That callback is registered by `bap_stream_io_attach()` and fires when the
transport socket drops.

As the **initiator** that is fine: it started the Release, so it tears the
transport down itself and the callback arrives.

As the **acceptor** the Release is requested by the remote.  Nothing on this
side tears the transport down, so the callback never fires and the stream is
stuck in `RELEASING`.

`bap_stream_state_changed()` shows the same asymmetry — its RELEASING handling
detaches the IO only `if (stream->client)`.

### Fix

Complete the transition on the server path: detach the IO and settle into
`CONFIG`.  Client behaviour is unchanged.

### Reproducing

The patch adds `BAP/USR/SCC/IO-RELEASE-01` to `unit/test-bap.c`.  It configures
a source stream, drives it to `STREAMING`, attaches an IO via
`bt_bap_stream_set_io()`, then has the remote send Release.

**Without the fix the test hangs**, because the expected Codec Configured
notification never arrives.

Note that it must be registered with `test_server_state`, not `test_server` —
only the former installs `cfg->state_func`.  (While writing it we noticed the
existing server-side configs that set `state_func` are registered with
`test_server`, so those callbacks never run.)

## 220, 221 — surviving an external owner restart

These matter when the PAC records and media endpoints are registered over D-Bus
by a separate process rather than built into `bluetoothd`.  When that process
restarts, BlueZ drops state that the remote still believes is present, and the
peer is never told, so it keeps a stale view.

221 is not the same as the upstream UAF fix in
`profiles/audio: fix UAF on external media service teardown` — that one keeps
queues consistent during teardown, while this keeps the registered services
alive across a restart of their owner.

## 222 — metadata validation

Assigned Generic Audio metadata LTVs were accepted without checking the
type/length combinations defined in the Assigned Numbers document.  Malformed
metadata should be answered with the defined ASCS response codes (`0x0A`
Unsupported Metadata, `0x0C` Invalid Metadata) rather than passed through.

## Testing

    BlueZ 5.87
    unit/test-bap: 709/709 pass with all four applied
    package build reproducible: two runs, byte-identical

223 has also been exercised on live hardware: a MediaTek controller acting as
the LE Audio acceptor for a Galaxy S23 over a 32 kHz LC3 unicast link.  Before
the fix every call left the ASE stuck and the next call failed.  After it, two
consecutive calls both released cleanly and the group stayed active.

## Applying

These are the tail of a longer series.  220-223 sit on top of the upstream
fixes that OpenWrt's bluez package carries as patches 201-219 — several of
which are backports of commits that are already in mainline
(`3f283c8e0`, `5c1c679ec`, `13b14db95`, `7f826d003`, and others).  Applied
against a bare 5.87 tree, 220 and 222 will not line up.

Use `patch(1)`, not `git apply`.  The offsets drift by up to ~180 lines
depending on what precedes them, and `git apply` refuses any fuzz:

    git clone git://git.kernel.org/pub/scm/bluetooth/bluez.git
    cd bluez && git checkout 5.87
    for p in /path/to/patches/*.patch; do patch -p1 < "$p"; done

Verified on a clean 5.87 checkout: the full 22-patch series applies with 0
failures, and 223 lands in `src/shared/bap.c`.

If you only want 223, it applies to a bare 5.87 tree on its own.

## Status

Not yet submitted upstream.  These are published here so the analysis and the
reproduction are available; the intent is to send 223 to
`linux-bluetooth@vger.kernel.org` on its own first, since it carries a failing
test and stands by itself.

## License

The patches modify BlueZ and are offered under the same terms, GPL-2.0-or-later.

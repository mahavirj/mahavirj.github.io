---
layout: post
title:  "Network Time Security on the ESP32"
date:   2026-11-12 10:00:00 +0530
categories: security ntp nts esp32 esp-idf esp-emu
excerpt: >-
  Plain SNTP will believe any clock that answers. This post brings Network Time
  Security (RFC 8915) to the ESP32, runs it against public NTS servers, and
  explains what a device gains from time it can verify.
---

![NTS on the ESP32: a chip holding a clock inside a shield](/assets/nts/banner.svg)

Every ESP32 powers up believing it is 1 January 1970. A few seconds later it asks
the network for the real time, and it believes the first answer that arrives.

That answer is a 48-byte UDP packet with no signature on it. Anyone who can sit
between the device and the time server can write their own answer, and the
device has no way to tell the difference.

Network Time Security (NTS) fixes this by adding **authentication** to NTP, and
several operators now run public NTS servers. This post looks at what it takes
for a microcontroller to use them. It presents a small NTS client for ESP-IDF,
tested end to end against public servers, with the logs to show for it.

## Why a device cares what time it is

Time looks like a convenience feature. It is really a security input. A device
uses its clock to decide:

- **Is this certificate still valid?** Every TLS certificate has a start date
  and an end date.
- **Is this token still valid?** Cloud tokens and signed URLs expire.
- **When did this happen?** Logs and audit trails are only as good as their
  timestamps.
- **Is it time to act?** Schedules, door locks and key rotation all follow the
  clock.

If an attacker can move the clock, they can move all of these decisions. Wind
the clock back, and an expired certificate looks fresh again. Wind it forward,
and valid credentials look expired, so the device locks itself out.

## What is wrong with plain SNTP

SNTP is simple, which is why almost every device uses it. The device sends one small
UDP packet and the server sends one back with the time in it. There is no
handshake, no key and no signature.

Think of it as asking a stranger in the street for the time. Most strangers are
honest. You still would not set a bank vault by it.

## NTS in plain English

NTS keeps the same NTP packets and adds two things: a way to agree on keys, and
a seal on every packet. It works in two steps.

![The two steps of NTS: key establishment over TLS, then sealed NTP packets](/assets/nts/nts-flow.svg)

**Step 1: Key establishment.** The device opens a normal TLS 1.3 connection to
the key server and checks its certificate, just as it would for HTTPS. Over
that connection, the two sides agree on a cipher. Both then derive two secret
keys from the TLS session itself, so the keys are never sent anywhere. The
server also hands over a set of **cookies**, usually eight.

**Step 2: Authenticated time.** The device sends an ordinary NTP request over
UDP, with three extras attached: a random unique ID, one cookie, and a seal
computed with its key. The server replies with the time, the same unique ID, and
its own seal.

The cookies are the clever part. A cookie is like a cloakroom ticket. The
server seals the keys inside the ticket and gives it to the device, then forgets
everything. When the ticket comes back with a request, the server opens it, finds
the keys, and can answer. The server stores nothing for each client, which is
how one machine can serve millions of them.

A ticket is used only once. Each reply carries a fresh one, hidden inside the
sealed part, so the device normally keeps eight in its pocket. Because no ticket is
ever seen twice, it is much harder for an observer to follow a device as it
moves between networks.

## The example

The code lives on a feature branch of an ESP-IDF fork:

[`examples/protocols/nts_client`](https://github.com/mahavirj/esp-idf/tree/feat/nts_client_example/examples/protocols/nts_client)

| File | What it does |
| :--- | :--- |
| `main/nts_client.c` | Key establishment over `esp_tls`, then the sealed NTP exchange |
| `main/aes_gcm_siv.c` | AES-128-GCM-SIV (RFC 8452), the preferred cipher |
| `main/aes_siv.c` | AES-SIV-CMAC-256 (RFC 5297), the cipher every NTS server must support |
| `main/nts_example_main.c` | Self tests, Wi-Fi connection, and the polling loop |

A few points about how it is built:

- **ESP-IDF already had most of it.** TLS 1.3, the ALPN extension, the
  certificate bundle, and the TLS key exporter are all there. Two options are
  off by default and must be enabled: `CONFIG_MBEDTLS_SSL_PROTO_TLS1_3` and
  `CONFIG_MBEDTLS_SSL_KEYING_MATERIAL_EXPORT`.
- **The two ciphers were missing.** Mbed TLS does not ship AES-SIV or
  AES-GCM-SIV, so the example implements both on top of the PSA Crypto API.
  Each one checks itself against the test vectors from its RFC at every boot.
- **It is small.** The protocol is about 600 lines of C and the two ciphers
  about 500 lines together.
- **It never stops.** The example asks for the time every 64 seconds and sets
  the system clock after each verified reply.
- **Silence is an answer.** Some public servers do not send an error when a
  request fails. The client treats three silent queries in a row as "start
  again with a new key exchange", and waits longer each time it has to.
- **It knows more than one server.** The example is configured with a short
  list of public NTS servers. If one stops working, it moves on to the next.

## Verifying it without hardware

The example runs on real hardware and has been verified on an ESP32-S3 board.
A board is not needed to try it, though. The logs below come from an emulated
ESP32-C6 in [ESP Emulator](https://github.com/espressif/esp-emulator)
(`esp-emu`). The emulator provides a Wi-Fi access point with the same default
name and password that ESP-IDF examples use, so no configuration is needed.

### 1. Get the example and build it

The example branch is based on the ESP-IDF v6.1 release (it uses Mbed TLS 4.x),
so run these commands in an ESP-IDF v6.1 checkout.

```sh
cd $IDF_PATH
git fetch https://github.com/mahavirj/esp-idf.git feat/nts_client_example
git checkout FETCH_HEAD -- examples/protocols/nts_client

cd examples/protocols/nts_client
idf.py set-target esp32c6
idf.py build
```

### 2. Make one flash image and boot it

```sh
cd build
esptool --chip esp32c6 merge-bin --format raw -o merged.bin @flash_args
esp-emu --chip esp32c6 --firmware merged.bin --net user
```

### 3. Watch the clock jump 56 years

```text
I (107) nts_example: AES-SIV and AES-GCM-SIV self tests passed
I (4107) nts_example: NTS-KE with nts.meta.com:4460
I (4777) nts: NTS-KE done: AEAD 30, 8 cookies of 68 bytes, NTP server time.meta.com:123
I (5077) nts_example: offset +1794467011532509 us, delay 290033 us, stratum 1, cookies 8 (+1)
I (5077) nts_example: Authenticated time: 2026-11-12 07:03:36 UTC
```

Reading the log from the top:

- The key exchange with the first server in the list took well under a second.
  The server chose cipher 30 (AES-128-GCM-SIV), handed over eight cookies, and
  named the time server to use.
- The first sealed reply said the clock was wrong by 1,794,467,011 seconds.
  That is about 56.9 years, the distance from 1970 to today.
- `cookies 8 (+1)` means one cookie was spent and one fresh cookie came back.

In a three-minute run with a query every eight seconds, 22 out of 22 replies
verified.

### 4. Now try to lie to it

A test that only shows success proves little. So, for this test, a temporary
one-line change damaged the device's copy of the server key for exactly one
query.
To the device, that makes the server's real reply look like a forgery. (This
change is not part of the example.)

```text
I (5077) nts_example: offset +1794469602094166 us, delay 241073 us, stratum 1, cookies 8 (+1)
W (11347) nts: ignoring response that failed NTS verification (ESP_ERR_INVALID_MAC)
W (14087) nts_example: NTS query failed: ESP_ERR_TIMEOUT
I (20357) nts_example: offset +66492 us, delay 264961 us, stratum 1, cookies 8 (+2)
```

The reply arrived, the seal did not match, and the device ignored it. It kept
waiting for a real answer until the three-second timeout. The clock was not
touched. On the next query the device asked for two cookies instead of one
(`+2`) to refill its pocket, and carried on.

## Security benefits

Here is what a device gains by moving from SNTP to NTS.

| Question | Plain SNTP | NTS |
| :--- | :--- | :--- |
| Can someone on the path forge the time? | Yes. One UDP packet is enough. | No. Every reply carries a seal that only the real server can produce. |
| Can an old reply be replayed? | Often, yes. | No. Each reply must return the fresh random ID of its request. |
| Does the device know who answered? | No. | Yes. The key server proves its identity with a TLS certificate, as for HTTPS and OTA. |
| Are secrets needed in the factory? | None, because nothing is protected. | None. Fresh keys are created for each device at run time. |
| What happens to a bad reply? | It sets the clock. | It is dropped, and the clock is not touched. |

The last row is the one that matters most. Certificate expiry checks, token
lifetimes, log timestamps and schedules all inherit a clock that an attacker
cannot simply rewrite.

NTS is not magic, and it is fair to say what it does not do:

- **It cannot stop a delay.** An attacker on the path can still hold a real
  packet back for a moment. The client limits the damage with a timeout: with
  the example's three seconds, the error can reach about one and a half
  seconds, but no more.
- **It cannot stop a blockade.** An attacker can drop every packet. The device
  then gets no time at all, which is a failure it can see, unlike a false time
  it cannot.

## The chicken and the egg

There is one honest problem. To check a TLS certificate you need to know the
date. To learn the date securely you need TLS. This is a known limit of NTS,
and RFC 8915 gives it a section of its own
([8.5, "Initial Verification of Server Certificates"](https://www.rfc-editor.org/rfc/rfc8915#section-8.5)).

On first boot the clock says 1970, so the certificate's validity dates cannot be
checked. ESP-IDF skips that check by default
(`CONFIG_MBEDTLS_HAVE_TIME_DATE` is off), and the example relies on this for its
first connection. Everything else about the certificate is still verified: the
signature chain, the trusted root, and the host name.

A product should narrow this gap. The RFC lists several ways to do it. Two
simple ones: save the last known good time so that a reboot does not start
again from 1970, and refuse any certificate that expired before that saved
time. Refusing any time earlier than the firmware build date is another cheap
check.

## Key takeaways

- **Time is a security input.** Certificates, tokens and logs all depend on it.
- **Plain SNTP trusts anyone.** One forged UDP packet can move the clock.
- **NTS seals every packet** with keys that come from a TLS session, and it
  needs no secrets in the factory.
- **An ESP32 can do it today.** ESP-IDF already has the TLS pieces. The example
  adds about 1,300 lines of C.
- **You can check all of this yourself** in a few minutes, on a board or in an
  emulator without any hardware.

## Links

- NTS at Meta, a description of a public NTS service: <https://engineering.fb.com/2026/10/06/production-engineering/nts-authenticated-time-at-meta/>
- RFC 8915, Network Time Security for NTP: <https://www.rfc-editor.org/rfc/rfc8915>
- RFC 8452, AES-GCM-SIV: <https://www.rfc-editor.org/rfc/rfc8452>
- RFC 5297, AES-SIV: <https://www.rfc-editor.org/rfc/rfc5297>
- The example: <https://github.com/mahavirj/esp-idf/tree/feat/nts_client_example/examples/protocols/nts_client>
- ESP Emulator: <https://github.com/espressif/esp-emulator>

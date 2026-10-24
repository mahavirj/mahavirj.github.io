---
layout: post
title:  "The Revocation That Never Arrives: eMudhra, Forged Chains, and the ESP32"
date:   2026-10-24 10:00:00 +0530
categories: security tls esp32 esp-idf esp-emu
---

On 17 October 2026, a security researcher found an alarming flaw in eight
certificates issued by the Indian certificate authority eMudhra: their private keys
were so weak they could be calculated directly from public keys in under two
hours. Six were intermediate certificates under eMudhra's `emSign ECC Root CA - G3`
capable of issuing credentials for any domain on the internet, and Sectigo had
cross-signed the exact same keys under its `AAA Certificate Services` root
(which Mozilla had previously retired for websites).

The web responded in days. eMudhra and Sectigo revoked the intermediates, Mozilla
removed `emSign ECC Root CA - G3` from Firefox, and Chrome, Linux, Python, and
curl followed. Within a week, the global web was protected.

A microcontroller sitting on a factory floor was not. It still trusts them.

This post explains why certificate revocation fails on embedded devices, how
the upstream certificate bundle snapshot behaves on ESP32, what it means for
device security and regulatory compliance, and how you can test your own
firmware in minutes without hardware. Links to primary sources are at the end.

## Five words you need

If you work with TLS every day, skip ahead. For everyone else, here is the
vocabulary in plain English:

- **Certificate:** A digital ID card proving a server (like `ota.example.com`)
  is authentic.
- **Certificate Authority (CA):** A trusted issuer (like eMudhra or Let's
  Encrypt) that signs these ID cards.
- **Certificate Chain:** Roots rarely sign servers directly; they sign an
  **intermediate CA**, which signs the server. The server presents its
  certificate along with the intermediate(s) so your device can trace a path to
  a trusted root.
- **Trust Store:** The pre-installed list of trusted root CAs baked into device
  firmware. If a chain leads to a root on this list, the connection is trusted.
- **Revocation:** A CA declaring an issued ID cancelled before expiration
  because its key was compromised.

## Revocation is a story browsers tell themselves

Revocation is not an inherent property of a certificate. It is a property of
whoever checks it. A certificate is only cancelled if the client actively checks
its status.

Browsers verify revocations continuously. Firefox downloads OneCRL in the
background, Chrome pushes CRLSets, and both ship fresh root stores every few
weeks. The modern web assumes a client that is online, phones home regularly,
and updates itself silently.

Embedded devices have none of this. Their trust store is frozen into flash
memory at build time, and classic revocation mechanisms are impractical due to
resource constraints:

- **CRLs (Certificate Revocation Lists):** Full lists of cancelled certificates
  are too large for constrained embedded devices to fetch, store, and parse for
  every CA.
- **OCSP (Online Certificate Status Protocol):** Querying CAs during every TLS
  handshake adds latency, drains bandwidth, and fails if the CA server is
  unreachable. Even modern browsers have largely retired live OCSP.

In ESP-IDF, mbedTLS lacks OCSP support, and `esp_crt_bundle` has no CRL parser.
Revocation is disabled by design. The device trusts what was baked into its
firmware until someone manually flashes a new build over the air.

This is not a bug in an SDK—it is the reality of constrained hardware. The
eMudhra incident just makes it unusually obvious, because the attack requires
nothing but public data.

## How the ESP32 validates certificates

An ESP32 running ESP-IDF connects over TLS using `esp_crt_bundle`, which compresses
Mozilla's root list into ~55 KB of flash (roughly 120 roots).

Suppose a fleet of devices connects to `ota.example.com` for firmware updates.
An attacker calculates the weak eMudhra intermediate key, creates a forged
certificate for `ota.example.com`, and hosts a rogue server.

Here is what happens when an unpatched device connects:

![How the ESP32 checks the forged eMudhra chain and which check is missing](/assets/cert-revocation/chain-check.svg)

Three checks run, and all three pass:

1. **Did certificate 2 sign certificate 1?**  
   **Yes.** The attacker signed the server certificate with the leaked key.
   The mathematics is completely valid.
2. **Did certificate 3 sign certificate 2?**  
   **Yes.** eMudhra genuinely issued that intermediate in 2025.
3. **Does the chain reach a trusted root?**  
   **Yes.** The device matches "emSign ECC Root CA - G3" in its bundled store.

The fourth check—*Was certificate 2 revoked?*—never runs. There is no list to
query and no code to ask. The device logs `Certificate validated` and connects.

*(Note: The attacker must send the intermediate certificates; a bare server
certificate fails. The same attack also works via Sectigo's cross-signed
intermediate, chaining to "AAA Certificate Services" instead of the eMudhra
root.)*

## The real-world impact: What an attacker can actually do

A forged certificate alone does not hijack a device; an attacker must first
intercept device traffic (a Man-in-the-Middle attack). While difficult on secured
desktops, IoT devices are uniquely vulnerable:

- **Unsegmented Networks:** Devices often sit on shared factory, commercial, or
  guest Wi-Fi networks where ARP spoofing and DHCP poisoning are trivial.
- **Unauthenticated DNS:** Microcontrollers routinely trust local DHCP DNS
  assignments without DNSSEC or encryption (DoH/DoT). A rogue Wi-Fi AP or router
  can instantly redirect connections.

Normally, TLS certificate validation blocks network attacks: even if redirected,
the device rejects the impostor. **Leaked intermediate keys completely destroy
this defence.** Worse, attackers no longer need to extract keys or craft chains
manually—standard interception tools like `mitmproxy` already ship the
compromised eMudhra keys behind a built-in CLI flag, turning local-network
redirection into a one-command TLS interception attack. Once bypassed, the
fallout is severe:

1. **Credential & Telemetry Interception:** Decrypting traffic exposes API keys,
   tokens, device credentials, and sensitive sensor streams in plaintext.
2. **Malicious Command Injection:** Attackers can fake cloud responses to trigger
   relays, unlock doors, or disrupt industrial controls.
3. **Fleet Takeover (Rogue OTA):** If firmware updates rely only on HTTPS
   without cryptographic image signing (Secure Boot), an attacker can push
   malicious firmware—permanently bricking devices or enlisting them into
   botnets.

## Which builds are exposed

ESP-IDF itself has no bug here—`esp_crt_bundle` simply ships a copy of Mozilla's
upstream root store synced at the time of each release. What matters is the
ESP-IDF release, the bundle set selected
(`CONFIG_MBEDTLS_CERTIFICATE_BUNDLE_DEFAULT_FULL` or `_CMN`), and whether
cross-signed verification is enabled
(`CONFIG_MBEDTLS_CERTIFICATE_BUNDLE_CROSS_SIGNED_VERIFY`, 6.x only).

I tested the stock ESP-IDF `https_request` example against Dennis Jackson's
forged test servers at `we-have-a-problem.com`—alongside a baseline request to
`howsmyssl.com` to confirm normal HTTPS connectivity—using an emulated ESP32-C3:

| ESP-IDF (pre-sync bundle) | Bundle Set | Cross-Signed Verify | Forged via emSign Root | Forged via AAA Root | Normal HTTPS (`howsmyssl.com`) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **v5.x (tested v5.5.5)** | FULL (default) | Not available | **Accepted** | **Accepted** | OK |
| **v6.x / master** | FULL (default) | On (default) | **Accepted** | Rejected | OK |
| **v6.x / master** | FULL | Off | **Accepted** | **Accepted** | OK |
| **v6.x / master** | CMN subset | On | Rejected | Rejected | OK |
| **v6.x / master** | CMN subset | Off | Rejected | **Accepted** | OK |

*"Accepted"* means the handshake succeeded and the host returned: *"Your client
successfully connected to the site. That's bad!"*

Key findings:
- **v5.x bundles carry both roots:** The 5.x bundle copy was synced from Mozilla
  in early 2025 before AAA was retired, and cross-signed verification toggles do
  not exist in 5.x.
- **v6.x carries the eMudhra root in pre-sync bundles:** The AAA root was removed
  from the main list, but an auxiliary file (`cacrt_local.pem`) re-introduces it
  whenever `CROSS_SIGNED_VERIFY` is disabled.
- **The CMN subset rejects eMudhra by luck, not design:** The smaller preset
  simply omits the eMudhra root to save flash space. Production apps using `FULL`
  carry the complete Mozilla snapshot.

## Reproducing it without hardware

You can verify your exposure without a development board using
[ESP Emulator](https://github.com/espressif/esp-emulator) (`esp-emu`),
Espressif's emulator for the ESP32 series.

### 1. Install the emulator

```sh
curl -fsSL https://raw.githubusercontent.com/espressif/esp-emulator/main/install.sh | sh
esp-emu --version
```

### 2. Patch the example

Apply [this patch](/assets/cert-revocation/https_request_emudhra.patch) to
`examples/protocols/https_request` in your ESP-IDF directory:

```c
struct emudhra_host { const char *name; const char *host; const char *path; };
static const struct emudhra_host emudhra_hosts[] = {
    { "FORGED-G3",  "dv-g3b-emudhra.we-have-a-problem.com",     "/" },
    { "FORGED-AAA", "dv-g3b-aaa-emudhra.we-have-a-problem.com", "/" },
    { "CONTROL",    "www.howsmyssl.com",                         "/a/check" },
};

static void https_emudhra_matrix(void)
{
    for (int i = 0; i < sizeof(emudhra_hosts) / sizeof(emudhra_hosts[0]); i++) {
        const struct emudhra_host *h = &emudhra_hosts[i];
        char url[160], req[320];
        snprintf(url, sizeof(url), "https://%s%s", h->host, h->path);
        snprintf(req, sizeof(req), "GET %s HTTP/1.1\r\nHost: %s\r\n"
                 "User-Agent: esp-idf/1.0 esp32\r\nConnection: close\r\n\r\n", url, h->host);
        ESP_LOGI(TAG, "EMUDHRA_TEST %s BEGIN %s", h->name, h->host);
        esp_tls_cfg_t cfg = { .crt_bundle_attach = esp_crt_bundle_attach };
        https_get_request(cfg, url, req);
        ESP_LOGI(TAG, "EMUDHRA_TEST %s END", h->name);
    }
    ESP_LOGI(TAG, "EMUDHRA_MATRIX_DONE");
}
```

```sh
cp -r $IDF_PATH/examples/protocols/https_request emudhra-check
cd emudhra-check
patch -p1 < https_request_emudhra.patch
```

### 3. Build the configuration

Add [this fragment](/assets/cert-revocation/sdkconfig.emudhra) next to
`sdkconfig.defaults` to configure emulator Wi-Fi and the full bundle:

```ini
CONFIG_EXAMPLE_CONNECT_WIFI=y
CONFIG_EXAMPLE_WIFI_SSID="myssid"
CONFIG_EXAMPLE_WIFI_PASSWORD="mypassword"
CONFIG_MBEDTLS_CERTIFICATE_BUNDLE_DEFAULT_FULL=y
CONFIG_MBEDTLS_CERTIFICATE_BUNDLE_CROSS_SIGNED_VERIFY=y
```

```sh
export SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.emudhra"
idf.py set-target esp32c3
idf.py build
idf.py merge-bin -o merged.bin
```

*(For v5.x builds, run inside `docker run -it -v $PWD:/work -w /work espressif/idf:release-v5.5`.)*

### 4. Run the test

```sh
esp-emu --chip esp32c3 \
  --firmware build/merged.bin \
  --elf build/https_request.elf \
  --timeout 150s \
  --exit-on "EMUDHRA_MATRIX_DONE"
```

With the pre-sync certificate bundle on ESP-IDF v6.x:

```text
I (4739) example: EMUDHRA_TEST FORGED-G3 BEGIN dv-g3b-emudhra.we-have-a-problem.com
I (7429) example: Connection established... (TLS HANDSHAKE OK)
Your client successfully connected to the site. That's bad!
I (18929) example: EMUDHRA_TEST FORGED-AAA BEGIN dv-g3b-aaa-emudhra.we-have-a-problem.com
I (20119) esp-x509-crt-bundle: No matching trusted root certificate found
E (20119) example: Connection failed... (TLS HANDSHAKE FAILED)
I (31119) example: EMUDHRA_TEST CONTROL BEGIN www.howsmyssl.com
I (32389) example: Connection established... (TLS HANDSHAKE OK)
```

In v5.5.5, both forged chains connect successfully. Running against the updated
bundle confirms both forged endpoints fail while the control endpoint succeeds.

## What the fix looks like, and what it costs

Because ESP-IDF's certificate bundle stays in sync with Mozilla's upstream trust
store, the fix simply syncs the bundle with current Mozilla data (removing
eMudhra) and drops AAA from `cacrt_local.pem`.

> **This update was merged into ESP-IDF master in commit
> [`2fbca5a4`](https://github.com/espressif/esp-idf/commit/2fbca5a49b34d12d565acfaa180ab52d286930ae)
> and is being backported across supported release branches (v5.3 through v6.1).**

Two apparent shortcuts were deliberately avoided:
- **Don't park compromised roots in the deprecated list:** `cacrt_deprecated.pem`
  is for lifecycle retirements where keys remain safe. Moving publicly compromised
  keys there turns a compatibility option into an attacker backdoor.
- **Don't rely on issue-date cut-offs:** Rejecting only certificates issued after
  the leak date does not work—attackers with the private key can simply backdate
  forged certificates.

**The operational cost:** Legitimate servers still chaining to eMudhra will fail
until their operators switch CAs. That is the price of revocation: when an issuer
falls, innocent certificates under it fall too.

## The bigger picture: Continuous security updates

For decades, the embedded industry operated under "flash-and-forget": flash the chip
at the factory and never touch it again.

Incidents like eMudhra make it clear that this model is dead. Even bug-free firmware
decays as the cryptographic environment evolves around it: CAs get breached,
algorithms weaken, and roots expire.

Continuous security updates are no longer just good engineering hygiene—they are
also a regulatory requirement. Cybersecurity regulations worldwide now mandate
vulnerability handling and verifiable over-the-air update mechanisms across a
connected product's lifetime. Shipping a device without an active, tested update
path is both a security exposure and a compliance failure.

## Key takeaways

- **Your trust store is frozen at build time:** ESP-IDF's certificate bundle is a
  snapshot synced from Mozilla upstream. Treat bundle refreshes as security
  updates on their own cadence rather than waiting for major SDK upgrades.
- **Continuous security updates are mandatory:** Post-market security maintenance
  and verifiable OTA update paths are essential for both field security and
  regulatory compliance.
- **Never rely on HTTPS alone for OTA:** Leaked intermediate keys allow attackers
  on the network path to forge server certificates. Enforce hardware Secure Boot
  and cryptographic image signing so a MitM attacker still cannot flash rogue
  firmware.
- **Verify the bundle you actually ship:** Inspect the generated artifact
  (`build/esp-idf/mbedtls/x509_crt_bundle`), not just source PEM files—build
  flags and presets change what lands in the image.
- **Consider lightweight intermediate blocklists:** A small hash list of
  known-bad intermediate keys (a few dozen bytes per entry) can block
  compromised intermediates without revoking an entire root.
- **Emulators make testing immediate:** Use ESP Emulator (`esp-emu`) locally or in
  CI pipelines to verify certificate bundles against test endpoints without
  hardware.

## Links

- Incident report, Mozilla Bugzilla: <https://bugzilla.mozilla.org/show_bug.cgi?id=2083449>
- Removal of emSign ECC Root CA - G3: <https://bugzilla.mozilla.org/show_bug.cgi?id=2084101>
- Mozilla's write-up: <https://blog.mozilla.org/security/2026/10/20/protecting-firefox-users-following-the-compromise-of-emudhra-intermediate-certificates/>
- Test hosts, chains and proofs: <https://emudhra.we-have-a-problem.com/> (source: <https://github.com/galadran/we-have-a-problem.com>)
- Mitmproxy 12.3 release notes (`--emudhra`): <https://www.mitmproxy.org/posts/releases/mitmproxy-12.3/>
- ESP Emulator: <https://github.com/espressif/esp-emulator>
- ESP-IDF certificate bundle documentation: <https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/protocols/esp_crt_bundle.html>
- Sample app patch and sdkconfig fragment from this post:
  [`https_request_emudhra.patch`](/assets/cert-revocation/https_request_emudhra.patch),
  [`sdkconfig.emudhra`](/assets/cert-revocation/sdkconfig.emudhra)

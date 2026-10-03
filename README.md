# Flutter Android Application Interception Handbook (v2)
### Burp Suite + Android Emulator/Device + System CA + Transparent Proxy + Runtime Bypass

**Target:** Flutter-based Android applications
**Proxy:** Burp Suite
**Audience:** Authorized mobile application security testers

> **Scope & Authorization**
> Use this workflow only on applications, accounts, devices, and environments you are explicitly authorized to test. Traffic interception, certificate installation, root access, and runtime instrumentation can affect device security posture — never perform this on production devices, third-party accounts, or out-of-scope targets.

---

## 1. What This Handbook Covers

This handbook is a field-tested workflow for intercepting Flutter application traffic with Burp Suite. It goes beyond the "standard" Android interception steps because **Flutter requires extra handling** — its networking stack does not always behave like a normal Android app. This version adds:

- Correct environment selection (and the #1 mistake that wastes the most time)
- Why "Chrome works, the app doesn't" happens specifically with Flutter
- Three independent fix layers (routing, trust, runtime bypass) and when each is actually needed
- A decision tree instead of a single linear path
- Emulator **and** real-device workflows
- Known environment pitfalls (Windows/Hyper-V, Genymotion networking, architecture mismatches)

---

## 2. Read This First: The Single Biggest Time-Waster

**Architecture mismatch is the most common root cause of "nothing works."**

Flutter ships a prebuilt native engine (`libflutter.so`) per CPU architecture (`arm64-v8a`, `armeabi-v7a`, `x86_64`, `x86`). Some patched-engine tools (e.g. reFlutter) publish pre-patched engines only for ARM architectures. If you test on an **x86_64 emulator**, a tool may silently repackage the APK while leaving the **original, unpatched** engine in place for that architecture — the patch step reports success, but nothing actually changes at runtime.

**Practical implication:**
- If you plan to use a statically patched engine (reFlutter or similar), verify the patch tool explicitly confirms support for your emulator's architecture. If it warns "no patched engine available for x64," that binary will behave exactly like the original app.
- If you can't get a matching patched engine, prefer **runtime bypass (Frida)** or **network-layer redirection (iptables)** instead — both are architecture-agnostic in the sense that they don't depend on which engine variant shipped inside the APK.
- Real ARM64 Android devices avoid this problem entirely, at the cost of convenience (see §9).

---

## 3. Environment Options Compared

| Environment | Root/writable-system | Architecture reality | Known friction |
|---|---|---|---|
| Android Studio AVD — **Google Play** image | Not supported (production-like, locked) | Usually x86_64 on Intel/AMD hosts | `adb root`/`disable-verity` fails with "bootloader unlocked" |
| Android Studio AVD — **Google APIs** image | Supported | Usually x86_64 on Intel/AMD hosts | `mount -o rw,remount /` fails — must use `adb remount`, which uses OverlayFS, not a raw rw mount |
| Genymotion | Rooted by default | x86/x86_64, with optional ARM translation layer | VirtualBox + Windows Hyper-V conflict causes DHCP failures, freezes, "System UI isn't responding" |
| Real Android device | Root depends on device (often not rooted) | Native, usually arm64 — matches patched-engine availability | Needs same Wi-Fi network, USB driver/cable issues, Samsung requires a screen lock before enabling USB debugging |

**Recommendation:** Start with an Android Studio **Google APIs** (non–Google Play) image if you need root + writable system quickly on a Windows/Intel host. Move to a real device only if you specifically need ARM-only patched engines or Genymotion's conflicts aren't worth resolving.

---

## 4. Emulator Setup (Google APIs image)

```
emulator -list-avds
emulator -avd <AVD_NAME> -writable-system
```

If another instance of the same AVD is already running, stop it first.

### 4.1 Verify Root

```
adb devices
adb root
adb shell id
```
Expected: `uid=0(root)`

### 4.2 Enable Writable System

```
adb root
adb remount
```

Expected (success) output includes:
```
Verity disabled; overlayfs enabled.
Now reboot your device for settings to take effect
```

Then:
```
adb reboot
```
After reboot, run `adb remount` again — it should now succeed silently/cleanly.

> **Do not** use `mount -o rw,remount /` as your test. On modern Android (dynamic partitions / dm-verity), `/` can remain backed by a read-only `dm-` block device even while `/system`, `/vendor`, `/product` etc. are writable through OverlayFS. Testing the raw `/` mount will give a false negative.

> **If `adb remount` reports "Device must be bootloader unlocked":** you are very likely on a **Google Play** system image, which deliberately restricts this for production-parity testing. Create a new AVD using a **Google APIs** (non–Play Store) system image instead — same API level is fine.

> **If the emulator was started without the `-writable-system` flag:** the GUI "Play" button alone does not add this flag. You must launch via command line as shown above, or `adb remount` will fail even on a Google APIs image.

---

## 5. Configure Burp Suite

**Proxy → Proxy settings → Add/Edit listener:**
- Bind to port: `4444` (or your chosen port — just be consistent everywhere below)
- Bind to address: **All interfaces**
- **Request handling tab:** Enable **"Support invisible proxying for non-proxy-aware clients"** — this matters for native/non-HTTP-proxy-aware clients, and is a common missed setting
- Certificate: Generate CA-signed per-host certificates (default)

If you will test from a **real device** instead of an emulator, also allow this port through **Windows Defender Firewall** (inbound rule), or phone-to-laptop connections will time out even though Burp itself is running correctly.

---

## 6. Configure Android-Side Proxying

For the Android Studio emulator, the host loopback address is **`10.0.2.2`** (not your machine's LAN IP — this is an emulator-only virtual address).

```
adb shell settings put global http_proxy 10.0.2.2:4444
adb shell settings get global http_proxy
```
Expected: `10.0.2.2:4444`

**For Genymotion**, the equivalent host address is typically **`10.0.3.2`**, not `10.0.2.2` — using the wrong address here is a common source of "internet just stopped working."

**For a real device**, use your laptop's actual LAN IP (from `ipconfig`), set via Wi-Fi → network → Advanced → Proxy → Manual, and the phone and laptop must be on the same Wi-Fi network (not a mobile hotspot unless the laptop is the host).

> **If you need to undo this and the internet stops working entirely:** a bad proxy value blocks *all* traffic, not just the target app's. Clear it immediately:
> ```
> adb shell settings put global http_proxy :0
> ```

---

## 7. Install the Burp CA as a System CA

System-level CA installation is required for the proxy to be *trusted*, not just reached. Browser traffic alone being decrypted does not confirm the app will trust the same cert — see §8.

### 7.1 Export and Verify the Burp CA
Burp → Proxy → Proxy settings → Import/Export CA certificate → export as DER.

```
openssl x509 -inform DER -in burpca.der -subject -issuer -noout
```
Expected: `O=PortSwigger ... CN=PortSwigger CA` on both lines.

### 7.2 Compute the Android Hash Filename

```
openssl x509 -inform DER -subject_hash_old -in burpca.der -noout
```
Example output: `9a5ba575` → rename the cert to `9a5ba575.0`

### 7.3 Push and Install

```
adb push 9a5ba575.0 /sdcard/
adb shell
su
mv /sdcard/9a5ba575.0 /system/etc/security/cacerts/
chmod 644 /system/etc/security/cacerts/9a5ba575.0
chown root:root /system/etc/security/cacerts/9a5ba575.0
reboot
```

### 7.4 Verify

```
adb shell
su
ls -l /system/etc/security/cacerts/9a5ba575.0
```
Also check: **Settings → Security → Trusted credentials → System** for the PortSwigger CA entry.

---

## 8. Why "Chrome Works, the App Doesn't" — And What Actually Fixes It

This is the single most common point of confusion in Flutter interception, so it gets its own section.

**What's actually happening:**
1. **Routing problem** — Some Flutter networking paths do not read the Android global HTTP proxy setting at all. Traffic simply never reaches Burp.
2. **Trust problem** — Even when traffic does reach Burp, some Flutter apps validate the TLS certificate using their own bundled logic/trust material rather than (or in addition to) the Android system trust store that Chrome uses. In that case the handshake fails and nothing usable appears in HTTP History, even though the connection attempt happened.
3. **Pinning problem** — The app explicitly pins a certificate or public key and rejects anything else, by design.

**Important correction to a common assumption:** it is *not* universally true that all Flutter apps ignore the system CA store — behavior varies by Flutter version, which networking package the app uses (`dart:io` `HttpClient`, `dio`, `cronet`, platform channels to native HTTP clients, etc.), and whether the developer added custom pinning. **Do not assume which of the three problems above you have — test for it directly, in this order:**

### 8.1 Fix Layer 1 — Network-Layer Redirection (solves the routing problem)

Mandatory for this workflow regardless of which app you're testing, because it costs nothing if unneeded and fixes the most common failure:

```
adb shell
su
iptables -t nat -A OUTPUT -p tcp --dport 443 -j DNAT --to-destination 10.0.2.2:4444
iptables -t nat -A OUTPUT -p tcp --dport 80  -j DNAT --to-destination 10.0.2.2:4444
```

This forces **all** outbound HTTP/HTTPS traffic to Burp at the packet level, independent of whether the app's HTTP client is proxy-aware. It is architecture-independent and works identically on x86_64 and arm64.

> **This rule does not survive a Cold Boot.** A normal Stop/Resume (quick-boot snapshot) preserves it; "Cold Boot Now" resets it and it must be re-added.

To remove when finished:
```
iptables -t nat -D OUTPUT -p tcp --dport 443 -j DNAT --to-destination 10.0.2.2:4444
iptables -t nat -D OUTPUT -p tcp --dport 80  -j DNAT --to-destination 10.0.2.2:4444
```
If `-D` by rule spec fails with "Index of deletion too big" or similar, list with line numbers and delete by index (highest index first), or flush the whole NAT table if you don't need to preserve other rules:
```
iptables -t nat -L OUTPUT -n --line-numbers
iptables -t nat -D OUTPUT <n>
# or, nuclear option:
iptables -t nat -F
```

### 8.2 Fix Layer 2 — System CA Trust (solves the trust problem, for apps that honor it)

This is §7 above. **Test empirically**: launch the app, perform a network action, check Burp HTTP History.

- If traffic now appears → you were only missing Layers 1 and/or 2. Done — proceed to testing (§11).
- If traffic still does not appear, or Burp's **event log/Dashboard** shows TLS handshake failures without a corresponding HTTP History entry → the app likely performs its own certificate validation independent of the system store, or pins. Proceed to Layer 3.

### 8.3 Fix Layer 3 — Runtime Bypass with Frida (solves the trust/pinning problem generically)

This works regardless of *how* the app validates certificates, and regardless of architecture (Frida ships binaries for x86, x86_64, arm, and arm64).

```
# On the device/emulator (rooted):
adb push frida-server-<version>-android-<arch> /data/local/tmp/frida-server
adb shell
su
chmod 755 /data/local/tmp/frida-server
/data/local/tmp/frida-server &

# On the host machine:
pip install frida-tools
frida-ps -U        # confirms the host can see the device

# Identify the target package:
adb shell pm list packages | findstr <part_of_app_name>

# Spawn with a pinning-bypass script:
frida -U -f <package.name> -l flutter_sslpin_bypass.js --no-pause
```

Notes:
- The `frida-server` binary version must match your installed `frida-tools`/`frida` Python package version, or the connection will fail silently or with a version-mismatch error.
- The app must be launched *through* Frida (`-f` spawn mode) each time you want the bypass active — opening it normally from the launcher will not apply the hook.
- A statically patched engine (reFlutter) and a runtime Frida bypass are **alternatives to each other**, not requirements to stack. If one already works reliably for your target, you don't need the other.

### 8.4 Decision Summary

```
Chrome traffic visible in Burp?
  NO  → Burp listener/port/firewall misconfigured. Fix §5–6 first.
  YES → continue

App traffic visible after Layer 1 (iptables) + Layer 2 (system CA)?
  YES → proceed to testing (§11)
  NO  → check Burp event log for TLS handshake failures
          YES (handshake failures logged) → apply Layer 3 (Frida)
          NO  (nothing logged at all)      → re-check iptables rules are present
                                              (did a Cold Boot reset them?),
                                              re-check proxy address value,
                                              re-check app is actually making
                                              network calls (not cached/offline)
```

---

## 9. Real Device Alternative

Useful when: patched-engine architecture availability only covers ARM, or emulator environment issues (Hyper-V conflicts, slow ARM translation) are costing more time than they save.

1. Enable Developer Options (tap Build Number 7×) and USB Debugging.
   - **Samsung devices:** USB debugging stays grayed out until a screen lock (PIN/Pattern/Password) is set — this is enforced by Samsung, not a bug.
2. Connect via USB. If the device only shows as charging with no data-transfer popup, check the USB mode (notification → File Transfer/MTP/PTP, not "Charging only"), try a different cable (many are charge-only), and try a direct port rather than a hub.
3. `adb devices` should show the device as `device` (not `unauthorized` — accept the RSA fingerprint popup on the phone, ideally with "always allow").
4. Root is **not guaranteed** on consumer devices — many modern phones cannot be rooted without an unlocked bootloader and custom recovery, which is a much bigger undertaking and higher risk to the device. If root isn't available, you're limited to:
   - User-level CA install (Settings → Security → Install a certificate) — works for apps that defer to the system/user trust store, but **as of Android 7+, apps must explicitly opt in via network security config to trust user certs**; many apps do not, which is a separate reason traffic may not decrypt even without "real" pinning.
   - Frida, **if** the device is rooted or you use Frida Gadget (an SDK injected into a repackaged APK) as a no-root alternative.
5. Set phone and laptop on the same Wi-Fi, configure manual proxy to the laptop's LAN IP:port, and allow the port through Windows Firewall.

---

## 10. Genymotion Alternative — Known Pitfalls

Genymotion is attractive because it's rooted out of the box, but on Windows hosts it frequently collides with **Hyper-V** (which Windows enables implicitly for WSL2, Docker Desktop, Windows Hypervisor Platform, or Device Guard/Credential Guard).

Symptoms of this conflict:
- "The virtual device did not get any IP address" / DHCP server errors on first boot
- Device freezes at boot or shows "System UI isn't responding"
- `player.exe is not responding` dialogs

Mitigations, in order of least disruptive first:
1. Set the VM's network mode to **NAT** rather than Bridged, especially if your host's primary adapter is Wi-Fi (bridging frequently fails on Wi-Fi adapters regardless of Hyper-V).
2. Increase allocated RAM (≥4GB) and CPU cores (2–4) in VirtualBox VM settings, and confirm VT-x/AMD-V + Nested Paging are enabled there.
3. If Genymotion/VirtualBox explicitly reports "Hyper-V detected — falls back on software emulation," you have two real options: disable Hyper-V/WSL2/Windows Hypervisor Platform (Windows Features) and reboot — **but this will likely break Android Studio's own AVD acceleration**, which also depends on Windows's hypervisor layer — or accept Genymotion will run in slow software-emulation mode.
4. Given the trade-off in point 3, if you already have a working Android Studio AVD path, it is often faster to continue with that rather than resolve the Hyper-V conflict.

Genymotion's host-to-guest address for proxy configuration is typically **`10.0.3.2`**, not `10.0.2.2`.

---

## 11. Flutter Application Interception Test

1. Launch the application (normally, or via `frida -U -f <package> -l script.js --no-pause` if using runtime bypass).
2. Perform a network action: login, OTP request, search, refresh, open an API-backed screen, submit a form.
3. Check **Burp → Proxy → HTTP History**.

**Expected result:** the application's HTTP/HTTPS requests appear in HTTP History.

If traffic is absent, work through §8.4's decision tree rather than assuming SSL pinning — **no traffic in Burp is not, by itself, evidence of pinning.** It is equally or more often a routing or environment misconfiguration.

---

## 12. Static Indicators of Certificate Pinning (for code/binary review)

- `CertificatePinner` (OkHttp)
- `X509TrustManager` / `checkServerTrusted` custom implementations
- Custom `HostnameVerifier`
- Custom `SSLSocketFactory`
- Custom certificate validation logic
- Native/engine-level TLS implementation (BoringSSL calls inside `libflutter.so` or a custom native library)

Generic presence of OkHttp or standard Android framework TLS classes, by itself, is **not** proof of pinning — look for the custom logic specifically.

---

## 13. Troubleshooting Matrix

| Problem | Likely Cause | Action |
|---|---|---|
| `adb root` / `disable-verity` fails: "bootloader unlocked" | Google Play system image | Recreate AVD with Google APIs (non–Play) image |
| `adb remount` fails after `disable-verity` succeeded | Emulator wasn't started with `-writable-system`, or needs a reboot first | Start via `emulator -avd X -writable-system`; run `adb reboot` then `adb remount` again |
| `mount -o rw,remount /` fails with "read-only" | Expected on dm-verity/OverlayFS systems | Don't use this as a test — use `adb remount` and check `/system` behavior, not `/` |
| Chrome traffic visible, app traffic absent | Routing and/or trust problem specific to the app's networking stack | Apply §8.1 (iptables) then §8.2 (system CA); if still absent, §8.3 (Frida) |
| Internet stops completely on device/emulator | Wrong or unreachable proxy address set globally | `adb shell settings put global http_proxy :0` to clear, then re-verify the correct host address |
| `iptables -D` fails ("Index of deletion too big") | Rule-spec mismatch or busybox iptables quirk | Delete by `--line-numbers` index (highest first) or `iptables -t nat -F` |
| Genymotion: DHCP/IP address errors, freezes, "System UI isn't responding" | Hyper-V/VirtualBox conflict on Windows | Switch to NAT networking; raise RAM/CPU; see §10 for the Hyper-V trade-off |
| Samsung device: USB debugging toggle greyed out | No screen lock configured | Set a PIN/Pattern/Password, then re-enter Developer Options |
| Phone shows "charging only," no debugging popup | Cable without data lines, wrong USB mode, or USB hub | Try a known-good data cable, set mode to File Transfer/PTP, connect directly to a laptop port |
| Patched engine (reFlutter, etc.) has no effect | Target architecture's engine wasn't actually patched (see §2) | Use an architecture the tool supports, or switch to the Frida runtime-bypass route (§8.3) |

---

## 14. Command Cheat Sheet

```bash
# AVD
emulator -list-avds
emulator -avd <AVD_NAME> -writable-system

# ADB / Root
adb devices
adb root
adb shell id
adb remount
adb reboot

# Android Proxy
adb shell settings put global http_proxy 10.0.2.2:4444     # Android Studio AVD
adb shell settings put global http_proxy 10.0.3.2:4444     # Genymotion
adb shell settings get global http_proxy
adb shell settings put global http_proxy :0                 # clear / emergency restore

# Burp CA Hash
openssl x509 -inform DER -subject_hash_old -in burpca.der -noout

# System CA Verification
adb shell
su
ls -l /system/etc/security/cacerts/<hash>.0

# Transparent HTTPS/HTTP Routing
iptables -t nat -A OUTPUT -p tcp --dport 443 -j DNAT --to-destination 10.0.2.2:4444
iptables -t nat -A OUTPUT -p tcp --dport 80  -j DNAT --to-destination 10.0.2.2:4444
iptables -t nat -L OUTPUT -n --line-numbers
iptables -t nat -D OUTPUT <line_number>
iptables -t nat -F                                           # flush all NAT OUTPUT rules

# Frida Runtime Bypass
adb push frida-server-<version>-android-<arch> /data/local/tmp/frida-server
adb shell "su -c 'chmod 755 /data/local/tmp/frida-server && /data/local/tmp/frida-server &'"
frida-ps -U
adb shell pm list packages | findstr <app_name_fragment>
frida -U -f <package.name> -l flutter_sslpin_bypass.js --no-pause
```

---

## 15. Complete Workflow — Quick View

```
Choose environment (Android Studio Google APIs AVD recommended first)
  ↓
Start with -writable-system
  ↓
adb root → adb remount (reboot if first attempt only disabled verity)
  ↓
Burp listener :4444, All interfaces, invisible proxying ON
  ↓
Android/Genymotion proxy → host address (10.0.2.2 or 10.0.3.2)
  ↓
Install Burp CA as System CA → reboot
  ↓
iptables HTTP(S) → host:4444  (mandatory first attempt)
  ↓
Launch Flutter app → perform a network action
  ↓
Traffic in Burp HTTP History?
  ├─ YES → start testing (§11–12)
  └─ NO  → check Burp event log for TLS failures
             ├─ failures logged → apply Frida runtime bypass (§8.3)
             └─ nothing logged  → re-verify iptables/proxy survived reboot,
                                  re-verify app actually made a network call
  ↓
Remove iptables rule when finished (§8.1 cleanup)
```

---

## 16. Final Testing Principle

Build confidence in layers rather than assuming failure means pinning:

1. Confirm Burp listener connectivity (browser test).
2. Confirm Android/Genymotion proxying reaches Burp.
3. Confirm CA trust (system store install).
4. Confirm transparent routing (iptables) as a safety net for proxy-unaware clients.
5. Only then test the Flutter application itself, and only conclude pinning is present after Layer 3 (Frida) also fails to produce traffic **and** static analysis (§12) confirms custom validation logic in the binary.

Treat architecture mismatch (§2) and environment-level networking conflicts (§10) as the first two suspects whenever "nothing is working," before suspecting the application's security controls.

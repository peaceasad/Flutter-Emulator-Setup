# Flutter SSL Interception Toolkit — Burp Suite + System CA + iptables + Frida
**A Small Handbook**
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

## 7. Force All App Traffic Through Burp (iptables)
 
Even with the proxy set (step 6) and the system CA installed (step 7), **many Flutter apps still won't show up in Burp.** This is because Flutter's networking layer frequently ignores Android's global HTTP proxy setting entirely — that setting was only ever built for apps that check it, and plenty of Flutter apps don't.
 
The fix is to redirect traffic at the network level instead, so it doesn't matter whether the app checks the proxy setting or not:
 
```
adb shell
su
iptables -t nat -A OUTPUT -p tcp --dport 443 -j DNAT --to-destination 10.0.2.2:4444
iptables -t nat -A OUTPUT -p tcp --dport 80  -j DNAT --to-destination 10.0.2.2:4444
```
 
This forces **every** outbound HTTP/HTTPS connection from the emulator to Burp, regardless of which app made it or how its code is written.
 
> **This rule does not survive a Cold Boot.** A normal Stop/Resume of the emulator (quick-boot snapshot) preserves it; "Cold Boot Now" resets it and the two commands above must be re-run.
 
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
---

## 8. Install the Burp CA as a System CA
 
System-level CA installation is required for the proxy to be *trusted*, not just reached. Browser traffic alone being decrypted does not confirm the app will trust the same cert — see §8.
 
### 8.1 Export and Verify the Burp CA
Burp → Proxy → Proxy settings → Import/Export CA certificate → export as DER.
 
```
openssl x509 -inform DER -in burpca.der -subject -issuer -noout
```
Expected: `O=PortSwigger ... CN=PortSwigger CA` on both lines.
 
### 8.2 Convert DER → CRT (PEM)
 
Compute the Android hash filename from a PEM-format certificate, not directly from the raw DER file — doing it directly from DER is unreliable across OpenSSL versions/builds. Convert first:
 
```
openssl x509 -inform DER -in burpca.der -out burpca.crt
```
 
### 8.3 Compute the Android Hash Filename
 
```
openssl x509 -inform PEM -subject_hash_old -in burpca.crt -noout
```
Example output: `9a5ba575` → rename the cert to `9a5ba575.0`
 
```
rename burpca.crt 9a5ba575.0
```
(on Linux/macOS use `mv burpca.crt 9a5ba575.0` instead of `rename`)
 
### 8.4 Push and Install
 
```
adb push 9a5ba575.0 /sdcard/
adb shell
su
mv /sdcard/9a5ba575.0 /system/etc/security/cacerts/
chmod 644 /system/etc/security/cacerts/9a5ba575.0
chown root:root /system/etc/security/cacerts/9a5ba575.0
reboot
```
 
### 8.5 Verify
 
```
adb shell
su
ls -l /system/etc/security/cacerts/9a5ba575.0
```Also check: **Settings → Security → Trusted credentials → System** for the PortSwigger CA entry.

---

## 9. Flutter Application Interception Test
 
1. Install the target app on the emulator (`adb install your-app.apk`).
2. Launch the application and perform a network action: login, OTP request, search, refresh, open an API-backed screen, submit a form.
3. Check **Burp → Proxy → HTTP History**.
**Expected result:** the application's HTTP/HTTPS requests appear in HTTP History.
 
If traffic is still absent after steps 6–8, don't immediately assume SSL pinning — re-check the proxy address (§6), confirm the CA actually installed (§7.6), and confirm the iptables rules are still present (§8, especially after a Cold Boot). Only once all of that checks out should you move on to §11.
 
---
 
## 10. Static Indicators of Certificate Pinning (for code/binary review)
 
- `CertificatePinner` (OkHttp)
- `X509TrustManager` / `checkServerTrusted` custom implementations
- Custom `HostnameVerifier`
- Custom `SSLSocketFactory`
- Custom certificate validation logic
- Native/engine-level TLS implementation (BoringSSL calls inside `libflutter.so` or a custom native library)
Generic presence of OkHttp or standard Android framework TLS classes, by itself, is **not** proof of pinning — look for the custom logic specifically.
 
---
 
## 11. If It Still Doesn't Work: Runtime Bypass with Frida
 
If steps 6–9 are all correctly configured and the target app's traffic *still* never appears (or Burp's event log shows TLS handshake failures), the app likely performs its own certificate validation on top of — or instead of — the system trust store.
 
```
# On the emulator (rooted):
adb push frida-server-<version>-android-x86_64 /data/local/tmp/frida-server
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

---

## 12. Troubleshooting Matrix

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

## 13. Command Cheat Sheet

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

## 14. Complete Workflow — Quick View

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

## 15. Final Testing Principle

Build confidence in layers rather than assuming failure means pinning:

1. Confirm Burp listener connectivity (browser test).
2. Confirm Android/Genymotion proxying reaches Burp.
3. Confirm CA trust (system store install).
4. Confirm transparent routing (iptables) as a safety net for proxy-unaware clients.
5. Only then test the Flutter application itself, and only conclude pinning is present after Layer 3 (Frida) also fails to produce traffic **and** static analysis (§12) confirms custom validation logic in the binary.

Treat architecture mismatch (§2) and environment-level networking conflicts (§10) as the first two suspects whenever "nothing is working," before suspecting the application's security controls.

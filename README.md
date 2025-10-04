# Android Security & Reverse-Engineering — Pocket Reference

Commands, short explanations, and practical notes (emulator, adb, certs, JADX, Frida, Objection, apktool). Keep this page open during your lab sessions.

## 1) Start the emulator (writable system & permissive SELinux)

Useful when you need to modify /system (install system CA, push binaries, run frida-server without restrictions, etc.). Replace the AVD name with your AVD.

```
emulator -avd Pixel_3_XL_API_30 -writable-system -selinux permissive
```

**What it does:** launches Android Virtual Device named `Pixel_3_XL_API_30`, mounts system image writable and sets SELinux to permissive so you can modify system files easily.

## 2) Core ADB workflow (must-know commands)

| Command | What it does / notes |
| --- | --- |
| ```adb devices``` | List attached devices/emulators. Use to confirm the device is visible. |
| ```adb root``` | Restart adb daemon as root (works on emulator and some rooted devices). If it fails, device isn't allowing adb root. |
| ```adb remount``` | Remounts /system as read-write when possible. Commonly used after `adb root`. Equivalent to mounting system RW so you can push files to /system. |
| ```adb push local.file /remote/path``` | Copy a file from host to device. Example: push a system CA into `/system/etc/security/cacerts/`. |
| ```adb pull /remote/path local.file``` | Copy a file from device to host. |
| ```adb shell``` | Start an interactive shell on the device. Good for running `ls, ps, getprop`, etc. |
| ```adb shell <command>``` | Run a single command on device without an interactive shell, e.g. `adb shell ls /system`. |
| ```adb install app.apk``` | Install an APK to the device (non-root path). Use `-r` to replace, `-g` to grant permissions. |
| ```adb uninstall com.example.app``` | Uninstall an app by package name. |
| ```adb logcat``` | Show device logs. Great for debugging crashes, log output, Frida/Logger noise. Use filters: `adb logcat -s MyTag:D`. |
| ```adb tcpip 5555adb connect <device-ip>:5555``` | Restart ADB daemon listening on TCP (useful for networked devices). Be mindful of network security. |
| ```adb forward tcp:9222 localabstract:chrome_devtools_remote``` | Forward local host port to device socket (useful for chrome remote debugging and some app sockets). |
| ```adb shell pm list packagesadb shell pm path com.example.appadb shell pm dump com.example.app``` | Package manager queries: list packages, get APK path, dump package info. |
| ```adb shell am start -n com.example/.MainActivity``` | Start an activity (launch an app component) via Activity Manager. |
| ```adb bugreport > bugreport.zip``` | Collect a device bug report—very useful to inspect system state. Large and detailed. |
| ```adb shell getprop ro.build.version.releaseadb shell getprop ro.build.version.sdk``` | Query Android version and SDK level. |

## 3) Installing Burp's CA as a _system_ certificate (step-by-step)

Why system CA? Since Android 7 (Nougat) apps no longer trust user-added certs by default. For intercepting TLS of non-debuggable apps you often need the CA in `/system` (or use an instrumented app / network\_security\_config). This example assumes a writable emulator (see the emulator flags above) or a rooted device.

### Export Burp CA from Burp (PEM)

```
-- in Burp: Proxy > Options > Import / Export CA Certificate --
Export as PEM (Base64) - save as burpca.pem
```

### Convert PEM & compute Android cert filename (hash)

```
# convert PEM to DER (Android expects DER binary certificate)
openssl x509 -in burpca.pem -outform DER -out burpca.der

# compute the Android certificate filename hash (legacy hash used by Android's cert store)
openssl x509 -in burpca.pem -subject_hash_old -noout | head -n 1

# example output: 9a5ba575
# final filename used by Android is: 9a5ba575.0

# combine conversion and renaming in one line (replace $(hash) with the hash above):
openssl x509 -in burpca.pem -outform DER -out 9a5ba575.0
```

### Push cert into the device and set permissions

```
# ensure adb is root and system is writable
adb root
adb remount

# push the DER file into the cacerts directory
adb push 9a5ba575.0 /system/etc/security/cacerts/

# set ownership and permissions (required)
adb shell chown root:root /system/etc/security/cacerts/9a5ba575.0
adb shell chmod 644 /system/etc/security/cacerts/9a5ba575.0

# verify
adb shell ls -l /system/etc/security/cacerts | grep 9a5ba575.0
```

**Notes & gotchas:**

*   From Android 7+, apps by default ignore user certs (`/data`). System CAs are trusted already. Installing into `/system` forces the device to trust the CA at the system level.
*   Some devices use a different hash algorithm (`subject_hash` vs `subject_hash_old`). Use `openssl x509 -hash -in cert.pem` and compare if needed. For most Android builds `subject_hash_old` is correct. If a cert isn't recognized, try both.
*   After pushing, reboot the emulator: `adb reboot`. Sometimes services only reload certs on reboot.
*   Network security configs in app manifests can explicitly pin certs or disallow system CAs — in that case you must instrument/apply other techniques (Frida, hooking TLS libs, patching APK, etc.).

## 4) Quick cert-generation cheatsheet

```
# export from Burp: save as burpca.pem
openssl x509 -in burpca.pem -outform DER -out burpca.der
openssl x509 -in burpca.pem -subject_hash_old -noout | head -n1
# rename/convert to final filename (example hash 9a5ba575):
openssl x509 -in burpca.pem -outform DER -out 9a5ba575.0
```

## 5) JADX (decompilation)

JADX is the go-to decompiler for turning APK bytecode into readable Java-like source. Use `jadx-gui` for interactive exploration.

```
# decompile an APK into a directory
jadx -d out_dir app.apk

# open GUI (if available)
jadx-gui app.apk
```

Files of interest after decompilation: `AndroidManifest.xml`, `smali/` alternative (use apktool), and decompiled Java source under `out_dir`.

## 6) apktool (rebuild/edit resources & smali)

```
# decode APK (resources + smali)
apktool d app.apk -o app_decoded

# rebuild after edits
apktool b app_decoded -o rebuilt.apk

# sign the rebuilt.apk (required to install)
apksigner sign --ks mykeystore.jks rebuilt.apk
# or use older jarsigner (not recommended for modern builds)
```

Use apktool to modify manifest or resources, then recompile and sign. Useful for removing cert pinning or tweaking network config (only for your testing builds).

## 7) Frida (dynamic instrumentation)

Frida allows you to hook functions at runtime. On Android you'll need a matching frida-server binary running on the device (root recommended) or use Gadget/Spawn methods.

```
# on host (install frida-tools via pip):
pip install --user frida-tools

# download frida-server that matches device ABI + frida version, push and run
adb push frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
adb shell "/data/local/tmp/frida-server &"

# list processes (host)
frida-ps -U

# spawn or attach (example: spawn and instrument)
frida -U -f com.example.app -l script.js --no-pause

# trace calls (convenience)
frida-trace -U -i "open*" com.example.app
```

If device is rooted you can put frida-server in `/system/xbin` and run as root. Some apps detect Frida — you may need to hide the server or use advanced evasion.

## 8) Objection (runtime mobile exploration)

Objection is a runtime exploration toolkit built on Frida. It simplifies common tasks: bypass SSL pinning, explore file system, dump databases, dump keystores (if rooted), etc.

```
# install objection on host
pip install --user objection

# explore an app (gadget mode or spawn)
# if app is debuggable or instrumented with Frida Gadget:
objection --gadget com.example.app explore

# run a single command via objection
objection -g com.example.app --startup-command 'android sslpinning disable' explore

# common actions within objection shell
# filesystem                -> `ls`, `cd`, `pull`, `cat`
# ssl pinning bypass       -> `android sslpinning disable`
# dump sqlite databases    -> `android database list` / `android database pull `
```

Objection sometimes requires the app to be debuggable or the device to be rooted. It's a high-level convenience layer over Frida.

## 9) Useful Frida/Objection snippets & tactics

```
// Example Frida JS (hook SSLContext init - vary by app)
Java.perform(function () {
  var SSLContext = Java.use('javax.net.ssl.SSLContext');
  SSLContext.init.overload('[Ljavax.net.ssl.KeyManager;','[Ljavax.net.ssl.TrustManager;','java.security.SecureRandom').implementation = function(a,b,c) {
    console.log('SSLContext.init called — replacing with permissive TrustManager');
    return this.init.call(this, a, b, c);
  };
});
```

Hook native libraries (OpenSSL, BoringSSL) differently — inspect the APK for native .so files under `lib/` and use frida-elf-utils or frida-trace to find symbols.

## 10) Handy extras (short commands and tips)

*   `adb shell settings put global http_proxy <host>:<port>` — set global HTTP proxy for the device (not always respected by all apps).
*   `adb shell settings delete global http_proxy` — remove proxy.
*   `openssl s_client -connect host:443 -CAfile burpca.pem` — verify TLS handshake with Burp CA from host.
*   `keytool -printcert -file cert.der` — inspect certificate fields.
*   To discover app network endpoints: inspect decompiled code, look for `OkHttpClient`, `HttpURLConnection`, `Retrofit`.
*   Apps using Certificate Pinning may still reject system CA — you will need to patch the APK, use Frida hooks, or run a patched lib (or use API proxying on the host).

## 11) Short glossary

| Term | Meaning |
| --- | --- |
| System CA | Certificates installed in `/system/etc/security/cacerts`, trusted by the system trust store. |
| User CA | Certificates added by the user in Settings > Security > Trusted credentials (not trusted by some apps on Android 7+). |
| SELinux permissive | SELinux logs policy violations but doesn't enforce them — useful for labs. |
| Frida | Dynamic instrumentation toolkit for hooking native/Java methods at runtime. |
| Objection | Frida-powered runtime exploration tool focused on mobile security tasks. |
| JADX | Bytecode-to-Java decompiler. Great for analysis of logic, manifest, and strings. |

A pragmatic reminder: messing with system certs and instrumentation is powerful — and messy. Use emulators or explicitly consented devices. Keep notes and snapshots of your emulator images so you can revert changes quickly.

# Troubleshooting Universal Control
Universal Control sometime can be flaky. It is a fussy mix of Bluetooth LE and Wi-Fi/AWDL (data), so any tiny hiccup it will ghost. Here are someways that hopefully can make it behave

## Quick "make it work" sequence
1. Keep `Wi-Fi and Bluetooth ON` for both devices even it is on Ethernet
2. From `System Settings → Displays → Advanced` → toggle "`Allow your pointer and keyboard..."` OFF, wait for 5 seconds, back ON
3. Toggle `Wi-Fi OFF/ON` on both devices to reet the hidden awdl0 link
4. If you are on `VPN`, pause it and try again
5. Lock/unlock each device once. Then try pushing the cursor across the edge again

## Nerdier check
1. On the Mac
    - ifconfig adl0 → should show `status: active`. If not, toggleing Wi-Fi usually brings it back
    - Live logs: __log stream --info --prediate 'subsystem == "com.apple.unversalcontrol" || subsystem == "com.apple.sidecar"
2. Conflicts: Disable `internet sharing`, captive portals, and `Personal Hotspot` when using `Universal Control`
3 Bluetooth reset (only if BT is clearly flaky):
    - remove __/Library/Preferenes/com.apple.Bluetooth.plist__ (or __~/Library/Preferences/ByHost/com.apple.Bluetooth.*.plst), reboot, and re-pair devices
    - Layout sanity: `System settings → Displays` → ensure the displays are arranged so the edge you push through actually touches the next device

## Make the closed lid Mac stay awake

### Option A -- True clamshell mode
1. Plug in `power` + an `external display` + `external keyboard/mouse`
2. To keep it awake, add a tiny LaunchAgent to run __caffeinate__ whenever you are on power:

   ```bash
     mkdir -p ~/Library/LaunchAgents
     cat <<EOF > ~/Library/LaunchAgents/com.user.caffeinate.plist
     <?xml version="1.0" encoding="UTF-8"?>
     <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
     <plist version="1.0">
       <dict>
           <key>Label</key>
           <string>local.caffeinate.uc</string>
           <key>ProgramArguments</key>
           <array>
               <string>/usr/bin/caffeinate</string>
               <string>-i</string>
           </array>
           <key>RunAtLoad</key>
           <true/>
           <key>KeepAlive</key>
           <true/>
           <key>StandardErrorPath</key>
           <string>/tmp/caffeinate.err</string>
           <key>StandardOutPath</key>
           <string>/tmp/caffeinate.out</string>
       </dict>
      </plist>
   ```

   set permissions:
   ```bash
   chmod 644 ~/Library/LaunchAgents/local.caffeinate.plist
   chmod 700 ~/Library/LaunchAgents
   chown "$USER":staff ~/Library/LaunchAgents/local.caffeinate.plist
   ```

   validate the plist
    ```bash
    plutil -lint ~/Library/LaunchAgents/local.caffeinate.plist
    ```

    load it:
    ```bash
    launchctl bootstrap ~/Library/LaunchAgents/local.caffeinate.plist \
    || launchctl load -w ~/Library/LaunchAgents/local.caffeinate.plist
    ```

    verify it is running:
    ```bash
    launchctl print gui/"$UID"/local.caffeinate.uc 2>/dev/null || rg 'state|pid|path' \
    || launchctl print user/$UID/local.caffeinate.uc 2>/dev/null | rg 'state|pid|path'
    ```

    check log to see what happened:
    ```bash
    /usr/bin/log show --predicate 'subsystem == "com.apple.xpc.launchd" || process == launchctl ' --info --last 1h
    ```

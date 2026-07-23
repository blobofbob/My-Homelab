# 06 — Minecraft
 
This guide sets up a PaperMC server with Bedrock crossplay via GeyserMC and Floodgate. PaperMC is a high-performance fork of the vanilla server — it supports plugins, fixes mechanics inconsistencies, and runs better on constrained hardware (such as a Raspberry Pi). GeyserMC and Floodgate are plugins that translate Bedrock Edition's network protocol into Java Edition packets and allow Bedrock players to connect without having a Java account, letting players on mobile, console, and Windows Bedrock join the same server as Java Edition players.
 
This is a service I run occasionally because it consumes a lot of RAM and should probably be set up on more performant hardware, but since I have a Raspberry Pi 4, here we are.
 
**Prerequisites:** [11 — UFW](11-ufw.md) if you want to expose Minecraft publicly. Java must be installed before anything else.
 
---
 
## Installing Java
 
PaperMC requires Java 25. On Debian 13, it is available directly from the default repositories:
 
```bash
sudo apt install openjdk-25-jre-headless -y
```
 
Verify the installation:
 
```bash
java --version
```
 
Expected output starts with `openjdk 25`.
 
If it doesn't say openjdk version "25", you just need to tell Debian to switch its primary pointer by running:
 
```bash
sudo update-alternatives --config java
```
 
and selecting the version that corresponds to OpenJDK 25
---
 
## Setting up the server
 
Create a directory for the server:
 
```bash
mkdir -p ~/minecraft_server
cd ~/minecraft_server
```
 
To download the latest PaperMC JAR file, go to `https://papermc.io/downloads/paper`, copy the link address for the latest version of minecraft, and then download it with:
 
```bash
wget # for example: https://fill-data.papermc.io/v1/objects/0555a0b0468a5198d8fb1a16e1f9e95c81a917a2dc8f2e09867b4044742f6401/paper-26.1.2-72.jar 
```
 
Rename it for convenience:
 
```bash
mv paper-*.jar server.jar
```
 
---
 
## First run and EULA
 
Run the server once to generate its files:
 
```bash
java -jar paper.jar nogui
```
 
It will exit immediately and print a message about the EULA. Accept it:
 
```bash
echo "eula=true" > eula.txt
```
 
> **What's the EULA?:** This is Mojang's End User Licence Agreement for running a Minecraft server. Accepting it is required to proceed. Read it at `https://aka.ms/MinecraftEULA`.
 
---
 
## Starting the server
 
For a shared system with other services running, a reasonable heap allocation is:
 
```bash
java -Xms1G -Xmx4G -XX:+UseZGC -XX:ConcGCThreads=1 -jar server.jar nogui
```
 
- `-Xms1G` sets the initial heap size. Starting low lets Java grow into the allocation rather than reserving 4 GB immediately, leaving memory free for your other homelab services when the Minecraft server is quiet or empty.
- `-Xmx4G` caps the heap at 4 GB, leaving headroom on the Raspberry Pi for other services, and the base operating system.
- `-XX:+UseZGC` enables Generational ZGC, the current recommended garbage collector for Minecraft servers — it minimizes tick-loop pause times compared to G1GC and actively uncommits/returns unused memory back to Debian.
- `-XX:ConcGCThreads=1` limits concurrent garbage collection to a single background CPU thread. This prevents the JVM from hijacking all 4 cores of your Raspberry Pi 4 during memory sweeps, protecting the performance of your network's DNS and web traffic.
- `-jar server.jar` specifies the jar file, replace accordingly.
- `nogui` disables the gui to save ram.
> For optimised JVM flags specific to your setup, PaperMC provides a startup script generator at `https://docs.papermc.io/paper/getting-started` — it produces a complete launch command tuned for current Paper versions.
 
The server is ready when you see:
 
```
Done (Xs)! For help, type "help"
```
 
---
 
## Running in the background with tmux
 
The server process needs to stay alive after you disconnect from SSH. tmux is the right tool for this — it keeps the process running in a detachable session and lets you re-attach to the console at any time.
 
Install tmux if it is not already present:
 
```bash
sudo apt install tmux -y
```
 
Start a named session and launch the server inside it:
 
```bash
tmux new-session -d -s minecraft -x 220 -y 50 \
  "cd ~/minecraft_server && java -Xms1G -Xmx4G -XX:+UseZGC -XX:ConcGCThreads=1 -jar server.jar nogui"
```
 
To attach to the server console:
 
```bash
tmux attach-session -t minecraft
```
 
To detach without stopping the server: `Ctrl+B`, then `D`.
 
---
 
## GeyserMC — Bedrock crossplay
 
GeyserMC allows Bedrock Edition players to join the Java server. Floodgate is a companion plugin that removes the requirement for Bedrock players to have a Java Edition account — they only need a Microsoft account.
 
**1 — Move to the plugins folder.**
 
```bash
cd /minecraft_server/plugins
```
 
**2 — Download the plugins.**
 
From `https://geysermc.org/download`, download the **Geyser-Spigot** and **Floodgate-Spigot** jar files by copying their link addresses and using wget:
 
```bash
wget # for example https://download.geysermc.org/v2/projects/geyser/versions/latest/builds/latest/downloads/spigot
wget # for example https://download.geysermc.org/v2/projects/floodgate/versions/latest/builds/latest/downloads/spigot
```
 
**3 — Rename the plugins.** The files you will get will be named `spigot` (yes, both of them), so it is important to rename them immediately after you download them with:
 
```bash
mv spigot geyser.jar # for the geyser plugin
mv spigot floodgate.jar # for the floodgate plugin
```
 
**4 — Disable chat signing in `server.properties`.**
 
GeyserMC requires this for Bedrock players to send chat messages:
 
```bash
nano ~/minecraft_server/server.properties
```
 
Find and set:
 
```
enforce-secure-profile=false
```
 
**5 — Restart the server.**
 
```bash
# In the tmux session
stop
```
 
Then start it again. Both plugins will load and generate their config files in `~/minecraft_server/plugins/Geyser-Spigot/` and `~/minecraft_server/plugins/floodgate/`.
 
**6 — Verify GeyserMC loaded.**
 
Look for this in the startup output:
 
```
[Geyser-Spigot] Started Geyser on 0.0.0.0:19132
```
 
Bedrock players can now connect using the server's public IP and port 19132. Java players connect on port 25565 as normal. [19] [20]
 
---
 
## UFW rules
 
If UFW is active and you want the server reachable from the internet, add the Minecraft ports:
 
```bash
sudo ufw allow 25565/tcp   # Java Edition
sudo ufw allow 19132/udp   # Bedrock Edition (GeyserMC)
sudo ufw allow 19133/udp   # Bedrock Edition (secondary)
```
 
These are listed as optional in the [Reference](../reference.md) port map.
 
---
 
## Keeping the server updated
 
PaperMC releases new builds frequently — for bug fixes, performance improvements, and Minecraft version updates. [16] [17]
 
To update:
 
```bash
# Stop the server first
cd ~/minecraft_server
tmux attach-session -t minecraft
stop 
# in the minecraft_server directory
mv server.jar server.jar.bak
# Download the latest build from papermc.io/downloads with wget and rename it to server.jar
java -jar server.jar nogui   # verify it starts cleanly, then delete the backup
rm server.jar.bak
```
 
For Geyser and Floodgate, check `https://geysermc.org/download` for updates after each Minecraft version bump — the plugins updates separately from PaperMC to match new Bedrock versions.
 
---
 
## Useful commands
 
```bash
# Attach to the running server console
tmux attach-session -t minecraft
 
# List active tmux sessions
tmux list-sessions
 
# Stop the server cleanly (from inside the tmux session)
stop
 
# Start the server
tmux new-session -d -s minecraft -x 220 -y 50 \
  "cd ~/minecraft_server && java -Xms1G -Xmx4G -XX:+UseZGC -XX:ConcGCThreads=1 -jar server.jar nogui"
 
# Check Java version
java --version
```
 
---
 
## Troubleshooting
 
**Server exits immediately after first start**
 
You have not accepted the EULA. Check `eula.txt` — it should contain `eula=true`.
 
---
 
**Bedrock players cannot connect**
 
Confirm GeyserMC loaded and is listening on port 19132:
 
```bash
sudo ss -ulnp | grep 19132
```
 
If nothing is listening, the plugin did not load. Check the startup logs for errors from GeyserMC:
 
```bash
grep -i geyser ~/minecraft_server/logs/latest.log
```
 
Also confirm port 19132 UDP is open on your router and in UFW.
 
---
 
**Bedrock players can join but cannot chat**
 
`enforce-secure-profile=false` is not set in `server.properties`. Set it and restart the server.
 
---
 
**Out of memory errors**
 
The `-Xmx` value may be too high for the available RAM with other services running. Check actual memory usage with `free -h` while the server is running, and reduce `-Xmx` if needed.
 
---
 
**Next:** [07 — Flipper Zero](07-flipper-zero.md)
 
**Sources:** [13] [14] [15] [16] [17] [18] [19] [20]
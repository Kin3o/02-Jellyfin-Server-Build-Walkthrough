# Jellyfin Server Build Walkthrough

This guide recreates the verified local Jellyfin deployment. It uses direct instructions and explains why each major choice matters. Values shown here belong to this lab; substitute addresses and resource allocations appropriate for your own environment.

For the mistakes encountered during the build, see [03-Jellyfin-Troubleshooting-Lab](https://github.com/Kin3o/03-Jellyfin-Troubleshooting-Lab). Planned NAS storage and remote access are documented separately in [04-Jellyfin-Future-Plans](https://github.com/Kin3o/04-Jellyfin-Future-Plans).

## Phase 1 - Confirm the architecture and prerequisites

### 1. Review the hardware

The verified host configuration was:

| Setting | Value |
|---|---|
| Computer | Dell OptiPlex 5070 |
| CPU | Intel Core i5; exact model not verified |
| RAM | 16 GB |
| Proxmox drive | 128 GB SATA |
| Proxmox version | 9.2.2 |

Do not enable Intel Quick Sync based only on the `Core i5` label. The exact processor and integrated GPU must be identified first.

### 2. Use the final network design

The Proxmox host was initially located at `192.168.3.34` behind an OPNsense lab interface. That routed path remained unverified. The successful configuration moved Proxmox directly onto the GL.iNet LAN.

| Device or service | Final value |
|---|---|
| GL.iNet gateway | `192.168.8.1` |
| Proxmox host | `192.168.8.3/24` |
| Proxmox GUI | `https://192.168.8.3:8006` |
| Jellyfin VM | `192.168.8.4/24` |
| Jellyfin GUI | `http://192.168.8.4:8096` |
| Proxmox bridge | `vmbr0` |

> **Warning:** Changing the Proxmox management address disconnects the active web session. Make network changes from a local console or another recovery-capable connection.

In Proxmox, select the node, open **System > Network**, and confirm the management address is assigned to `vmbr0`, not directly to the physical bridge port. The final bridge uses `192.168.8.3/24` with gateway `192.168.8.1`.

The [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.pdf) describes Linux bridges, virtual machines, automatic startup, and QEMU Guest Agent integration.

## Phase 2 - Download the Debian installer

### 1. Open ISO storage

1. Sign in to `https://192.168.8.3:8006`.
2. Select the Proxmox node.
3. Select **local** storage.
4. Select **ISO Images**.
5. Select **Download from URL**.

### 2. Download Debian 13.7 amd64 netinst

Use the Debian 13.7 amd64 netinst image. Debian publishes the current installer and its integrity files on the [official Debian installer page](https://www.debian.org/releases/stable/debian-installer/).

The build used the amd64 netinst ISO because it provides a small installer that downloads the selected packages during setup.

Select **Download**, wait for the task to finish, and confirm the ISO appears under **ISO Images**.

## Phase 3 - Create the Proxmox VM

### 1. Start the VM wizard

Select **Create VM** and enter the following verified values.

| Wizard page | Setting | Value |
|---|---|---|
| General | VM ID | `100` |
| General | Name | `jellyfin` |
| OS | ISO | Debian 13.7 amd64 netinst |
| OS | Guest type | Linux |
| System | SCSI controller | VirtIO SCSI single |
| Disk | Storage | `local-lvm` |
| Disk | Size | `32 GB` |
| CPU | Sockets | `1` |
| CPU | Cores | `4` |
| Memory | Initial memory | `4096 MiB` |
| Network | Bridge | `vmbr0` |
| Network | Model | VirtIO |

Leave unrecorded firmware and machine-type values at the Proxmox defaults rather than claiming a setting that was not preserved.

The 32 GB disk holds Debian, Jellyfin, its database, cache, and temporary test media. It is not large enough for the planned permanent collection.

### 2. Finish and start the VM

1. Review the summary.
2. Select **Finish**.
3. Select VM `100`.
4. Select **Start**.
5. Open **Console**.

## Phase 4 - Install Debian

The official [Debian 13 amd64 installation guide](https://www.debian.org/releases/stable/amd64/) explains the complete installer. The selections below reproduce this lab.

### 1. Begin installation

1. Select **Graphical Install**.
2. Select the desired language, location, and keyboard.
3. Enter `jellyfin` for the hostname.
4. Enter `home.arpa` for the domain.
5. Configure the root and normal-user credentials requested by the installer.

Use a strong password. Do not publish it in documentation or screenshots.

### 2. Partition the virtual disk

> **Warning:** Partitioning erases the selected target. Confirm the installer is operating on the VM's 32 GB virtual disk.

1. Select **Guided - use entire disk**.
2. Select the 32 GB virtual disk.
3. Select **All files in one partition**.
4. Select **Finish partitioning and write changes to disk**.
5. Confirm the write operation.

An all-in-one partition keeps a first server installation simple. The permanent media library will later live on a NAS rather than this filesystem.

### 3. Select server software

At the text-mode software-selection prompt, an asterisk means an item is selected. Enter:

```text
11 12
```

This selects:

- `11`: SSH Server
- `12`: standard system utilities

It also leaves the Debian desktop environment and GNOME deselected. A graphical desktop would consume RAM, disk space, and background resources without helping a headless Jellyfin server.

### 4. Install GRUB and reboot

1. Select **Yes** when asked to install GRUB.
2. Select the VM's primary virtual disk.
3. Allow Debian to finish.
4. Reboot the VM.

If the installer starts again, stop the VM, open **Hardware > CD/DVD Drive**, select **Do not use any media**, and start the VM.

## Phase 5 - Configure Debian after installation

### 1. Update Debian and install management packages

Sign in at the VM console as root. Run:

```bash
apt update
apt full-upgrade -y
apt install -y sudo curl qemu-guest-agent openssh-server
```

- `apt update` refreshes the package index.
- `apt full-upgrade` applies available updates and handles dependency changes.
- `sudo` allows an approved normal account to run administrative commands.
- `curl` downloads files over HTTPS.
- `qemu-guest-agent` improves communication between Proxmox and the VM.
- `openssh-server` provides remote terminal access.

While logged in as root, do not prefix commands with `sudo`. Root already has administrative privileges.

### 2. Enable the services

```bash
systemctl enable --now qemu-guest-agent
systemctl enable --now ssh
systemctl status qemu-guest-agent --no-pager
systemctl status ssh --no-pager
```

`enable --now` starts each service immediately and configures it to start at boot. Confirm both status commands report `active (running)`.

In Proxmox, open VM `100` and confirm **Options > QEMU Guest Agent** is enabled. Also set **Start at boot** to **Yes**.

### 3. Verify the network

This build ended with the static address `192.168.8.4/24` and gateway `192.168.8.1`. The exact Debian configuration file or command used to assign that static address was not preserved, so this guide does not invent it.

Confirm the live result:

```bash
ip -br address
ip route
```

Expected state:

- The VM interface shows `192.168.8.4/24`.
- The default route points to `192.168.8.1`.

Use the Debian networking method appropriate to your installation if those values are absent. Keep console access available while changing networking.

### 4. Connect from Windows with SSH

Open Windows PowerShell and run:

```powershell
ssh YOUR_USERNAME@192.168.8.4
```

Type `yes` the first time Windows asks whether to trust the host key. Enter the normal Debian user's password; the password is intentionally not displayed while typing.

To enter a root shell after connecting, run:

```bash
su -
```

Enter the root password. Do not enable direct password-based root SSH access merely for convenience.

## Phase 6 - Install Jellyfin

Jellyfin's [official Debian and Ubuntu instructions](https://jellyfin.org/docs/general/installation/linux/) provide a script and a separate checksum file. Verifying the checksum confirms the downloaded script matches the file published by Jellyfin.

### 1. Download and verify the installer

From a root shell, run:

```bash
cd /tmp
curl -fsSLO https://repo.jellyfin.org/install-debuntu.sh
curl -fsSLO https://repo.jellyfin.org/install-debuntu.sh.sha256sum
sha256sum -c install-debuntu.sh.sha256sum
```

Do not continue unless the output includes:

```text
install-debuntu.sh: OK
```

Optionally inspect the script before running it:

```bash
less install-debuntu.sh
```

Press `q` to leave `less`.

### 2. Resolve the `/tmp` capacity check

The first run found slightly less than the required 2 GB free in `/tmp`. The VM disk still had free space; the failure involved the memory-backed temporary filesystem.

> **Warning:** Shut down the VM before changing its memory allocation.

1. Run `poweroff` inside Debian.
2. In Proxmox, select VM `100`.
3. Open **Hardware > Memory > Edit**.
4. Change memory from `4096 MiB` to `8192 MiB`.
5. Start the VM.
6. Sign in and run `df -h /tmp`.

Because `/tmp` is temporary, the downloaded script may disappear after shutdown. If necessary, repeat the download and checksum commands.

### 3. Run the installer

```bash
cd /tmp
bash install-debuntu.sh
```

Review the detected Debian release and architecture, then press Enter when prompted.

### 4. Enable and verify Jellyfin

```bash
systemctl enable --now jellyfin
systemctl status jellyfin --no-pager
ss -lntp | grep ':8096'
```

Confirm Jellyfin reports `active (running)` and a process is listening on TCP `8096`.

Open the web interface from a computer on the same GL.iNet LAN:

```text
http://192.168.8.4:8096
```

Complete the first-run wizard and create a Jellyfin administrator account. Disable automatic router port mapping. Do not create a public TCP `8096` port forward.

## Phase 7 - Create the media directories

Create separate roots for movies, shows, and music:

```bash
mkdir -p /srv/media/movies
mkdir -p /srv/media/tv
mkdir -p /srv/media/music
```

`/srv` is intended for data served by the system. Do not create the media library under `/tmp`; temporary data may be deleted during reboot.

> **Warning:** Recursive ownership commands affect everything below the target. Confirm the path is exactly `/srv/media` before pressing Enter.

The build used the following permission pattern:

```bash
chown -R root:jellyfin /srv/media
chmod -R 2775 /srv/media
```

Mode `2775` provides read, write, and directory traversal to the owner and group. The leading `2` sets the set-group-ID bit so newly created content inherits the directory's group.

Verify the directories:

```bash
ls -ld /srv/media /srv/media/movies /srv/media/tv /srv/media/music
```

The directory ownership was configured during the build but was not independently audited afterward. Treat the commands above as the intended configuration, not proof of a later permission review.

## Phase 8 - Configure Jellyfin libraries

Open **Dashboard > Libraries > Add Media Library** and create one library at a time.

### 1. Movies

| Setting | Value |
|---|---|
| Content type | Movies |
| Display name | Movies |
| Folder | `/srv/media/movies` |
| Enable library | Enabled |
| Metadata language | English |
| Country/Region | United States |
| Prefer embedded titles | Disabled |
| Embedded subtitles | Allow All |
| Real-time monitoring | Enabled |
| Metadata providers | TheMovieDb first, The Open Movie Database second |
| Automatic metadata refresh | Never |
| NFO saver | Disabled |
| Image fetchers | TheMovieDb, The Open Movie Database, Embedded Image Extractor |
| Screen Grabber | Disabled |
| Similar-item provider | Local Genre/Tag enabled |
| Save artwork into media folders | Disabled |
| Trickplay and chapter images | Disabled |

`Never` disables periodic metadata replacement; it does not prevent the initial scan from fetching metadata.

### 2. TV Shows

| Setting | Value |
|---|---|
| Content type | Shows |
| Display name | TV Shows |
| Folder | `/srv/media/tv` |
| Real-time monitoring | Enabled |
| Metadata providers | Retain the available defaults |
| NFO saver | Disabled |
| Save artwork into media folders | Disabled |
| Trickplay and chapter images | Disabled |

### 3. Music

| Setting | Value |
|---|---|
| Content type | Music |
| Display name | Music |
| Folder | `/srv/media/music` |
| Prefer embedded titles | Enabled |
| Real-time monitoring | Enabled |
| LUFS scan | Enabled |
| Artist metadata | MusicBrainz first, TheAudioDB second |
| Album metadata | MusicBrainz first, TheAudioDB second |
| Embedded Image Extractor | Enabled |
| Screen Grabber | Disabled |
| ListenBrainz | Disabled |
| Local Genre/Tag | Enabled |
| Save artwork into media folders | Disabled |
| Save lyrics into media folders | Enabled |
| Prefer `ARTISTS` tag | Disabled |
| Custom tag delimiter | Disabled |

LUFS scanning measures perceived loudness so clients can normalize volume, but it makes the initial scan longer. Screen Grabber, trickplay, and chapter images were left disabled to avoid unnecessary processing and storage use on this small VM. Jellyfin warns that chapter-image extraction can be resource intensive in its [chapter-image documentation](https://jellyfin.org/docs/general/server/metadata/chapter-images/).

## Phase 9 - Download and test *Big Buck Bunny*

Blender publishes *Big Buck Bunny* as an openly licensed film. Its [official download page](https://peach.blender.org/download/) links to Blender's movie files.

### 1. Create the movie folder

```bash
mkdir -p "/srv/media/movies/Big Buck Bunny (2008)"
```

### 2. Download the test movie

```bash
curl -L "https://download.blender.org/peach/bigbuckbunny_movies/big_buck_bunny_480p_h264.mov" \
  -o "/srv/media/movies/Big Buck Bunny (2008)/Big Buck Bunny (2008).mov"
```

Apply the intended directory ownership and permissions:

```bash
chown -R root:jellyfin "/srv/media/movies/Big Buck Bunny (2008)"
chmod -R 2775 "/srv/media/movies/Big Buck Bunny (2008)"
```

### 3. Scan and play

1. Open **Dashboard > Libraries**.
2. Select **Scan All Libraries**.
3. Wait for the scan to complete.
4. Select the back arrow to leave the administrator page.
5. Open **Movies** from the normal Jellyfin home screen.
6. Select *Big Buck Bunny*.
7. Select **Play**.

The test was successful: Jellyfin detected the movie, downloaded artwork and metadata, and played it.

### 4. Organize future media correctly

Jellyfin's official guides describe supported organization for [movies](https://jellyfin.org/docs/general/server/media/movies/), [TV shows](https://jellyfin.org/docs/general/server/media/shows/), and [music](https://jellyfin.org/docs/general/server/media/music/).

```text
/srv/media/movies/
└── Movie Name (Year)/
    └── Movie Name (Year).mkv
```

```text
/srv/media/tv/
└── Series Name (Year)/
    └── Season 01/
        └── Series Name - S01E01 - Episode Name.mkv
```

```text
/srv/media/music/
└── Artist Name/
    └── Album Name/
        └── 01 - Track Name.flac
```

## Phase 10 - Verify the completed local deployment

Run these checks inside the VM:

```bash
ip -br address
ip route
systemctl is-enabled jellyfin
systemctl is-active jellyfin
systemctl is-active qemu-guest-agent
systemctl is-active ssh
df -h /
df -h /tmp
```

Confirm:

- The VM uses `192.168.8.4/24`.
- The default route uses `192.168.8.1`.
- Jellyfin, SSH, and QEMU Guest Agent are active.
- Jellyfin starts automatically.
- `http://192.168.8.4:8096` opens from the GL.iNet LAN.
- *Big Buck Bunny* plays successfully.
- Hardware acceleration remains `None`.
- Automatic router port mapping remains disabled.
- No public TCP `8096` port forward exists.
- The 32 GB system disk is treated only as temporary test media storage.

The verified local build is complete. Continue with the planned storage and resilience work in [04-Jellyfin-Future-Plans](https://github.com/Kin3o/04-Jellyfin-Future-Plans).

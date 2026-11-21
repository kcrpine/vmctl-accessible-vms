# vmctl: accessible virtual machines on a Linux server

vmctl is one Bash script that builds and manages virtual machines (KVM, QEMU and libvirt) on a Linux server. It was written to be used with a screen reader: every question is a plain numbered menu or a yes/no prompt, every step prints a sentence about what it is doing, and nothing needs a graphical installer.

What it does:

- **Windows 11, Windows 10, Windows 11 LTSC, Windows 10 LTSC and Tiny11, fully unattended.** It installs VirtIO drivers, sound, the QEMU guest agent and Remote Desktop. **NVDA is installed and starts at the sign-in screen**, so you hear Windows from the first boot.
- **Linux.** You can install from an installer ISO, such as Debian, Ubuntu, Fedora or Mint, and talk to the installer through remote-viewer (SPICE, with sound). Or you can use a Debian/Ubuntu cloud image, which cloud-init sets up for you.
- **Reads the screen for you.** While a VM boots or installs, vmctl reads its screen text with OCR and announces the steps: Windows Setup stages, disk checks, the sign-in screen.
- **Remote access ports.** Each VM gets its own port for remote-viewer (SPICE) and Remote Desktop or SSH. You can forward more ports (`vmctl forward`) and limit who may connect (`--allow`).
- **Snapshots.** You can take snapshots by hand and roll back, and there are daily automatic snapshots with a limit on how many are kept.
- **Shared folders.** A VM can use a folder on the server: a drive in Windows, `/mnt/vmshare` in Linux. That makes it easy to get installers, documents or music into a VM.
- **VM owners.** You can let ordinary server users manage snapshots and shared folders of *their own* VMs through `sudo`, without giving them root.
- **Installs what it needs.** On first use it installs its dependencies with apt, dnf, zypper or pacman.

## Requirements

- An x86_64 Linux server with hardware virtualization (Intel VT-x or AMD-V) enabled. `vmctl check` tells you if it is missing.
- Root access (vmctl runs as root, or through `sudo`).
- Enough disk space. A Windows ISO is about 6 to 9 GB, and VM disks grow as they fill. Disks live in `/var/lib/libvirt/images/vmctl` (change `STORE` in the settings).
- vmctl is developed and tested on **Debian 13**. Ubuntu should work the same way. Fedora, openSUSE and Arch are supported on a best-effort basis: the package names are there, but they are less tested.
- On your own computer, to see or hear a VM: [remote-viewer (virt-viewer)](https://virt-manager.org/download.html). For Windows VMs, a Remote Desktop client also works.

## Installing

Download the script, put it where root can run it, and let it check the server:

```sh
sudo curl -fLo /usr/local/sbin/vmctl https://raw.githubusercontent.com/kcrpine/vmctl-accessible-vms/main/vmctl
sudo chmod 755 /usr/local/sbin/vmctl
sudo vmctl check
```

`vmctl check` installs anything missing: QEMU, libvirt, virt-install, UEFI firmware, a TPM emulator, the OCR tools and so on. It loads the kernel modules and starts libvirt. It explains each step and is safe to run again at any time.

Or clone the repository:

```sh
git clone https://github.com/kcrpine/vmctl-accessible-vms.git
sudo install -m 755 vmctl-accessible-vms/vmctl /usr/local/sbin/vmctl
```

## Your first VM

Run this and answer the questions:

```sh
sudo vmctl create
```

vmctl asks for the name, the operating system, memory, processors, disk size, the user account and password, and who may connect. Every answer can also be given as an option so nothing is asked, for example:

```sh
sudo vmctl create --name office --os win11ltsc --ram 8G --cpus 4 --disk 100G \
    --user alex --allow 203.0.113.7 --yes
```

`sudo vmctl create --help` lists every option.

When a Windows install starts, vmctl offers to follow it (`vmctl progress NAME`). It announces each stage until Windows is ready and NVDA is talking. A full Windows install takes about 30 to 50 minutes and needs no help from you.

To connect:

- `vmctl info NAME` prints the remote-viewer address, the password, and the Remote Desktop or SSH port.
- `vmctl viewer NAME` writes a `.vv` file. Copy it to your computer and open it with remote-viewer.
- **Remote Desktop to Windows:** sign in as `.\USERNAME` (with the `.\` in front). Otherwise Windows may treat the name as a Microsoft account and refuse the password.

### Windows ISOs

- **Windows 11 LTSC and Windows 10 LTSC** (90-day evaluation) download automatically from Microsoft.
- **Windows 11 and Windows 10 (consumer editions):** vmctl asks Microsoft for a download link. Microsoft often refuses these requests from data-centre IP addresses. In that case, get a link on your own computer from Microsoft's download page and pass it with `--iso-url`, or upload the ISO and use `--iso-path`.
- **Tiny11:** Microsoft doesn't host it, so give it with `--iso-url` or `--iso-path`. Before the VM is made, vmctl looks inside the ISO's install image for Microsoft Edge (`msedge.exe`). This only reads the image's file list and takes about a second.
  - **Edge is still there:** Windows uses Edge and downloads NVDA itself, the same way as the other Windows VMs.
  - **Edge was removed** (tiny11builder removes it): the server downloads the newest Firefox (64-bit MSI, checked against Mozilla's SHA512SUMS) and NVDA (checked against NV Access's checksum) into the VM's shared folder. Windows copies Firefox over and installs it first, then installs NVDA. Each installer is deleted from the folder once it is installed.

  Either way the shared folder stays, so you can drop in other installers later. `vmctl info NAME` shows which browser the VM got.

## Everyday commands

| Command | What it does |
|---|---|
| `vmctl list` | All VMs with their state and ports |
| `vmctl info NAME` | Connection details for one VM |
| `vmctl start NAME` | Start a VM and read out its boot until it is ready |
| `vmctl stop NAME` | Clean shutdown |
| `vmctl reboot NAME` | Restart and read out the boot |
| `vmctl screen NAME` | Read out the text on the VM's screen right now |
| `vmctl progress NAME` | Follow a Windows install step by step |
| `vmctl password NAME [USER]` | Reset a password inside the VM |
| `vmctl forward add NAME HOSTPORT GUESTPORT` | Open another port to the VM |
| `vmctl disk add NAME SIZE` | Add a data disk (formatted and mounted for you) |
| `vmctl edit NAME` | Change RAM, CPU cores or disk sizes (menu, or `--ram`, `--cpus`, `--disk`, `--data-disk`) |
| `vmctl disk grow NAME ID SIZE` | Make one disk bigger (`system`, `data1`...) and use the space inside the VM |
| `vmctl disk expand NAME` | Grow the partitions inside the VM to fill its disks |
| `vmctl autostart NAME on\|off` | Start the VM whenever the server starts |
| `vmctl snapshot ...` | Take, list, restore and delete snapshots; daily settings |
| `vmctl share ...` | Give the VM a folder from the server |
| `vmctl owner NAME USER` | Let a server user manage this VM's snapshots and shared folder |
| `vmctl delete NAME` | Delete a VM, its disks, ports and firewall rules |
| `vmctl help` | Everything else |

## Changing a VM (RAM, CPUs, disk size)

Run `vmctl edit NAME` for a numbered menu, or give the changes directly:

```
vmctl edit win11 --ram 16G --cpus 6 --restart
vmctl edit win11 --disk 200G --data-disk data1=500G
```

- **RAM and CPU cores** go into the VM's saved settings. If the VM is running, vmctl asks whether to
  restart it now (`--restart` / `--no-restart` answer up front); it shuts down cleanly first. A restart
  from inside the VM is not enough, it needs a full shutdown and start. Cores are given as one socket, so
  Windows Home and Pro use all of them.
- **Disks can only grow.** Disks are thin: a 2000 GB disk takes only what the VM has written, and growing
  it takes no extra space on the server. vmctl grows the disk (while running, if the VM is on), then uses
  the guest agent to grow the partition and filesystem inside the VM: drive C: and the Data drive in
  Windows, `/` (plain partitions or LVM; ext4, XFS, Btrfs) and `/data` in Linux. If the VM is off, it
  offers to start it, or run `vmctl disk expand NAME` later.
- vmctl changes nothing inside a Linux VM when the root partition is not the last one on its disk (for
  example swap right after it) or is encrypted. It tells you, and the space waits there for you to use.
- Restoring a snapshot taken before a disk was grown keeps the new disk size; run `vmctl disk expand NAME`
  afterwards so the VM uses it again.

## Shared folders

```sh
sudo vmctl share add NAME              # a new folder ~/vmshare/NAME, just for this VM
sudo vmctl share add NAME ~/Installers # or a folder you already have
sudo vmctl share show NAME
sudo vmctl share remove NAME           # disconnects it; your files stay
sudo vmctl share                       # or use the menu
```

In Windows the folder is a drive (usually Z:). In Linux it is mounted at `/mnt/vmshare`, and the VM's user also gets a `~/vmshare` shortcut. The VM has to be switched off and on once to connect the folder. vmctl offers to do that, then sets the folder up inside the VM through the guest agent. For Windows that means installing WinFsp and the VirtIO-FS service. For Linux it means adding an `/etc/fstab` entry.

Linux VMs installed from an ISO need the `qemu-guest-agent` package for this. Without it, vmctl prints the two commands to run inside the VM.

**Security:** the file sharing for an owned VM runs as the VM's owner, not as root. The VM can only reach what that server user can reach. Everything it writes, even as root inside the VM, belongs to that user. Owners can only share folders inside their own home folder.

## VM owners (letting other users manage their own VMs)

```sh
sudo vmctl owner office alex
```

From then on, the server user `alex` can run `sudo vmctl snapshot` and `sudo vmctl share` for `office`, and for no other VM. vmctl installs a sudoers rule (`/etc/sudoers.d/vmctl-snapshot`) that allows exactly those two commands for every user, and vmctl itself checks who owns which VM. Members of the `sudo`, `wheel` or `admin` groups count as administrators and can manage every VM.

## Snapshots

```sh
sudo vmctl snapshot take NAME [SNAP-NAME] [--note TEXT] [--with-data]
sudo vmctl snapshot list NAME
sudo vmctl snapshot restore NAME SNAP-NAME
sudo vmctl snapshot daily NAME on --keep 10
```

A running VM shuts down cleanly for about a minute while a snapshot is taken, then starts again. Daily snapshots run at 04:00 by default (`vmctl snapshot time HH:MM`). Only the automatic ones are cleaned up; snapshots you take yourself stay until you delete them.

## Settings

Settings are in `/etc/vmctl/vmctl.conf`, written the first time vmctl runs. To change one, remove the `#` in front of the line. The most useful ones:

- `PUBLIC_HOST`: the host name that connection files point to (default: the server's main IP address).
- `STORE`: where disks and ISOs are kept.
- `RDP_BASE`, `SSH_BASE`, `SPICE_BASE`: the first port handed out for each kind of access (13389, 12201, 5910).
- `NET_PREFIX`: the private network for the VMs (default 192.168.150).
- `WIN_LOCALE`, `WIN_TIMEZONE`: Windows language and time zone.

## Firewall and networking

VMs live on a private libvirt network (`vmctl`, bridge `virbr-vmctl`). vmctl reaches them from outside with its own nftables table (`table ip vmctl`), which does the port forwarding, and it rebuilds that table at boot (`vmctl-forwards.service`). If ufw is active, vmctl also adds the matching `ufw route allow` rules. All of its rules carry a comment starting with `vmctl:`, and they are removed again when a VM is deleted. Other firewall rules are left alone.

A host port that is already in use (by another VM or by any service listening on the server) is refused, so other services are not disturbed. The remote-viewer ports start at 5910, not 5900, to keep clear of other software that uses 5900.

## Where things are

| Path | Contents |
|---|---|
| `/usr/local/sbin/vmctl` | The script |
| `/etc/vmctl/` | Settings, one record per VM, remote-viewer files |
| `/var/lib/libvirt/images/vmctl/` | Disks, ISOs, install discs, logs, snapshots |
| `/var/log/vmctl-install.log` | Package installs and other long-running steps |
| `/usr/sbin/vmctl-virtiofsd` | Small helper that runs file sharing as the VM's owner (written by vmctl) |

## Uninstalling

Delete each VM with `sudo vmctl delete NAME`. When you delete the last one, vmctl offers to remove the downloaded ISOs and its network and firewall rules as well (`--purge`). Then remove the script, `/etc/vmctl`, `/etc/sudoers.d/vmctl-snapshot`, `/usr/sbin/vmctl-virtiofsd` and the `vmctl-forwards` and `vmctl-snapshots` systemd units.

## Feedback

Issues and pull requests are welcome, especially reports from other distributions and from screen reader users about anything that is hard to follow.

## License

MIT. See [LICENSE](LICENSE).

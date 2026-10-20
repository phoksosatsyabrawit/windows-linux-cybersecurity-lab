# Windows + Linux Cybersecurity Lab

A small home lab built with **VMware Workstation**, a Windows host, and a Kali Linux virtual machine. The goal is to learn networking fundamentals and safely practise Linux administration and remote access in a controlled environment.

> **Current status (verified 2026-10-20):** Windows and Kali communicate successfully over the VMware Host-only network; SSH from Windows to Kali works; Kali also has working NAT connectivity and DNS resolution.

## Goals

- Understand NAT and Host-only virtual networking.
- Configure two network adapters on a Kali Linux VM.
- Verify IP addresses, subnets, routes, and connectivity.
- Connect to Kali remotely from Windows using SSH.
- Build a foundation for future multi-VM cybersecurity exercises.

## Lab architecture

                         Internet
                            |
                    Windows physical network
                            |
                 +-----------------------+
                 | Windows host          |
                 |                       |
                 | VMnet8: NAT network    |
                 | VMnet1: 192.168.59.1   |
                 +-----------+-----------+
                             |
                  VMware Host-only network
                       192.168.59.0/24
                             |
                 +-----------+-----------+
                 | Kali Linux VM          |
                 |                        |
                 | eth1: 192.168.59.11/24  |  Host-only (verified)
                 | eth0: 192.168.217.157/24|  NAT (verified)
                 +------------------------+


Both adapter paths have been verified. NAT provides internet access, while Host-only provides a private connection between Windows and Kali.

## Environment

| Component | Configuration |
|---|---|
| Host OS | Windows |
| Virtualization | VMware Workstation |
| Guest OS | Kali Linux (64-bit) |
| Host-only virtual network | VMnet1 |
| Host VMnet1 address | `192.168.59.1/24` |
| Kali Host-only address | `192.168.59.11/24` on `eth1` |
| Host-only subnet | `192.168.59.0/24` |
| SSH | TCP port 22; verified from Windows to Kali |
| NAT adapter | `eth0` at `192.168.217.157/24` |
| NAT gateway | `192.168.217.2` |
| Default route | Via `192.168.217.2` on `eth0` |
| DNS server observed | `192.168.217.2` |

IP addresses can differ on another computer or after changing VMware's virtual network configuration. Use the values shown by your own system rather than copying these blindly.

## VMware configuration

1. Shut down the Kali VM before changing virtual hardware.
2. Open **VM > Settings** for the Kali VM.
3. Configure **Network Adapter 1** as **NAT** and enable **Connected** and **Connect at power on**.
4. Add **Network Adapter 2** and configure it as **Host-only**. Enable **Connected** and **Connect at power on**.
5. Open **Edit > Virtual Network Editor** in VMware Workstation (administrator privileges may be required).
6. Confirm that `VMnet1` is Host-only and that **Connect a host virtual adapter to this network** is enabled. Local DHCP can be enabled to assign addresses automatically.
7. Confirm that `VMnet8` is NAT. Keep the existing working Host-only settings unchanged while troubleshooting NAT.

VMware commonly uses VMnet1 for Host-only and VMnet8 for NAT by default, but verify the actual settings in Virtual Network Editor.

## Verification commands

### On Windows (PowerShell)

Show host and VMware virtual adapter addresses:

```powershell
ipconfig
```

Test Windows-to-Kali connectivity using Kali's Host-only address:

```powershell
ping 192.168.59.11
```

Connect to Kali over SSH:

```powershell
ssh psb@192.168.59.11
```

### On Kali Linux

Show interface addresses:

```bash
ip -br addr
```

Show routes:

```bash
ip route
```

Show NetworkManager device status and saved connections:

```bash
nmcli device status
nmcli connection show
```

Test Kali-to-Windows connectivity:

```bash
ping -c 4 192.168.59.1
```

Check SSH service status:

```bash
sudo systemctl status ssh
```

Check whether SSH is listening on TCP port 22:

```bash
sudo ss -tlnp | grep ':22'
```

## SSH setup

If the SSH server is not installed, install and start it on Kali:

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

The service was verified as `active (running)` and listening on port 22. The Windows SSH login to `psb@192.168.59.11` succeeded, and `whoami` returned `psb` inside the remote session.

When connecting for the first time, SSH may ask you to verify the server host key. Confirm that the key belongs to your own VM before accepting it. Never share your password or private SSH keys in this repository.

## Results verified (2026-10-20)

- [x] Windows VMnet1 adapter has address `192.168.59.1/24`.
- [x] Kali `eth1` has address `192.168.59.11/24`.
- [x] Windows can ping Kali: 4 packets received, 0% loss.
- [x] Kali can ping Windows: 4 packets received, 0% loss.
- [x] Kali SSH service is active and listening on port 22.
- [x] Windows can SSH to Kali as user `psb`.
- [x] Kali NAT adapter (`eth0`) has IPv4 address `192.168.217.157/24`.
- [x] Kali has a default route via `192.168.217.2` on `eth0`.
- [x] Kali can ping `8.8.8.8`: 4 packets received, 0% loss.
- [x] DNS lookup for `google.com` succeeds using DNS server `192.168.217.2`.

### NAT connection setup

When `eth0` was up but disconnected and had no IPv4 address, a DHCP NetworkManager profile was created after confirming the VMware adapter was set to NAT:

```bash
sudo nmcli connection add type ethernet ifname eth0 con-name VMware-NAT ipv4.method auto ipv6.method auto
sudo nmcli connection up VMware-NAT
```

The connection then obtained `192.168.217.157/24` and installed a default route through `192.168.217.2`. The following commands verified the result:

```bash
ip -br addr
ip route
nmcli device status
ping -c 4 8.8.8.8
getent hosts google.com
nslookup google.com
```

Observed key output:

```text
eth0  UP  192.168.217.157/24
eth1  UP  192.168.59.11/24

default via 192.168.217.2 dev eth0
192.168.59.0/24 dev eth1 src 192.168.59.11
192.168.217.0/24 dev eth0 src 192.168.217.157
```

The NAT adapter and Host-only adapter are now both active. Keep the Host-only connection on `eth1` unchanged while troubleshooting or maintaining NAT in the future.

## Safety and scope

- Use this lab only for systems and networks you own or are explicitly authorized to test.
- Keep vulnerable practice targets on isolated virtual networks; do not use Bridged mode for intentionally vulnerable machines unless you understand and accept the exposure.
- Host-only networking provides a private host/VM network, but it is not a substitute for backups or careful security practices.
- Do not commit passwords, private keys, access tokens, personal IP details, VM snapshots containing secrets, or other sensitive files.

## Planned next steps

1. Save a clean VM snapshot now that both network adapters are working.
2. Add a second test VM later and attach it to the Host-only network.
3. Practise basic subnetting, SSH, firewall rules, service discovery, and packet analysis only inside the lab.
4. Document each exercise with the objective, commands, expected results, actual results, and lessons learned.

## References

- [VMware Workstation Pro User Guide — virtual networking](https://techdocs2-prod.adobecqms.net/content/dam/broadcom/techdocs/us/en/pdf/vmware/desktop-hypervisors/workstation/vmware-workstation-pro-17-0.pdf)
- [Broadcom Developer Portal — VMware virtual networks](https://developer.broadcom.com/xapis/vmware-workstation-pro-api/latest/api/vmnets/post/)

---

*Personal learning lab documentation. Update the status checkboxes as each feature is tested on your own system.*

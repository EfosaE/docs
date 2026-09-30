# Enabling SSH Password / Root Login on the NPU VM

## Purpose

Allow SSH access to the NPU VM (`10.205.50.8`) from the Jump Server (`ai-gateway-hcs-ubuntu`) using a password, either as the `core` user (recommended) or as `root`.

**Starting state (from the console screenshots):**

- The VM prompt is `[core@localhost ~]$`, so the default login user is `core`, not `root`.
- `grep` on `/etc/ssh/sshd_config` showed only `Include /etc/ssh/sshd_config.d/*.conf`. `PermitRootLogin` and `PasswordAuthentication` are not set in the main file.
- `/etc/ssh/sshd_config.d/` contains at least `50-redhat.conf`.
- SSH from the Jump Server failed with `Permission denied, please try again.` for both `root@10.205.50.8` and `10.205.50.8` (the latter tries the Jump Server's own username, `root`).

> UNVERIFIED: the exact OS is not confirmed. `50-redhat.conf` and `core@localhost` suggest a RHEL-family or CoreOS-style image. Confirm with `cat /etc/os-release`.

---

## Key concept: how sshd reads its config

1. The `Include /etc/ssh/sshd_config.d/*.conf` line is near the **top** of `sshd_config`.
2. sshd uses the **first value it finds** for each option.
3. Drop-in files are read in **alphabetical order**.

Consequences:

- Values in a drop-in file **override** the same option set later in `sshd_config`. Editing the main file alone may have no effect if a drop-in such as `50-redhat.conf` sets the option.
- A drop-in named `01-...` is read before `50-redhat.conf`, so it wins.

This is why the procedure below uses a drop-in file instead of editing `sshd_config` directly.

---

## Procedure (run on the VM console)

### 1. Inspect the current state

```bash
ls -l /etc/ssh/sshd_config.d/
sudo sshd -T | grep -Ei '^(passwordauthentication|permitrootlogin|pubkeyauthentication)'
```

`sshd -T` prints the **effective** configuration, which is what matters.

### 2. Create the drop-in

Use a one-line command. Pasting a multi-line heredoc into the console breaks it (the shell waits at `>` for an `EOF` that never arrives).

**Option A: password login for `core` (recommended)**

```bash
echo 'PasswordAuthentication yes' | sudo tee /etc/ssh/sshd_config.d/01-enable-password.conf
sudo passwd core
```

**Option B: also allow root login over SSH**

```bash
sudo passwd root
echo 'PermitRootLogin yes' | sudo tee /etc/ssh/sshd_config.d/01-permit-root.conf
```

Root always exists (UID 0), but its password is usually locked. Check with:

```bash
getent passwd root
sudo passwd -S root      # L = locked, P = password set, NP = no password
```

### 3. Validate and apply

```bash
sudo sshd -t                        # prints nothing if the config is valid
sudo systemctl enable --now sshd
sudo systemctl restart sshd         # reload also works
sudo systemctl status sshd          # expect: Active: active (running)
```

### 4. Confirm the effective values

```bash
sudo sshd -T | grep -Ei '^(passwordauthentication|permitrootlogin)'
```

Expected:

```text
passwordauthentication yes
permitrootlogin yes        # only if Option B was applied
```

### 5. Connect from the Jump Server

```bash
ssh core@10.205.50.8       # Option A
ssh root@10.205.50.8       # Option B
```

With Option A, you can become root without a root password:

```bash
sudo -i
```

---

## Alternative: editing `sshd_config` directly

```bash
sudo nano /etc/ssh/sshd_config
```

Set:

```text
PermitRootLogin yes
PasswordAuthentication yes
```

Save with `Ctrl+O`, `Enter`, then `Ctrl+X`. Then restart sshd.

**Caveat:** the `Include` line is at the top of the file, so any drop-in that sets the same option (for example `50-redhat.conf`) takes precedence over your edit. Always verify with `sshd -T`. To list all active settings in the main file:

```bash
sudo grep -E '^(Include|PermitRootLogin|PasswordAuthentication)' /etc/ssh/sshd_config
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Permission denied` for `root` | Root login blocked (default `prohibit-password`) or root password locked/unset | Use `core` + `sudo -i`, or apply Option B |
| `Permission denied` for `core` | No password set, or `PasswordAuthentication` still `no` | `sudo passwd core`, then check `sshd -T` |
| `sshd -T` still shows `no` after the change | An earlier-sorting drop-in overrides yours | Rename your file to sort before it (for example `00-...`) or edit the conflicting file |
| Heredoc leaves the shell at `>` prompt | Multi-line paste broke the `EOF` terminator | `Ctrl+C`, then use the `echo ... \| sudo tee` one-liner |
| Cannot connect at all | sshd not running, wrong IP, or firewall | `systemctl status sshd`, `ip -br a`, check firewall rules |
| `ssh 10.205.50.8` fails | It uses your current local username (`root` on the Jump Server) | Specify the user: `ssh core@10.205.50.8` |

---

## Security notes

- Prefer **key-based authentication** over passwords. Add your public key to `~core/.ssh/authorized_keys` (or via the image's provisioning config).
- Avoid enabling `PermitRootLogin yes` on a network-reachable host. Use `core` + `sudo` where possible.
- Do not store passwords in this repo or in shell history.

## Rollback

```bash
sudo rm /etc/ssh/sshd_config.d/01-enable-password.conf /etc/ssh/sshd_config.d/01-permit-root.conf
sudo sshd -t && sudo systemctl restart sshd
```

---

# Addon: Configuring a Static IP with nmcli

Use this to set the VM's static IP (`10.205.50.8/25`) with NetworkManager. Run it on the VM **console**.

| Setting | Value |
|---|---|
| Interface / connection | `enp189s0f0` |
| IPv4 address | `10.205.50.8/25` (netmask `255.255.255.128`) |
| Gateway | `10.205.50.1` |
| DNS | `10.205.50.4` |

> UNVERIFIED: the commands assume the NetworkManager connection profile is named the same as the device (`enp189s0f0`). Step 1 shows the real profile names. If the profile name differs, use it in place of `enp189s0f0` in `nmcli connection modify` and `nmcli connection up`.

## 1. Check devices and profile names

```bash
nmcli device status
nmcli connection show
```

The `CONNECTION` column of `nmcli device status` shows which profile is bound to `enp189s0f0`.

## 2. Set the static IPv4 configuration

```bash
nmcli connection modify enp189s0f0 \
ipv4.method manual \
ipv4.addresses 10.205.50.8/25 \
ipv4.gateway 10.205.50.1 \
ipv4.dns 10.205.50.4
```

## 3. Verify the saved settings (not yet applied)

```bash
nmcli connection show enp189s0f0 | grep -E 'ipv4.method|ipv4.addresses|ipv4.gateway|ipv4.dns'
```

Expected:

```text
ipv4.method:                            manual
ipv4.addresses:                         10.205.50.8/25
ipv4.gateway:                           10.205.50.1
ipv4.dns:                               10.205.50.4
```

## 4. Apply

```bash
nmcli connection up enp189s0f0
```

## 5. Confirm it is live

```bash
ip -br a
ip route
ping -c 3 10.205.50.1
```

## Notes

- Run this from the VM console, not over SSH. Re-applying the connection can drop an SSH session, especially if the address changes.
- `nmcli connection modify` only edits the saved profile. Nothing changes on the interface until `nmcli connection up` runs.
- To add a second DNS server later: `nmcli connection modify enp189s0f0 +ipv4.dns <ip>`.
- To revert to DHCP: `nmcli connection modify enp189s0f0 ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""` and then `nmcli connection up enp189s0f0`.

# Percistence mode off for NVIDIA RTX Pro 5000 Blackwell

```bash
sudo systemctl enable --now nvidia-persistenced
```
Failed to enable unit: Unit file nvidia-persistenced.service does not exist.

That error tells that the persistence daemon unit was never installed; it’s optional and on RHEL 9 it often isn’t present by default, especially with the open driver stack.[^1]

### 1. Check if the binary exists at all and which package owns it

```bash
which nvidia-persistenced
rpm -qf "$(which nvidia-persistenced 2>/dev/null)" 2>/dev/null
```

- If `which` prints nothing, the daemon is not installed.
- If it prints a path but `rpm -qf` fails, it likely came from the `.run` installer and not from an RPM.


### 2. If you want the systemd service, install the right package

For RPM-based installs, the systemd unit file usually lives in the same package as the driver or a `*-persistence` subpackage. On RHEL‑like systems this is often provided by the main driver package; some vendor docs for RHEL 9 show the unit as part of their “GPU driver” RPM.[^1]

So the logic is:

```bash
# List all NVIDIA packages to see what you actually have
rpm -qa | grep -i nvidia | sort

# After confirming which package owns nvidia-persistenced (if any), reinstall it:
sudo dnf reinstall <that-package>
```

If after installing the packege and reinitializing the persistenced mode you still don’t get a unit file at `/usr/lib/systemd/system/nvidia-persistenced.service`, then the RHEL 9 packaging you’re using simply doesn’t ship it, and you can either:

- Skip persistence mode and just rely on default behavior (fine for most workstation use), or
- Run it manually in the background:

```bash
sudo nvidia-persistenced --user root
```

### 3. Consider: You may not need it at all

On modern datacenter/workstation usage, `nvidia-persistenced` mainly matters to:

- Keep GPU state warm between CUDA jobs on headless nodes.
- Avoid long re-init times when many short-lived processes use CUDA.

If your GPUs are already visible and stable and this is a workstation where GNOME is running on one GPU and compute jobs on the other, you can safely operate without the persistence daemon unless you see measurable overhead from repeated GPU initialization.

## You want persistenced service!!


### Check where is nvidia-persistenced and which package owns it for service
```bash
which nvidia-persistenced #/usr/bin/nvidia-persistenced >> `nvidia-persistenced` binary installed,
# but RHEL 9 setup simply doesn’t ship (or hasn’t installed) a corresponding systemd unit file, which is why `nvidia-persistenced.service` doesn’t exist
```

### Confirm which package owns it
```bash
rpm -qf /usr/bin/nvidia-persistenced #
```
This tells you which RPM dropped the binary. On many distros there is a separate `nvidia-persistenced` (or similar) package that also 
contains `/usr/lib/systemd/system/nvidia-persistenced.service`, but some RHEL‑side driver builds ship only the binary and expect you to manage it yourself

If the owning package does not provide a unit file under `/usr/lib/systemd/system/`, that’s expected for that packaging and not an error.

### Create your own unit. If no package owns or runs it, create your own

You can manage it with a tiny custom service:

```bash
sudo tee /etc/systemd/system/nvidia-persistenced.service << 'EOF'
[Unit]
Description=NVIDIA Persistence Daemon
After=multi-user.target

[Service]
Type=forking
ExecStart=/usr/bin/nvidia-persistenced --user root
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now nvidia-persistenced
systemctl status nvidia-persistenced
```

That gives you the same effect as the “official” unit: the daemon starts at boot and keeps the driver state warm

### It’s fine to skip it

If your GPUs are already behaving well and you are not doing latency‑sensitive HPC jobs with lots of very short-lived CUDA processes, 
you can just leave persistence mode off and not run the daemon at all. Modern drivers and RHEL 9 work fine without it for typical workstation use.



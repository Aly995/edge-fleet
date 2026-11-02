# edge-fleet

Ansible automation for a fleet of Raspberry Pi edge devices. Each device drives
USB/serial sensor peripherals and sends telemetry to a central API. The playbooks
take a freshly flashed Pi to a running, verified service, then keep the whole
fleet consistent over mixed local and cellular networks.

> This is a sanitized portfolio copy of a production project. All names,
> addresses, identities and secrets were replaced with placeholders. The
> application code that runs on the devices is not included.

---

## What it does

| Stage | Playbook | What happens |
|---|---|---|
| Validate | `preflight.yml` | Checks every per-device variable, format, flag and identity. Fails **before** anything changes. |
| Provision | `provision.yml` | Apt packages, I2C/SPI, UART overlays, kernel cmdline, groups, pinned Python/ML stack, LoRa radio stack. |
| Deploy | `deploy.yml` | rsync the agent, render its per-device `config.py`, install the systemd unit, byte-compile, import-test, confirm the service is active. |
| Observe | `health_check.yml` | Read-only sweep: service state, newest log age, offline queue. No sudo, no vault. |
| Everything | `site.yml` | All of the above, in order. |

### Hardware it manages

- **Raspberry Pi 5** (RP1 I/O chip) and **Raspberry Pi 4**, mixed in one fleet
- **GPIO** through the `rpi-lgpio` shim. `python3-rpi.gpio` gets removed because it does not work on the RP1 chip.
- **UART:** GPS on UART0 (GPIO14/15) and a microcontroller on GPIO4/5. That pair is UART2 on a Pi 5 and UART3 on a Pi 4.
- **SPI:** SX127x LoRa radio via `pyLoRa`, with a verified pin map
- **I2C:** SH1106 OLED status display
- **USB cameras**, bound by udev port path instead of `/dev/video*` index
- **ADB** to an attached Android device
- **systemd** service, plus **Ansible Vault** for per-device secrets

---

## Architecture

```
                 control node (Linux / WSL2)
        ┌────────────────────────────────────────────┐
        │ inventory/                                 │
        │   inventory.yml        hosts → profile_*   │
        │   group_vars/all/      fleet defaults      │
        │   group_vars/profile_* which peripherals   │
        │   host_vars/<device>/  per-unit + vault    │
        │ playbooks/  →  roles/                      │
        │   preflight · system_deps · lora_stack     │
        │   edge_agent · health_check                │
        └───────────────┬────────────────────────────┘
                        │ SSH (LAN IP, DNS or Tailscale name)
         ┌──────────────┼──────────────┬──────────────┐
         ▼              ▼              ▼              ▼
     device-001     device-002     device-003      … device-N
     Pi 5 / full    Pi 5 / full    Pi 4 / vision
     edge-agent.service → https://api.example.com/v1/telemetry
```

### Three variable tiers

Most fleet bugs come from a value set at the wrong tier.

| Tier | Where | Holds |
|---|---|---|
| Fleet | `group_vars/all/` | paths, API URL, GPIO standard, fail-safe flags (all **off**) |
| Profile | `group_vars/profile_*/` | which peripherals a model has |
| Unit | `host_vars/<device>/` | identity, secret, **board**, anything supply-dependent |

Hardware choices are deliberately made per unit, not per model. In practice two
units of the same model can ship with different boards, different
microcontrollers, or a dead GPIO. `edge_board: pi4` is the only setting you
change, and the correct UART overlay and `/dev/ttyAMA*` node are derived from it:

```yaml
edge_board_profiles:
  pi5: { serial_mcu_overlay: uart2-pi5, serial_mcu_port: /dev/ttyAMA2 }
  pi4: { serial_mcu_overlay: uart3,     serial_mcu_port: /dev/ttyAMA3 }
```

### Secrets

Each secret uses the public-name → `vault_*` indirection:

```yaml
# host_vars/device-001/main.yml   (plaintext, committed)
edge_device_secret: "{{ vault_device_secret }}"
# host_vars/device-001/vault.yml  (ansible-vault encrypted, git-ignored)
vault_device_secret: "…"
```

You can grep which secrets exist without decrypting anything. `preflight`
refuses a missing, short or placeholder secret. It also refuses two devices
sharing an ID or secret, comparing secrets as SHA-256 hashes so no value
reaches the log.

---

## Why Ansible and not containers

The devices exist to talk to hardware: GPIO lines, three UARTs, SPI, I2C,
USB cameras and an ADB-attached phone. Containerising that means
`--privileged` or a long list of `--device` mappings, udev rules duplicated
inside the image, and boot-level state that a container cannot own anyway:

- `/boot/firmware/config.txt` overlays (`uart2-pi5`, `disable-bt`)
- the kernel cmdline (freeing the serial console for GPS)
- `raspi-config` bus enablement
- group membership and reboots

That state is the hard part of the job, and it is host configuration, which
is exactly what Ansible is for. Ansible is also agentless: nothing extra runs
on a Pi with 4–8 GB of RAM, and a device only needs SSH and Python, both of
which ship with Raspberry Pi OS.

---

## Usage

```bash
# 1. Control node
python3 -m pip install --user ansible-core
ansible-galaxy collection install -r requirements.yml -p ./collections

# 2. Inventory: copy the templates and fill them in
cp inventory/inventory.example.yml inventory/inventory.yml
cp inventory/group_vars/all/vault.example.yml inventory/group_vars/all/vault.yml
mkdir -p inventory/host_vars/device-001
cp inventory/host_vars/device-001.example.yml       inventory/host_vars/device-001/main.yml
cp inventory/host_vars/device-001.vault.example.yml inventory/host_vars/device-001/vault.yml
ansible-vault encrypt inventory/group_vars/all/vault.yml inventory/host_vars/*/vault.yml
head -c 14 inventory/host_vars/device-001/vault.yml   # must print $ANSIBLE_VAULT

# 3. Key-based SSH to each device (rsync cannot use --ask-pass)
ssh-copy-id edgeuser@192.0.2.11

# 4. Run
ansible-playbook playbooks/preflight.yml --ask-vault-pass
ansible-playbook playbooks/provision.yml -e target=device-001 --ask-vault-pass -K
ansible-playbook playbooks/deploy.yml    --ask-vault-pass -K
ansible-playbook playbooks/deploy.yml    --tags config -e target=device-003 --ask-vault-pass -K
ansible-playbook playbooks/health_check.yml
```

Useful tags: `system_deps`, `packages`, `boot`, `python`, `lora`, `code`, `config`.

---

## How it scales

- **Adding a device** means one inventory line and one `host_vars/<device>/`
  directory. Roles and playbooks don't change.
- **Adding a model** means one `group_vars/profile_<name>/main.yml` with six booleans.
- **Rolling deploys:** `deploy.yml` uses `serial: 1` with `max_fail_percentage: 0`
  by default, so the first failure stops the rollout. For hundreds of devices,
  pass `-e batch=10%`. `provision.yml` defaults to 25% batches.
- **Parallelism and speed:** `forks = 20`, SSH pipelining and ControlPersist,
  plus jsonfile fact caching so repeated runs skip fact gathering.
- **Safety at scale:**
  - `preflight` runs first in every write playbook. It catches duplicated
    identities, missing vars and wrong-board settings across the whole inventory.
  - It also logs in to compare the device's real hostname and user with the
    inventory, so a DHCP lease that moved to another machine can't be deployed to.
- **Networks:** `ansible_host` can be a LAN IP, a DNS name or a Tailscale
  MagicDNS name. Joining devices to a tailnet happens outside this repository.

---

## Hard-won details encoded in the roles

- **Pi 5 GPIO:** `RPi.GPIO` cannot drive the RP1 chip. `lora_stack` removes it
  and installs `rpi-lgpio`, which keeps the `RPi.GPIO` API that pyLoRa imports.
- **Two classes in one file:** pyLoRa's `board_config.py` defines `class BOARD`
  and `class BOARD2`. `lineinfile` edits the *last* match, so it wrote the pin map
  into dead code. The role uses `replace` bounded with `after`/`before`, then
  imports the library and asserts the pins it actually reports.
- **The `(?m)` flag:** `replace` compiles `after`/`before` without
  `re.MULTILINE`, so an anchor like `^class BOARD:` silently matched nothing and
  the task reported `ok`. The role carries the flag and asserts on the module
  message, so the failure surfaces at the task that caused it.
- **Silent overlays:** the Pi 5 UART overlay loads into nothing on a Pi 4, with
  no error. The fix was to derive overlay and port from one `edge_board` value.
- **Lost reboots:** a play that fails after a boot-config change never runs its
  handlers, and a re-run doesn't re-notify. `force_handlers = True` fixes this.
- **Pinned ML wheels:** an unpinned `torch` wheel crashed with SIGILL on the
  Pi 5's Cortex-A76. Versions are pinned, and provisioning smoke-tests a matrix
  multiply on the device.
- **Import-test after deploy:** `compileall` passes on a broken package layout,
  so `deploy.yml` also imports the package on the device.

---

## Verification status of this copy

| Check | Result |
|---|---|
| `ansible-lint` (profile `production`) | passed, 0 failures, 0 warnings |
| `ansible-playbook --syntax-check` (all 5 playbooks) | passed |
| Templates rendered for two example devices, result imported by Python | passed (Pi 5 → `/dev/ttyAMA2`, Pi 4 → `/dev/ttyAMA3`) |
| `preflight` negative tests: missing vars, duplicate ID, placeholder secret | each failed as intended |
| Run against physical devices | **not done for this restructured copy.** The tasks come from a production fleet, but the renamed and restructured layout has only been linted and tested locally. |

Known trade-off: Python packages install with `pip --user --break-system-packages`,
matching what runs in the field. A venv with `--system-site-packages` would be
cleaner, and that's the planned next step.

---

## Layout

```
edge-fleet/
├── ansible.cfg            forks, pipelining, fact cache, force_handlers
├── requirements.yml       ansible.posix
├── .ansible-lint          production profile
├── inventory/
│   ├── inventory.example.yml
│   ├── group_vars/all/{main.yml, vault.example.yml}
│   ├── group_vars/profile_{full,vision,bridge}/main.yml
│   └── host_vars/device-00{1,3}.example.yml, device-001.vault.example.yml
├── playbooks/  site · preflight · provision · deploy · health_check
└── roles/
    ├── preflight/     validation + live identity check
    ├── system_deps/   apt, buses, overlays, pinned pip stack, reboot handler
    ├── lora_stack/    rpi-lgpio, pyLoRa, verified SX127x pin map
    ├── edge_agent/    rsync, config.py + systemd templates, verify, restart handler
    └── health_check/  read-only liveness
```

---

Built by [bayanatk.com](https://bayanatk.com). MIT licensed.

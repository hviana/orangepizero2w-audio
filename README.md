# enable orangepi zero 2w audio and gpu
(Thanks to Totof in https://dietpi.com/forum/t/no-sound-card-detected-for-opi-zero-2w/20133/89)
compile:
```bash
#!/usr/bin/env bash

set -euo pipefail
[[ $EUID -eq 0 ]] || { echo "Run as root (sudo)"; exit 1; }

# Create the directory for user overlays
# Create the overlay source file
cat << '_EOF_' > sun50i-h616-audiogpu.dts
/dts-v1/;
/plugin/;
/ {
	compatible = "allwinner,sun50i-h618";
	fragment@0 {
		target = <&pio>;
		__overlay__ {
			ahub_daudio0_pins_d: ahub_daudio0_sleep {
				pins = "PI0", "PI1", "PI2", "PI3", "PI4";
				function = "gpio_in";
				drive-strength = <0x14>;
				bias-disable;
			};
			ahub_daudio0_pins_a: ahub_daudio0@0 {
				pins = "PI0", "PI1", "PI2";
				function = "i2s0";
				drive-strength = <0x14>;
				bias-disable;
			};
			ahub_daudio0_pins_b: ahub_daudio0@1 {
				pins = "PI3";
				function = "i2s0_dout0";
				drive-strength = <0x14>;
				bias-disable;
			};
			ahub_daudio0_pins_c: ahub_daudio0@2 {
				pins = "PI4";
				function = "i2s0_din0";
				drive-strength = <0x14>;
				bias-disable;
			};
		};
	};
	fragment@1 {
		target-path = "/soc";
		__overlay__ {
			ahub0_plat: ahub0_plat {
				#sound-dai-cells = <0>;
				compatible = "allwinner,sunxi-snd-plat-ahub";
				apb_num = <0>; /* for dma port 3 */
				dmas = <&dma 3>, <&dma 3>;
				dma-names = "tx", "rx";
				playback_cma = <128>;
				capture_cma = <128>;
				tx_fifo_size = <128>;
				rx_fifo_size = <128>;
				tdm_num         = <0>;
				tx_pin          = <0>;
				rx_pin          = <0>;
				pinctrl-names = "default", "sleep";
				pinctrl_used;
				pinctrl-0 = <&ahub_daudio0_pins_a>, <&ahub_daudio0_pins_b>, <&ahub_daudio0_pins_c>;
				pinctrl-1 = <&ahub_daudio0_pins_d>;
				status = "okay";
			};
			ahub0_mach: ahub0_mach {
				compatible = "allwinner,sunxi-snd-mach";
				soundcard-mach,name = "ahubi2s0";
				soundcard-mach,format = "i2s";
				soundcard-mach,frame-master = <&ahub0_cpu>;
			        soundcard-mach,bitclock-master = <&ahub0_cpu>;
			        soundcard-mach,slot-num = <2>;
			        soundcard-mach,slot-width = <32>;
				status = "okay";

				ahub0_cpu: soundcard-mach,cpu {
					sound-dai = <&ahub0_plat>;
					soundcard-mach,pll-fs = <4>;
				        soundcard-mach,mclk-fs = <0>;
				};

				soundcard-mach,codec {
				};
			};
		};
	};
	fragment@2 {
		target-path = "/soc";
		__overlay__ {
			codec: codec {
			                       compatible = "allwinner,sun50i-h616-codec";
			                       status = "okay";
			                       allwinner,audio-routing = "Line Out", "LINEOUT";
			                       };
			 };
	};
	fragment@3 {
		target-path = "/soc";
		__overlay__ {
			ahub_dam_plat: ahub_dam_plat {
			                       compatible = "allwinner,sunxi-snd-plat-ahub_dam";
			                       status = "okay";
			                       };
			 };
	};
	fragment@4 {
		target-path = "/soc";
		__overlay__ {
			ahub1_plat: ahub1_plat {
			                       compatible = "allwinner,sunxi-snd-plat-ahub";
			                       #sound-dai-cells = <0x00>;
			                       status = "okay";
			                       };
			 };
	};
	fragment@5 {
		target-path = "/soc";
		__overlay__ {
			ahub1_mach: ahub1_mach {
			                       compatible = "allwinner,sunxi-snd-mach";
			                       soundcard-mach,name = "HDMI";
			                       soundcard-mach,format = "i2s";
			                       status = "okay";
			                       
			                       soundcard-mach,cpu {
					           sound-dai = <&ahub1_plat>;
				                   };

				               soundcard-mach,codec {
				                   };
			                       };
			 };
	};
	fragment@6 {
		target-path = "/soc";
		__overlay__ {
			gpu: gpu {
			                       compatible = "allwinner,sun50i-h616-mali", "arm,mali-bifrost";
			                       status = "okay";
			                       };
			 };
	};
};
_EOF_

# Install the device tree compiler
sudo apt install -y device-tree-compiler
# Compile the overlay binary file
dtc -@ -I dts -O dtb -o sun50i-h616-audiogpu.dtbo sun50i-h616-audiogpu.dts

#copy to overlays:
sudo cp sun50i-h616-audiogpu.dtbo /boot/dtb/allwinner/overlay/

echo "FINALLY, you MUST ENABLE audiogpu OVERLAY ON KERNEL CONFIG!"

```

# Games to run:
```bash
sudo apt install -y supertux frozen-bubble freedoom openarena torcs
```

# Enable **Allwinner SoC Audio support V2** options in Armbian compilation with (*):

~~~
Device Drivers
  → Sound card support
    → Advanced Linux Sound Architecture
      → ALSA for SoC audio support
        → Allwinner SoC Audio support V2
			→ Allwinner AAUDIO support -> if you use analog audio
			→ Allwinner AHUB Support -> if you use hdmi/digital audio

~~~
#ps4 joystick
#enable modules
```bash
#!/usr/bin/env bash
# enable-ps4-joystick.sh
# Purpose: Enable and auto-load everything a PS4 (DualShock 4) controller needs
# on Armbian/Debian — both over USB and Bluetooth.
# - Loads kernel modules now and on every boot
# - Ensures BlueZ starts with input/HOG plugins (no --noplugin=input)
# - Creates sane udev permissions for /dev/input/js*
# - Adds your user to the 'input' group (for joystick access)
#
# Usage:
#   sudo bash enable-ps4-joystick.sh [--user <username>]
#
# After running:
#   Pair over BT: hold SHARE + PS until fast flashing, then:
#     bluetoothctl
#       agent on
#       default-agent
#       scan on
#       pair XX:XX:XX:XX:XX:XX
#       trust XX:XX:XX:XX:XX:XX
#       connect XX:XX:XX:XX:XX:XX

set -Eeuo pipefail

# ---------- helpers ----------
log() { printf "\033[1;32m[OK]\033[0m %s\n" "$*"; }
warn() { printf "\033[1;33m[WARN]\033[0m %s\n" "$*"; }
err() { printf "\033[1;31m[ERR]\033[0m %s\n" "$*" >&2; }
need_root() { [[ $EUID -eq 0 ]] || { err "Run as root (sudo)."; exit 1; }; }

need_root

TARGET_USER="${SUDO_USER:-${USER:-}}"
# Allow override via --user <name>
if [[ "${1:-}" == "--user" && -n "${2:-}" ]]; then
  TARGET_USER="$2"
fi
if [[ -z "${TARGET_USER}" || "${TARGET_USER}" == "root" ]]; then
  warn "No non-root user detected to grant joystick access. You can rerun with --user <name> later."
fi

# ---------- packages ----------
# Ensure BlueZ and tools are present (no-op if already installed)
if command -v apt-get >/dev/null 2>&1; then
  export DEBIAN_FRONTEND=noninteractive
  apt-get update -y || true
  apt-get install -y --no-install-recommends \
    bluez bluetooth rfkill joystick evtest >/dev/null 2>&1 || true
  log "Ensured packages: bluez bluetooth rfkill joystick evtest"
fi

# ---------- modules to load ----------
# We choose a broad, safe set. Missing ones will be skipped gracefully.
ALL_MODULES=(
  bluetooth    # BT core
  btusb btrtl btintel btmtk ath3k  # common BT USB chipsets (load if present)
  hidp         # legacy HIDP (still useful for some pads)
  uhid uinput  # userspace HID + input emulator
  joydev evdev # /dev/input/js* + evdev
  hid_sony     # DualShock/DualSense family (DS4 uses this)
  hid_generic usbhid hid  # USB HID stack
  hid_multitouch           # not strictly required, harmless if present
)

FOUND_MODULES=()
for m in "${ALL_MODULES[@]}"; do
  if modinfo "$m" >/dev/null 2>&1; then
    FOUND_MODULES+=("$m")
  fi
done

if [[ ${#FOUND_MODULES[@]} -eq 0 ]]; then
  err "No relevant kernel modules found. Are kernel headers/modules installed?"
  exit 1
fi

# ---------- persist modules across boots ----------
MODULES_FILE="/etc/modules-load.d/ps4-joystick.conf"
{
  echo "# Autoloaded modules for PS4 (DualShock 4) controller"
  echo "# Generated by $(basename "$0") on $(date -Iseconds)"
  for m in "${FOUND_MODULES[@]}"; do echo "$m"; done
} > "$MODULES_FILE"
log "Wrote $MODULES_FILE"

# ---------- load now ----------
for m in "${FOUND_MODULES[@]}"; do
  if modprobe -q "$m"; then
    : # loaded
  else
    warn "Could not load module: $m (skipping)"
  fi
done
log "Requested immediate load of detected modules"

# ---------- BlueZ service: make sure input plugin is NOT disabled ----------
# If the unit ever used '--noplugin=input', replace ExecStart via drop-in.
if systemctl cat bluetooth.service 2>/dev/null | grep -q -- '--noplugin=input'; then
  mkdir -p /etc/systemd/system/bluetooth.service.d
  BLUETOOTHD_PATH="/usr/lib/bluetooth/bluetoothd"
  [[ -x "$BLUETOOTHD_PATH" ]] || BLUETOOTHD_PATH="/usr/libexec/bluetooth/bluetoothd"
  if [[ ! -x "$BLUETOOTHD_PATH" ]]; then
    warn "Could not find bluetoothd binary to override ExecStart; continuing anyway."
  else
    cat > /etc/systemd/system/bluetooth.service.d/override.conf <<EOF
[Service]
ExecStart=
ExecStart=${BLUETOOTHD_PATH}
EOF
    systemctl daemon-reload
    log "Installed bluetooth.service drop-in without '--noplugin=input'"
  fi
else
  log "BlueZ input plugin not disabled — no override needed"
fi

# Ensure bluetooth service is enabled and running
systemctl enable --now bluetooth.service >/dev/null 2>&1 || true
log "Ensured bluetooth.service is enabled and active"

# ---------- udev: permissions for /dev/input/js* ----------
# Grant group 'input' read/write on joystick nodes.
mkdir -p /etc/udev/rules.d
cat > /etc/udev/rules.d/99-joystick-perms.rules <<'EOF'
# PS4 / general joystick access: allow members of 'input' group to read/write js nodes
KERNEL=="js[0-9]*", MODE="0660", GROUP="input"
EOF
udevadm control --reload-rules
udevadm trigger --subsystem-match=input || true
log "Installed udev rule 99-joystick-perms.rules"

# ---------- add user to input group ----------
if [[ -n "${TARGET_USER}" && "${TARGET_USER}" != "root" && "$(getent passwd "$TARGET_USER")" ]]; then
  if id -nG "$TARGET_USER" | tr ' ' '\n' | grep -qx input; then
    log "User '$TARGET_USER' already in 'input' group"
  else
    usermod -aG input "$TARGET_USER"
    log "Added '$TARGET_USER' to 'input' group (relog to take effect)"
  fi
else
  warn "Skipped adding user to 'input' group (no suitable non-root user provided)"
fi

# ---------- final checks ----------
sleep 1
if ls /dev/input/js* >/dev/null 2>&1; then
  log "Joystick device(s) found: $(ls -1 /dev/input/js* | xargs -n1 basename | paste -sd, -)"
else
  warn "No /dev/input/js* yet. This is normal until the controller is connected and recognized."
  warn "After pairing/plugging, you should see /dev/input/js0 appear."
fi

echo
log "Done. Next steps:"
echo " - For USB: plug in the controller; it should load 'hid_sony' and create /dev/input/js0."
echo " - For Bluetooth: put controller in pairing mode (SHARE+PS), then pair via 'bluetoothctl'."
echo " - Test: 'jstest /dev/input/js0' (from 'joystick' package) or 'evtest' on the matching event node."
```

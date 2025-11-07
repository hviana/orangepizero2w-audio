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
	compatible = "xunlong,orangepi-zero2w",
		     "allwinner,sun50i-h618",
		     "allwinner,sun50i-h616";

    fragment@0 {
        target = <&pio>;
        __overlay__ {
            ahub_daudio0_pins_sleep: ahub-daudio0-pins-sleep {
                pins = "PI0", "PI1", "PI2", "PI3", "PI4";
                function = "gpio_in";
                drive-strength = <20>;
                bias-disable;
            };

            ahub_daudio0_pins_a: ahub-daudio0-pins-a {
                pins = "PI0", "PI1", "PI2";
                function = "i2s0";
                drive-strength = <20>;
                bias-disable;
            };

            ahub_daudio0_pins_b: ahub-daudio0-pins-b {
                pins = "PI3";
                function = "i2s0_dout0";
                drive-strength = <20>;
                bias-disable;
            };

            ahub_daudio0_pins_c: ahub-daudio0-pins-c {
                pins = "PI4";
                function = "i2s0_din0";
                drive-strength = <20>;
                bias-disable;
            };
        };
    };

    /* ---------- AHUB platform + machine (retain as new /soc children) ---------- */
    fragment@1 {
        target-path = "/soc";
        __overlay__ {
            ahub0_plat: ahub0-plat {
                #sound-dai-cells = <0>;
                compatible = "allwinner,sunxi-snd-plat-ahub";
                apb_num = <0>; /* DMA port 3 */
                dmas = <&dma 3>, <&dma 3>;
                dma-names = "tx", "rx";
                playback_cma = <128>;
                capture_cma  = <128>;
                tx_fifo_size = <128>;
                rx_fifo_size = <128>;
                tdm_num = <0>;
                tx_pin  = <0>;
                rx_pin  = <0>;

                pinctrl-names = "default", "sleep";
                pinctrl-0 = <&ahub_daudio0_pins_a>, <&ahub_daudio0_pins_b>, <&ahub_daudio0_pins_c>;
                pinctrl-1 = <&ahub_daudio0_pins_sleep>;

                status = "okay";
            };

            ahub0_mach: ahub0-mach {
                compatible = "allwinner,sunxi-snd-mach";
                soundcard-mach,name           = "ahubi2s0";
                soundcard-mach,format         = "i2s";
                soundcard-mach,frame-master   = <&ahub0_cpu>;
                soundcard-mach,bitclock-master= <&ahub0_cpu>;
                soundcard-mach,slot-num       = <2>;
                soundcard-mach,slot-width     = <32>;
                status = "okay";

                ahub0_cpu: soundcard-mach,cpu {
                    sound-dai = <&ahub0_plat>;
                    soundcard-mach,pll-fs  = <4>;
                    soundcard-mach,mclk-fs = <0>;
                };

                soundcard-mach,codec { };
            };
        };
    };

    /* ---------- Codec: patch existing node instead of creating a new one ---------- */
    fragment@2 {
        target = <&codec>;
        __overlay__ {
            status = "okay";
            allwinner,audio-routing = "Line Out", "LINEOUT";
        };
    };

    /* ---------- AHUB DAM platform ---------- */
    fragment@3 {
        target-path = "/soc";
        __overlay__ {
            ahub_dam_plat: ahub-dam-plat {
                compatible = "allwinner,sunxi-snd-plat-ahub_dam";
                status = "okay";
            };
        };
    };

    /* ---------- Secondary AHUB platform ---------- */
    fragment@4 {
        target-path = "/soc";
        __overlay__ {
            ahub1_plat: ahub1-plat {
                compatible = "allwinner,sunxi-snd-plat-ahub";
                #sound-dai-cells = <0>;
                status = "okay";
            };
        };
    };

    /* ---------- Secondary machine ---------- */
    fragment@5 {
        target-path = "/soc";
        __overlay__ {
            ahub1_mach: ahub1-mach {
                compatible = "allwinner,sunxi-snd-mach";
                soundcard-mach,name   = "HDMI";
                soundcard-mach,format = "i2s";
                status = "okay";

                soundcard-mach,cpu {
                    sound-dai = <&ahub1_plat>;
                };

                soundcard-mach,codec { };
            };
        };
    };

    /* ---------- GPU: patch existing node ---------- */
    fragment@6 {
        target = <&gpu>;
        __overlay__ {
            status = "okay";
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

If you needs AHUB:
```bash
#!/usr/bin/env bash
# ahub-split-mods-v2.sh — build ONLY the AHUB pieces from mainline+patch, robust init detection
# Requires: exact headers at /lib/modules/$(uname -r)/build

set -euo pipefail
[ "$EUID" -eq 0 ] || { echo "Run as root (sudo)"; exit 1; }

PATCH_URL="${PATCH_URL:-https://gitlab.manjaro.org/manjaro-arm/packages/core/linux-rc/-/blob/master/0621-sound-soc-add-sunxi_v2-for-h616-ahub.patch?ref_type=heads}"
KVER="$(uname -r)"
KDIR="/lib/modules/${KVER}/build"
DEST="/lib/modules/${KVER}/extra"
CONF="/etc/modules-load.d/ahub.conf"

[ -d "$KDIR" ] || { echo "Missing headers/build dir: $KDIR"; exit 1; }

apt-get update -y >/dev/null 2>&1 || true
apt-get install -y --no-install-recommends git curl patch build-essential >/dev/null

WORK="/usr/local/src/ahub-split"
rm -rf "$WORK"; mkdir -p "$WORK"; cd "$WORK"

# 1) mainline + patch
git clone --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git mainline
cd mainline
RAW_URL="$(printf '%s' "$PATCH_URL" | sed -E 's@/-/blob/@/-/raw/@')"
curl -fsSL "$RAW_URL" -o ahub.patch
git apply --whitespace=nowarn ahub.patch || patch -p1 --forward < ahub.patch || patch -p0 --forward < ahub.patch

# 2) locate AHUB dir
AHUBDIR=""
for d in sound/soc/sunxi_v2 sound/soc/sunxi; do
  if [ -d "$d" ] && find "$d" -maxdepth 1 -type f -name '*ahub*.c' | grep -q .; then
    AHUBDIR="$PWD/$d"; break
  fi
done
[ -n "$AHUBDIR" ] || { echo "Patch applied, but no AHUB sources found."; exit 1; }
cd "$AHUBDIR"

# 3) detect init-bearing sources (many drivers use macros, not module_init)
mapfile -t ALL_C < <(ls -1 *.c 2>/dev/null | sort)
PATTERN='module_init\(|module_exit\(|module_platform_driver\(|module_spi_driver\(|module_i2c_driver\(|subsys_initcall\(|builtin_platform_driver\('
mapfile -t INIT_C < <(grep -El "$PATTERN" -- *.c 2>/dev/null | sort || true)

# Fallback to well-known filenames if grep found nothing
if [ "${#INIT_C[@]}" -eq 0 ]; then
  for guess in snd_sunxi_ahub.c snd_sunxi_ahub_dam.c snd_sunxi_mach.c; do
    [ -f "$guess" ] && INIT_C+=("$guess")
  done
fi
# Last-resort: pick the biggest 1–3 files (likely to contain init code)
if [ "${#INIT_C[@]}" -eq 0 ]; then
  mapfile -t INIT_C < <(ls -1S *.c | head -n 3)
fi

# helpers = ALL - INIT
HELPERS=()
for f in "${ALL_C[@]}"; do
  skip=0; for g in "${INIT_C[@]}"; do [[ "$f" == "$g" ]] && { skip=1; break; }; done
  [[ $skip -eq 0 ]] && HELPERS+=("$f")
done

# 4) Makefile: one module per init-file, each linked with all helpers (avoids multiple init symbol clashes)
mv -f Makefile Makefile.bak.ahub 2>/dev/null || true
{
  echo -n "obj-m :="
  for init in "${INIT_C[@]}"; do
    base="${init%.c}"
    echo -n " ${base}.o"
  done
  echo
  for init in "${INIT_C[@]}"; do
    base="${init%.c}"
    echo -n "${base}-objs := ${base}.o"
    for h in "${HELPERS[@]}"; do
      echo -n " ${h%.c}.o"
    done
    echo
  done
} > Makefile

echo "Building AHUB modules from: $(basename "$AHUBDIR")"
make -C "$KDIR" M="$AHUBDIR" -j"$(nproc)" modules

# 5) install + depmod
mkdir -p "$DEST"
built=( $(find "$AHUBDIR" -maxdepth 1 -type f -name '*.ko' -printf '%f\n') )
[ "${#built[@]}" -gt 0 ] || { echo "No .ko produced (headers/ABI mismatch?)."; exit 1; }
for ko in "${built[@]}"; do
  cp -f "$AHUBDIR/$ko" "$DEST/"
done
command -v depmod >/dev/null 2>&1 && depmod "$KVER" || true

# 6) autoload the likely core modules (idempotent)
touch "$CONF"
for name in snd_sunxi_ahub snd_sunxi_ahub_dam snd_sunxi_mach; do
  grep -qx "$name" "$CONF" 2>/dev/null || echo "$name" >> "$CONF"
done

echo "Done ✅
• Installed to: $DEST
• Built modules: ${built[*]}
• Autoload list: $(tr '\n' ' ' < "$CONF")
Reboot, then check:  lsmod | grep -E 'sunxi|ahub'  &&  dmesg | grep -i ahub"

```

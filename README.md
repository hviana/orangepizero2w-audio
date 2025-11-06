# enable orangepi zero 2w audio and gpu
(Thanks to Totof in https://dietpi.com/forum/t/no-sound-card-detected-for-opi-zero-2w/20133/89)
compile:
```bash
#!/usr/bin/env bash
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

#install AHUB (kernel 6.16.8+)
set -Eeuo pipefail

# --- helpers --------------------------------------------------------------

require_root() {
  if [[ ${EUID:-$(id -u)} -ne 0 ]]; then
    exec sudo -E "$0" "$@"
  fi
}

have() { command -v "$1" >/dev/null 2>&1; }

die() { echo "Error: $*" >&2; exit 1; }

# Pure-bash Levenshtein distance (O(n*m)) + common-prefix bonus for "smart" similarity
# Returns a numeric score where LOWER distance and HIGHER prefix length => LOWER score (better).
levenshtein() {
  local s1=$1 s2=$2
  local -i len1=${#s1} len2=${#s2}
  local -a prev curr
  local -i i j cost del ins sub

  # prefix bonus (larger common prefix => better); we subtract it later to improve similarity
  local -i p=0
  local -i minlen=$(( len1 < len2 ? len1 : len2 ))
  for ((i=0;i<minlen;i++)); do
    [[ ${s1:i:1} == "${s2:i:1}" ]] || break
    ((p++))
  done

  for ((j=0;j<=len2;j++)); do prev[j]=j; done
  for ((i=1;i<=len1;i++)); do
    curr[0]=$i
    for ((j=1;j<=len2;j++)); do
      cost=$(( ${s1:i-1:1} == "${s2:j-1:1}" ? 0 : 1 ))
      del=$(( prev[j] + 1 ))
      ins=$(( curr[j-1] + 1 ))
      sub=$(( prev[j-1] + cost ))
      curr[j]=$del
      (( ins < curr[j] )) && curr[j]=$ins
      (( sub < curr[j] )) && curr[j]=$sub
    done
    prev=("${curr[@]}")
  done

  # We bias the score with prefix bonus (bigger prefix lowers the score).
  # Also include absolute length gap to prefer very close lengths.
  local -i dist=${prev[len2]}
  local -i len_gap=$(( len1 > len2 ? len1 - len2 : len2 - len1 ))
  # Final score: distance + small length gap penalty - prefix bonus
  echo $(( dist + (len_gap/2) - p ))
}

best_header_pkg() {
  local kr="$(uname -r)"
  local best_pkg="" best_suffix="" best_score=999999

  # 1) Gather candidate names from APT (package names only)
  mapfile -t pkgs < <(apt-cache --names-only search '^linux-headers-' 2>/dev/null | while read -r a _; do [[ $a == linux-headers-* ]] && echo "$a"; done)

  # 2) Fallback to any installed headers in /usr/src (derive a pkg-like name)
  if ((${#pkgs[@]}==0)); then
    for d in /usr/src/linux-headers-*; do
      [[ -d "$d" ]] || continue
      pkgs+=("$(basename "$d")")
    done
  fi

  ((${#pkgs[@]})) || die "No linux-headers packages found via APT or /usr/src."

  # 3) Score each candidate against uname -r using our pure-bash similarity
  for pkg in "${pkgs[@]}"; do
    local suffix="${pkg#linux-headers-}"
    local score
    score=$(levenshtein "$kr" "$suffix")
    if (( score < best_score )); then
      best_score=$score
      best_pkg="$pkg"
      best_suffix="$suffix"
    fi
  done

  echo "$best_pkg"
}

try_install_pkg() {
  local pkg=$1
  if dpkg -s "$pkg" >/dev/null 2>&1; then
    echo "[$pkg] already installed."
  else
    echo "Installing [$pkg]..."
    apt-get update -y
    apt-get install -y "$pkg"
  fi
}

load_ahub_module() {
  # Try a handful of plausible module names across sunxi variants.
  local candidates=(
    sun8i-ahub
    sun50i-ahub
    sunxi-ahub
    snd-soc-sun8i-ahub
    snd-sun8i-ahub
    snd-soc-sunxi-ahub
  )

  for m in "${candidates[@]}"; do
    if modinfo "$m" >/dev/null 2>&1; then
      echo "Found module [$m]; attempting to load..."
      if modprobe "$m"; then
        echo "Loaded [$m] successfully ✅"
        return 0
      fi
    fi
  done

  echo "Could not find a loadable AHUB module on this kernel."
  echo "If your kernel doesn't ship AHUB as a module, you may need a kernel with AHUB enabled or build it via DKMS."
  return 1
}

# --- main ----------------------------------------------------------------

require_root "$@"

KREL="$(uname -r)"
echo "Running kernel: $KREL"
echo "Discovering best-matching linux-headers package using a pure-bash similarity algorithm..."

HDR_PKG="$(best_header_pkg)"
[[ -n "$HDR_PKG" ]] || die "Failed to determine a headers package."

echo
echo "Suggested headers package: $HDR_PKG"
read -r -p "Install this header for \"$KREL\"? [Y/n] " ans
ans=${ans:-Y}
if [[ "$ans" =~ ^[Yy]$ ]]; then
  try_install_pkg "$HDR_PKG"
else
  echo "Please install the correct headers for your kernel manually (matching: $(uname -r)), then re-run this script."
  exit 1
fi

# Try to also install 'linux-modules-extra-<suffix>' if it exists (Debian generic often uses this)
SUFFIX="${HDR_PKG#linux-headers-}"
EXTRA="linux-modules-extra-$SUFFIX"
if apt-cache policy "$EXTRA" 2>/dev/null | grep -q 'Candidate:'; then
  try_install_pkg "$EXTRA" || true
fi

echo
echo "Attempting to load an AHUB module..."
if load_ahub_module; then
  echo "Done."
  exit 0
fi

echo
echo "AHUB module not present or not loadable."
echo "Next steps:"
echo "  1) Ensure you're on a kernel that includes Allwinner AHUB as a module."
echo "  2) If not, build AHUB via DKMS or switch to a kernel flavor that ships it."
exit 2



# Install the device tree compiler
apt install device-tree-compiler
# Compile the overlay binary file
dtc -@ -I dts -O dtb -o sun50i-h616-audiogpu.dtbo sun50i-h616-audiogpu.dts

#copy to overlays:
sudo cp sun50i-h616-audiogpu.dtbo /boot/dtb/allwinner/overlay/

#FINALLY, ENABLE audiogpu OVERLAY ON KERNEL CONFIG!

```

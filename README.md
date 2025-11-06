# enable orangepi zero 2w audio and gpu
(Thanks to Totof in https://dietpi.com/forum/t/no-sound-card-detected-for-opi-zero-2w/20133/89)
compile:
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

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

#install AHUB

# ===================== USER-CONFIGURABLE SECTION ==========================
# Define the modules you want. These are PATTERNS (e.g., "ahub", "v4l2", "r8152").
MODULE_PATTERNS=( "ahub" )

# Optional global hints to bias selection when multiple module names exist.
# You can freely add unrelated names; the script ranks them anyway.
MODULE_NAME_HINTS=( "snd-soc-sun8i-ahub" "snd-soc-sunxi-ahub" "sun8i-ahub" "sun50i-ahub" )

# Extra build tools for DKMS fallback:
BUILD_DEPS=( dkms build-essential bc flex bison libssl-dev libelf-dev dwarves )
# ==========================================================================

require_root() {
  if [[ ${EUID:-$(id -u)} -ne 0 ]]; then
    echo "[sudo] escalating privileges..."
    exec sudo -E "$0" "$@"
  fi
}

have() { command -v "$1" >/dev/null 2>&1; }
die()  { echo "Error: $*" >&2; exit 1; }

ask_choice() {
  # ask_choice "Prompt" [default=Y] -> prints: yes | next | cancel
  local prompt="$1" default="${2:-Y}" ans
  read -r -p "$prompt [Y]es/[n]ext/[c]ancel (default: ${default}) " ans || true
  ans=${ans:-$default}
  case "${ans,,}" in
    y|yes) echo "yes" ;;
    n|no|next) echo "next" ;;
    c|cancel|q|quit) echo "cancel" ;;
    *) echo "next" ;;
  esac
}

# --------- Pure-bash similarity: Levenshtein + prefix bonus + length gap -----
levenshtein() {
  local s1=$1 s2=$2
  local -i len1=${#s1} len2=${#s2} i j
  local -a prev curr
  local -i p=0 minlen=$(( len1 < len2 ? len1 : len2 ))

  # common-prefix length (bonus)
  for ((i=0;i<minlen;i++)); do
    [[ ${s1:i:1} == "${s2:i:1}" ]] || break
    ((p++))
  done

  for ((j=0;j<=len2;j++)); do prev[j]=j; done
  for ((i=1;i<=len1;i++)); do
    curr[0]=$i
    for ((j=1;j<=len2;j++)); do
      local cost
      if [[ ${s1:i-1:1} == "${s2:j-1:1}" ]]; then cost=0; else cost=1; fi
      local del=$(( prev[j] + 1 ))
      local ins=$(( curr[j-1] + 1 ))
      local sub=$(( prev[j-1] + cost ))
      local v=$del
      (( ins < v )) && v=$ins
      (( sub < v )) && v=$sub
      curr[j]=$v
    done
    prev=("${curr[@]}")
  done

  local -i dist=${prev[len2]}
  local -i len_gap=$(( len1 > len2 ? len1 - len2 : len2 - len1 ))
  echo $(( dist + (len_gap/2) - p ))  # smaller is better
}

rank_by_similarity() {
  # rank_by_similarity <target> <items...>
  local target="$1"; shift
  local item score
  for item in "$@"; do
    [[ -n "$item" ]] || continue
    score=$(levenshtein "$target" "$item")
    printf "%s\t%s\n" "$score" "$item"
  done | sort -n -k1,1 | cut -f2-
}

best_of_list() {
  local target="$1"; shift
  rank_by_similarity "$target" "$@" | head -n1
}

kernel_rel() { uname -r; }

hardware_signature() {
  local s="" a=""
  if [[ -r /proc/device-tree/model ]]; then
    s="$(tr -d '\0' </proc/device-tree/model 2>/dev/null || true)"
  fi
  for f in /sys/devices/virtual/dmi/id/{board_vendor,board_name,product_name,sys_vendor}; do
    [[ -r $f ]] && a+=" $(<"$f")"
  done
  a+=" $(uname -m) $(uname -s)"
  s="${s:-$a}"
  echo "${s,,}"  # lowercase fingerprint
}

# --------------------- Headers discovery & install --------------------------
find_header_pkgs() {
  mapfile -t _pkgs < <(apt-cache --names-only search '^linux-headers-' 2>/dev/null \
    | while read -r name _; do [[ $name == linux-headers-* ]] && echo "$name"; done)
  if ((${#_pkgs[@]}==0)); then
    for d in /usr/src/linux-headers-*; do [[ -d $d ]] && _pkgs+=("$(basename "$d")"); done
  fi
  ((${#_pkgs[@]})) || return 1
  printf "%s\n" "${_pkgs[@]}"
}

install_pkg_if_needed() {
  local pkg="$1"
  if dpkg -s "$pkg" >/dev/null 2>&1; then
    echo "[$pkg] already installed."
  else
    echo "Installing [$pkg]..."
    apt-get update -y
    DEBIAN_FRONTEND=noninteractive apt-get install -y "$pkg"
  fi
}

choose_header_pkg_iteratively() {
  local kr hdr candidates suffix ranked pkg
  kr="$(kernel_rel)"
  echo "Running kernel: $kr"
  echo "Collecting candidate linux-headers packages..."
  mapfile -t candidates < <(find_header_pkgs) || return 1

  # build suffix list
  mapfile -t suffix < <(for pkg in "${candidates[@]}"; do echo "${pkg#linux-headers-}"; done | uniq)

  # rank suffixes by similarity to uname -r, then map back to pkg names per step
  mapfile -t ranked < <(rank_by_similarity "$kr" "${suffix[@]}")

  local s
  for s in "${ranked[@]}"; do
    for pkg in "${candidates[@]}"; do
      if [[ ${pkg#linux-headers-} == "$s" ]]; then
        echo
        echo "Suggested headers package: $pkg"
        case "$(ask_choice 'Install this header?')" in
          yes)
            install_pkg_if_needed "$pkg"

            # Try modules-extra with same suffix if it exists
            local extra
            extra="linux-modules-extra-$s"
            if apt-cache policy "$extra" 2>/dev/null | grep -q 'Candidate:'; then
              install_pkg_if_needed "$extra" || true
            fi
            echo "$pkg"
            return 0
            ;;
          cancel)
            echo "Header selection cancelled by user."
            return 2
            ;;
          next) ;; # try next suggestion
        esac
      fi
    done
  done

  echo "No header selected. Skipping header installation."
  return 3
}

# --------------------- Module discovery / loading ---------------------------
list_module_files_for_pattern() {
  local pattern="$1"
  local kr; kr="$(kernel_rel)"
  find "/lib/modules/$kr" -type f -name "*${pattern,,}*.ko*" -printf "%f\n" 2>/dev/null | sort -u || true
}

basename_without_ko() {
  local base="${1##*/}"
  base="${base%.xz}"
  base="${base%.gz}"
  base="${base%.zst}"
  base="${base%.ko}"
  echo "$base"
}

module_loaded()  { lsmod | awk -v p="$1" 'BEGIN{IGNORECASE=1} $1~p{print; exit}' >/dev/null; }
module_present() { modinfo "$1" >/dev/null 2>&1; }

pick_module_names_ranked() {
  local pattern="$1"
  local hw; hw="$(hardware_signature)"
  local kr; kr="$(kernel_rel)"

  mapfile -t files < <(list_module_files_for_pattern "$pattern")
  local -a names=()
  local f
  for f in "${files[@]}"; do names+=( "$(basename_without_ko "$f")" ); done
  names+=( "${MODULE_NAME_HINTS[@]}" )

  # Dedup
  local -A seen=()
  local -a uniq=()
  for f in "${names[@]}"; do
    [[ -n $f && -z ${seen[$f]+x} ]] && { uniq+=("$f"); seen[$f]=1; }
  done
  ((${#uniq[@]})) || return 1

  local target="${hw} ${kr} ${pattern,,}"
  rank_by_similarity "$target" "${uniq[@]}"
}

any_loaded_matching() {
  local pattern="$1"
  local name
  name="$(lsmod | awk -v p="$pattern" 'BEGIN{IGNORECASE=1} $1~p{print $1; exit}')"
  [[ -n "$name" ]] && { echo "$name"; return 0; }
  return 1
}

load_or_install_module_iterative() {
  local pattern="$1"

  echo; echo "== Handling module pattern: '$pattern' =="
  # Early exit if something matching is already loaded
  if any_loaded_matching "$pattern" >/dev/null; then
    local loaded; loaded="$(any_loaded_matching "$pattern")"
    echo "A matching module is already loaded: $loaded"
    return 0
  fi

  local cand
  if mapfile -t cand < <(pick_module_names_ranked "$pattern"); then
    local name
    for name in "${cand[@]}"; do
      [[ -n "$name" ]] || continue
      echo
      echo "Suggested module: $name"
      case "$(ask_choice 'Use this module?')" in
        yes)
          if module_loaded "$name"; then
            echo "Module '$name' already loaded."
            return 0
          fi
          if module_present "$name"; then
            echo "Attempting to load '$name'..."
            if modprobe "$name"; then
              echo "Loaded '$name' successfully ✅"
              return 0
            else
              echo "modprobe failed for '$name'."
            fi
          else
            echo "Module '$name' not present on disk."
          fi
          ;;
        cancel)
          echo "Module selection cancelled by user."
          return 2
          ;;
        next) ;; # try next suggestion
      esac
    done
  fi

  echo "Falling back to DKMS build attempt for pattern '$pattern'..."
  dkms_build_module "$pattern"
}

# --------------------------- DKMS fallback (generic) ------------------------
choose_linux_source_tarball() {
  local -a tars=()
  shopt -s nullglob
  for t in /usr/src/linux-source-*.tar.*; do tars+=( "$(basename "$t")" ); done
  shopt -u nullglob
  ((${#tars[@]})) || return 1

  local kr; kr="$(kernel_rel)"
  best_of_list "$kr" "${tars[@]}"
}

ensure_linux_source() {
  if ! choose_linux_source_tarball >/dev/null 2>&1; then
    echo "No linux-source tarball found under /usr/src; trying to install..."
    apt-get update -y
    DEBIAN_FRONTEND=noninteractive apt-get install -y linux-source || true
  fi
  choose_linux_source_tarball || return 1
}

extract_linux_source_if_needed() {
  local tarball="$1"
  local base="${tarball%.tar.*}"
  local dir="/usr/src/${base}"
  if [[ ! -d "$dir" ]]; then
    echo "Extracting /usr/src/$tarball ..."
    case "$tarball" in
      *.tar.xz)  tar -C /usr/src -xf "/usr/src/$tarball" ;;
      *.tar.gz)  tar -C /usr/src -xzf "/usr/src/$tarball" ;;
      *.tar.zst) tar -C /usr/src --use-compress-program=unzstd -xf "/usr/src/$tarball" ;;
      *)         tar -C /usr/src -xf "/usr/src/$tarball" ;;
    esac
  fi
  echo "$dir"
}

dkms_build_module() {
  local pattern="$1"
  local kr; kr="$(kernel_rel)"

  echo "Installing build deps (if missing): ${BUILD_DEPS[*]}"
  apt-get update -y
  DEBIAN_FRONTEND=noninteractive apt-get install -y "${BUILD_DEPS[@]}"

  # Ensure headers (best-effort exact)
  install_pkg_if_needed "linux-headers-$(kernel_rel)" || true

  local tarball
  tarball="$(ensure_linux_source || true)"
  if [[ -z $tarball ]]; then
    echo "Could not get linux-source automatically. Please enable deb-src or install a suitable 'linux-source-*' package, then re-run."
    return 2
  fi

  local srcdir
  srcdir="$(extract_linux_source_if_needed "$tarball")"

  # Find a plausible driver source file generically (prefer common driver trees)
  local relc
  relc="$(cd "$srcdir" && { \
    find drivers sound net crypto fs -type f -iname "*${pattern}*.c" -print 2>/dev/null \
    | sort | head -n1; } || true)"
  if [[ -z $relc ]]; then
    echo "Could not locate a '*${pattern}*.c' inside $srcdir."
    return 2
  fi

  local drvdir="$srcdir/$(dirname "$relc")"
  local drvbase="$(basename "${relc%.c}")"
  local kmake="$drvdir/Makefile"

  # Try to infer module name from Makefile; else default to file basename
  local modko="$drvbase"
  if [[ -f $kmake ]]; then
    # 1) Look for '<mod>-objs := ... <drvbase>.o' or '<mod>-y :=' lines
    local line
    line="$(grep -E "^[[:alnum:]_+-]+-(objs|y)[[:space:]]*:[=].*\\b${drvbase}\\.o\\b" "$kmake" | head -n1 || true)"
    if [[ -n $line ]]; then
      modko="$(sed -E 's/^[[:space:]]*([[:alnum:]_+.-]+)-(objs|y).*/\1/i' <<<"$line")"
    else
      # 2) Try 'obj-$(CONFIG_...) += <mod>.o' lines that include basename
      line="$(grep -E "obj-[^+]*\\+=.*\\b${drvbase}\\.o\\b" "$kmake" | head -n1 || true)"
      if [[ -n $line ]]; then
        modko="$(sed -E 's/.*\+=\s*([[:alnum:]_+.-]+)\.o.*/\1/' <<<"$line")"
      fi
    fi
  fi

  local pkgname="${modko}-dkms"
  local pkgver="1.0"
  local dkms_root="/usr/src/${pkgname}-${pkgver}"

  echo "Preparing DKMS package: $pkgname $pkgver"
  rm -rf "$dkms_root"
  mkdir -p "$dkms_root/build"

  # Copy the driver directory (keeps local headers and Makefile if needed)
  (cd "$drvdir/.." && tar -cf - "$(basename "$drvdir")") | (cd "$dkms_root/build" && tar -xf -)

  # Minimal Kbuild (generic)
  cat >"$dkms_root/build/Makefile" <<EOF
# Auto-generated DKMS Kbuild
obj-m := ${modko}.o
${modko}-y := ${drvbase}.o
ccflags-y += -Wno-error
EOF

  # dkms.conf
  cat >"$dkms_root/dkms.conf" <<EOF
PACKAGE_NAME="${pkgname}"
PACKAGE_VERSION="${pkgver}"
BUILT_MODULE_NAME[0]="${modko}"
DEST_MODULE_LOCATION[0]="/kernel/extra"
AUTOINSTALL="yes"
MAKE[0]="make -C /lib/modules/\${kernelver}/build M=\${dkms_tree}/${pkgname}/${pkgver}/build modules"
CLEAN="make -C /lib/modules/\${kernelver}/build M=\${dkms_tree}/${pkgname}/${pkgver}/build clean"
EOF

  echo "Registering DKMS module..."
  dkms remove -m "$pkgname" -v "$pkgver" --all >/dev/null 2>&1 || true
  dkms add -m "$pkgname" -v "$pkgver"

  echo "Building for $kr ..."
  if dkms build -m "$pkgname" -v "$pkgver" -k "$kr"; then
    echo "Installing DKMS module..."
    dkms install -m "$pkgname" -v "$pkgver" -k "$kr"
    echo "Attempting to load '${modko}' ..."
    if modprobe "$modko"; then
      echo "Loaded '${modko}' successfully ✅"
      return 0
    else
      echo "DKMS build installed, but modprobe failed for '${modko}'."
      return 3
    fi
  else
    echo "DKMS build failed. You may need a source tree closer to your exact kernel or additional dependencies."
    return 4
  fi
}

# --------------------------------- MAIN -------------------------------------
main() {
  require_root "$@"

  # 1) Headers: iteratively suggest best candidates; allow next/cancel
  choose_header_pkg_iteratively || true

  # 2) For each requested module pattern: iteratively suggest best module names; allow next/cancel
  local pat
  for pat in "${MODULE_PATTERNS[@]}"; do
    load_or_install_module_iterative "$pat" || true
  done

  echo
  echo "All done. If a module didn't load:"
  echo "  - Ensure your kernel actually supports it (not built-in or disabled)"
  echo "  - Check dmesg for details"
  echo "  - Consider upgrading to a kernel flavor that ships the module"
}

main "$@"



# Install the device tree compiler
apt install device-tree-compiler
# Compile the overlay binary file
dtc -@ -I dts -O dtb -o sun50i-h616-audiogpu.dtbo sun50i-h616-audiogpu.dts

#copy to overlays:
sudo cp sun50i-h616-audiogpu.dtbo /boot/dtb/allwinner/overlay/

echo "FINALLY, you MUST ENABLE audiogpu OVERLAY ON KERNEL CONFIG!"

```

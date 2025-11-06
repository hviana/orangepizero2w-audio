# enable orangepi zero 2w audio and gpu
(Thanks to Totof in https://dietpi.com/forum/t/no-sound-card-detected-for-opi-zero-2w/20133/89)
compile:
```bash
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
#!/bin/bash
# Script to automatically install the ahub kernel module using DKMS on Debian
# V6: Uses a Pure BASH/AWK Weighted Similarity Score to find the best header match.

# --- Configuration ---
MODULE_NAME="ahub"
MODULE_VERSION="1.0"
SOURCE_DIR="$HOME/ahub-module-source" # *** REPLACE WITH ACTUAL PATH TO AHUB SOURCE ***
RUNNING_KERNEL="$(uname -r)"

# --- Check for Root Privileges ---
if [ "$EUID" -ne 0 ]; then
  echo "❌ Error: This script must be run as root."
  echo "   Please run with 'sudo ./install_ahub_smarter.sh'"
  exit 1
fi

echo "🚀 Starting automatic DKMS installation for the '$MODULE_NAME' module..."
echo "Running Kernel (uname -r): $RUNNING_KERNEL"
echo "---"

# --- 1. Install Dependencies ---
echo "⚙️  1. Ensuring necessary tools (dkms, build-essential) are installed..."
apt update
apt install -y dkms build-essential

# --- 2. AWK Function for Similarity Calculation (Weighted Match Score) ---
# This AWK script calculates a score based on character matches:
# Score = (Total Matching Characters / Total Characters in Longest String) * Positional Adjustment
# This is a proxy for Levenshtein that works within the shell environment.
AWK_SCRIPT='
BEGIN {
    FS="|"
    # Store the running kernel name for comparison
    target_name = ARGV[1];
    ARGV[1] = "" # Clear the target name from ARGV so awk doesn't treat it as a file
}

function calculate_score(package_name, target) {
    # 1. Normalize the package name (remove "linux-headers-")
    gsub("linux-headers-", "", package_name);

    len_pkg = length(package_name);
    len_target = length(target);
    len_max = (len_pkg > len_target) ? len_pkg : len_target;
    
    match_count = 0;
    
    # Simple weighted character match: +1 for character match, -0.1 for mismatch
    for (i = 1; i <= len_max; i++) {
        pkg_char = substr(package_name, i, 1);
        target_char = substr(target, i, 1);

        if (pkg_char == target_char) {
            match_count += 1;
        } else if (pkg_char != "" && index(target, pkg_char) > 0) {
            # Character exists but is misplaced: score partial credit
            match_count += 0.5;
        } else if (target_char != "" && index(package_name, target_char) > 0) {
            # Character exists but is misplaced: score partial credit
            match_count += 0.5;
        } else {
            # No match or misalignment (penalty equivalent)
            match_count -= 0.1;
        }
    }
    
    # Final Score: Normalized by length
    if (len_max > 0) {
        return (match_count / len_max);
    }
    return 0;
}

# Main loop to process packages fed via stdin
{
    package_name = $1;
    # Only process packages starting with 'linux-headers'
    if (package_name ~ /^linux-headers/) {
        score = calculate_score(package_name, target_name);
        if (score > max_score) {
            max_score = score;
            best_match = package_name;
        }
        
        # Print all scores for debugging (optional)
        # print package_name "|" score > "/dev/stderr"; 
    }
}

END {
    # Output the best match and score
    if (best_match != "") {
        printf "%s|%.4f\n", best_match, max_score;
    } else {
        printf "NO_MATCH|0.0\n";
    }
}'

# --- 3. Execute AWK Similarity Finder ---
echo "🔍 2. Searching repositories for best kernel header match..."
# 1. Get all linux-headers packages
# 2. Pipe the list into the AWK script for scoring
MATCH_OUTPUT=$(apt-cache search linux-headers 2>/dev/null | awk '{print $1}' | awk -v target_name="$RUNNING_KERNEL" "$AWK_SCRIPT" "$RUNNING_KERNEL")

BEST_HEADER_PACKAGE=$(echo "$MATCH_OUTPUT" | cut -d'|' -f1)
SIMILARITY_SCORE=$(echo "$MATCH_OUTPUT" | cut -d'|' -f2)

# --- 4. User Confirmation ---
if [ "$BEST_HEADER_PACKAGE" = "NO_MATCH" ] || (( $(echo "$SIMILARITY_SCORE < 0.3" | bc -l) )); then
    echo "❌ ERROR: No suitable linux-headers package found (Max Score: $SIMILARITY_SCORE)."
    echo "   Please check your APT configuration and repositories."
    exit 1
fi

echo "---"
echo "✅ BEST MATCH FOUND (based on weighted character match):"
echo "   Running Kernel: $RUNNING_KERNEL"
echo "   Package:        $BEST_HEADER_PACKAGE"
echo "   Similarity:     $(echo "scale=2; $SIMILARITY_SCORE * 100" | bc | cut -d'.' -f1,2)%"
echo "---"

read -r -p "Is this the correct header package to install? (Y/n): " CONFIRMATION

if [[ "$CONFIRMATION" =~ ^[Nn]$ ]]; then
    echo "Aborting at user request. Please run 'apt-cache search linux-headers' to find the correct package manually."
    exit 0
fi

# --- 5. Install Headers and Continue DKMS Process ---
echo "⚙️  3. Installing the confirmed header package: $BEST_HEADER_PACKAGE"
apt install -y "$BEST_HEADER_PACKAGE"

if [ $? -ne 0 ]; then
  echo "❌ Error: Failed to install the header package ($BEST_HEADER_PACKAGE). Aborting."
  exit 1
fi

# --- 6. DKMS Source Validation ---
echo "---"
echo "📂 4. Validating DKMS source directory and configuration..."
if [ ! -d "$SOURCE_DIR" ]; then
  echo "❌ Error: Module source directory not found at $SOURCE_DIR"
  exit 1
fi

if [ ! -f "$SOURCE_DIR/dkms.conf" ]; then
  echo "❌ Error: **dkms.conf** not found in the source directory. Aborting."
  echo "   (You must create this file for DKMS to work.)"
  exit 1
fi

# --- 7. DKMS Installation Steps ---
DKMS_TREE="/usr/src/$MODULE_NAME-$MODULE_VERSION"
echo "📂 5. Copying source to DKMS tree: $DKMS_TREE"
rm -rf "$DKMS_TREE"
mkdir -p "$DKMS_TREE"
cp -r "$SOURCE_DIR"/* "$DKMS_TREE"/

echo "➕ 6. Adding module to DKMS"
dkms add "$MODULE_NAME/$MODULE_VERSION"

echo "🔨 7. Building the kernel module"
dkms build "$MODULE_NAME/$MODULE_VERSION"
if [ $? -ne 0 ]; then
  echo "❌ Error: Failed to build the module. Check your module source/Makefile."
  exit 1
fi

echo "✅ 8. Installing the kernel module"
dkms install "$MODULE_NAME/$MODULE_VERSION"
if [ $? -ne 0 ]; then
  echo "❌ Error: Failed to install the module."
  exit 1
fi

# --- 8. Load Module and Final Checks ---
echo "🔌 9. Loading the module into the running kernel"
modprobe "$MODULE_NAME"

echo "🔍 10. Verifying installation..."
if lsmod | grep -q "$MODULE_NAME"; then
  echo "🎉 **SUCCESS!** The '$MODULE_NAME' module is loaded and installed via DKMS."
  echo "   It will be automatically rebuilt for future kernel updates."
else
  echo "⚠️  Warning: Module installation finished, but it is not currently loaded. Check 'dmesg'."
fi

echo "---"
echo "DKMS Status:"
dkms status "$MODULE_NAME"


# Install the device tree compiler
apt install device-tree-compiler
# Compile the overlay binary file
dtc -@ -I dts -O dtb -o sun50i-h616-audiogpu.dtbo sun50i-h616-audiogpu.dts

#copy to overlays:
sudo cp sun50i-h616-audiogpu.dtbo /boot/dtb/allwinner/overlay/

#FINALLY, ENABLE audiogpu OVERLAY ON KERNEL CONFIG!

```

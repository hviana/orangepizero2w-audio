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
# Detects correct linux-headers package name for custom/edge kernels.

# --- Configuration ---
MODULE_NAME="ahub"
MODULE_VERSION="1.0" # Change to your module's version
SOURCE_DIR="$HOME/ahub-module-source" # *** REPLACE WITH ACTUAL PATH TO AHUB SOURCE ***

# --- Check for Root Privileges ---
if [ "$EUID" -ne 0 ]; then
  echo "❌ Error: This script must be run as root."
  echo "   Please run with 'sudo ./install_ahub_dkms.sh'"
  exit 1
fi

echo "🚀 Starting automatic DKMS installation for the '$MODULE_NAME' module..."
echo "Module Source Directory: $SOURCE_DIR"
echo "Running Kernel: $(uname -r)"

# --- 1. Detect and Install Headers ---
echo "⚙️  1. Detecting and installing necessary dependencies (dkms, build-essential)..."
apt update
apt install -y dkms build-essential

# 1.1 Extract core kernel version and architecture
# Example: 6.16.8-edge-sunxi64 -> 6.16 (or 6.16.8), and -arm64 (or sunxi64)
CURRENT_KERNEL_VER=$(uname -r | cut -d'-' -f1) # Extracts 6.16.8 (or similar)
KERNEL_ARCH=$(dpkg --print-architecture)      # Extracts the architecture (e.g., arm64)

# 1.2 Search for the correct package name
echo "🔍 Searching for the correct linux-headers package for version $CURRENT_KERNEL_VER..."
# Searches for packages containing 'linux-headers' and the current major/minor version (e.g., 6.16)
HEADER_PACKAGE=$(apt-cache search "linux-headers-$CURRENT_KERNEL_VER" | grep "linux-headers" | grep "$KERNEL_ARCH" | awk '{print $1}' | head -n 1)

if [ -z "$HEADER_PACKAGE" ]; then
    echo "⚠️  Warning: Could not find an exact match ($CURRENT_KERNEL_VER-$KERNEL_ARCH)."
    # Fallback: Search just by major version (e.g., 6.16) and architecture, accepting the first result
    HEADER_PACKAGE=$(apt-cache search "linux-headers-$CURRENT_KERNEL_VER" | grep "linux-headers" | awk '{print $1}' | head -n 1)
fi

if [ -z "$HEADER_PACKAGE" ]; then
    echo "❌ Error: Failed to find any suitable 'linux-headers' package in the repositories."
    echo "   Please manually verify the package name in 'apt-cache search linux-headers' and run the install command."
    exit 1
fi

echo "✅ Found header package: **$HEADER_PACKAGE**"
echo "   Installing headers..."

# 1.3 Install the detected headers
apt install -y "$HEADER_PACKAGE"

if [ $? -ne 0 ]; then
  echo "❌ Error: Failed to install the detected headers package ($HEADER_PACKAGE). Aborting."
  exit 1
fi

# --- 2. Check Source Directory and dkms.conf ---
if [ ! -d "$SOURCE_DIR" ]; then
  echo "❌ Error: Module source directory not found at $SOURCE_DIR"
  echo "   Please ensure the path is correct and the source code (including dkms.conf) is present."
  exit 1
fi

if [ ! -f "$SOURCE_DIR/dkms.conf" ]; then
  echo "❌ Error: **dkms.conf** not found in the source directory."
  echo "   You must create a dkms.conf file in $SOURCE_DIR for DKMS to work. Aborting."
  exit 1
fi

# --- 3. Copy Source to DKMS Tree ---
DKMS_TREE="/usr/src/$MODULE_NAME-$MODULE_VERSION"
echo "📂 2. Copying source to DKMS tree: $DKMS_TREE"
rm -rf "$DKMS_TREE" # Clean up previous attempts/versions
mkdir -p "$DKMS_TREE"
cp -r "$SOURCE_DIR"/* "$DKMS_TREE"/

# --- 4. Add Module to DKMS ---
echo "➕ 3. Adding module to DKMS"
dkms add "$MODULE_NAME/$MODULE_VERSION"

# --- 5. Build Module ---
echo "🔨 4. Building the kernel module"
dkms build "$MODULE_NAME/$MODULE_VERSION"

if [ $? -ne 0 ]; then
  echo "❌ Error: Failed to build the module. Check your module source and Makefile."
  exit 1
fi

# --- 6. Install Module ---
echo "✅ 5. Installing the kernel module"
dkms install "$MODULE_NAME/$MODULE_VERSION"

if [ $? -ne 0 ]; then
  echo "❌ Error: Failed to install the module. Aborting."
  exit 1
fi

# --- 7. Load Module and Final Checks ---
echo "🔌 6. Loading the module into the running kernel"
modprobe "$MODULE_NAME"

echo "🔍 7. Verifying installation..."
if lsmod | grep -q "$MODULE_NAME"; then
  echo "🎉 **SUCCESS!** The '$MODULE_NAME' module is loaded and installed via DKMS."
  echo "   It will be automatically rebuilt for future kernel updates."
else
  echo "⚠️  Warning: Module installation finished, but it is not currently loaded."
  echo "   Check 'dmesg' for potential errors."
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

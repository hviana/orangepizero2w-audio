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
# V4: Generic, multi-stage intelligence for finding linux-headers package.

# --- Configuration ---
MODULE_NAME="ahub"
MODULE_VERSION="1.0" # Change to your module's version
SOURCE_DIR="$HOME/ahub-module-source" # *** REPLACE WITH ACTUAL PATH TO AHUB SOURCE ***

# --- Check for Root Privileges ---
if [ "$EUID" -ne 0 ]; then
  echo "❌ Error: This script must be run as root."
  echo "   Please run with sudo"
  exit 1
fi

echo "🚀 Starting automatic DKMS installation for the '$MODULE_NAME' module..."
echo "Module Source Directory: $SOURCE_DIR"
RUNNING_KERNEL="$(uname -r)"
KERNEL_ARCH="$(dpkg --print-architecture)"
echo "Running Kernel: $RUNNING_KERNEL"
echo "System Architecture: $KERNEL_ARCH"

# --- 1. Install Dependencies and Headers ---
echo "⚙️  1. Installing necessary dependencies (dkms, build-essential)..."
apt update
apt install -y dkms build-essential

# 1.1 Define a function to try installing headers
install_headers() {
    HEADER_PACKAGE="$1"
    echo "Attempting to install headers: $HEADER_PACKAGE"
    # Check if the package exists before trying to install (saves time)
    if apt-cache policy "$HEADER_PACKAGE" | grep -q 'Candidate'; then
        apt install -y "$HEADER_PACKAGE"
        return $?
    else
        echo "   Package '$HEADER_PACKAGE' not found in repositories. Skipping."
        return 1
    fi
}

HEADER_INSTALLED=false

# --- Stage 1: Exact uname -r match (Most specific, least likely to work on custom kernels) ---
echo "🔎 STAGE 1: Checking for exact package name match (e.g., linux-headers-$RUNNING_KERNEL)..."
if install_headers "linux-headers-$RUNNING_KERNEL"; then
    echo "✅ Success in Stage 1."
    HEADER_INSTALLED=true
fi

# --- Stage 2: Dynamic apt-cache search (Best generic attempt) ---
if [ "$HEADER_INSTALLED" = false ]; then
    echo "🔎 STAGE 2: Using dynamic apt-cache search for major.minor version and arch..."
    # Extracts major.minor version (e.g., 6.16) from 6.16.8-edge-sunxi64
    CURRENT_KERNEL_VER=$(echo "$RUNNING_KERNEL" | cut -d'-' -f1 | cut -d'.' -f1,2) 

    echo "   Searching for: 'linux-headers', version $CURRENT_KERNEL_VER, and architecture $KERNEL_ARCH..."
    
    # Search for any package containing 'linux-headers', the major.minor version, and the architecture
    HEADER_PACKAGE=$(apt-cache search "linux-headers" | grep "linux-headers" | grep "$CURRENT_KERNEL_VER" | grep "$KERNEL_ARCH" | awk '{print $1}' | head -n 1)

    if [ ! -z "$HEADER_PACKAGE" ]; then
        echo "   Found header package: **$HEADER_PACKAGE**"
        if install_headers "$HEADER_PACKAGE"; then
            echo "✅ Success in Stage 2."
            HEADER_INSTALLED=true
        fi
    fi
fi

# --- Stage 3: Generic Architecture Meta-package (Ultimate fallback) ---
if [ "$HEADER_INSTALLED" = false ]; then
    GENERIC_PACKAGE="linux-headers-$KERNEL_ARCH"
    echo "🔎 STAGE 3: Falling back to generic architecture meta-package ($GENERIC_PACKAGE)..."
    
    if apt-cache search "^$GENERIC_PACKAGE$" | grep -q "$GENERIC_PACKAGE"; then
        if install_headers "$GENERIC_PACKAGE"; then
            echo "✅ Success in Stage 3."
            HEADER_INSTALLED=true
        fi
    fi
fi

# --- Final Check for Headers Installation ---
if [ "$HEADER_INSTALLED" = false ]; then
    echo "❌ **FATAL ERROR:** Failed to find and install any suitable 'linux-headers' package."
    echo "   **Action Required:** You must manually find the correct package name using:"
    echo "   'apt-cache search linux-headers | less'"
    exit 1
fi

# --- 2. DKMS Source Validation ---
echo "---"
echo "📂 2. Validating DKMS source directory and configuration..."
if [ ! -d "$SOURCE_DIR" ]; then
  echo "❌ Error: Module source directory not found at $SOURCE_DIR"
  exit 1
fi

if [ ! -f "$SOURCE_DIR/dkms.conf" ]; then
  echo "❌ Error: **dkms.conf** not found in the source directory. Aborting."
  exit 1
fi

# --- 3. DKMS Installation Steps ---
DKMS_TREE="/usr/src/$MODULE_NAME-$MODULE_VERSION"
echo "📂 3. Copying source to DKMS tree: $DKMS_TREE"
rm -rf "$DKMS_TREE"
mkdir -p "$DKMS_TREE"
cp -r "$SOURCE_DIR"/* "$DKMS_TREE"/

echo "➕ 4. Adding module to DKMS"
dkms add "$MODULE_NAME/$MODULE_VERSION"

echo "🔨 5. Building the kernel module"
dkms build "$MODULE_NAME/$MODULE_VERSION"
if [ $? -ne 0 ]; then
  echo "❌ Error: Failed to build the module. Check your module source/Makefile."
  exit 1
fi

echo "✅ 6. Installing the kernel module"
dkms install "$MODULE_NAME/$MODULE_VERSION"
if [ $? -ne 0 ]; then
  echo "❌ Error: Failed to install the module."
  exit 1
fi

# --- 7. Load Module and Final Checks ---
echo "🔌 7. Loading the module into the running kernel"
modprobe "$MODULE_NAME"

echo "🔍 8. Verifying installation..."
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

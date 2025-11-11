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
sudo apt install -y supertux frozen-bubble freedoom chromium-bsu lbreakout2
```
```bash
sudo apt install flatpak
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak install flathub org.xonotic.Xonotic
```
PS4 joystick fix xonotic
```bash
mkdir -p ~/.var/app/org.xonotic.Xonotic/.xonotic/data
cat > ~/.var/app/org.xonotic.Xonotic/.xonotic/data/autoexec.cfg <<'EOF'
// --- Xonotic: PS4/DS4 joystick fix (Linux) ---
// Enable joystick and pick the first detected pad (js0)
joy_enable "1"
joy_index "0"

// Left stick: move/strafe
joy_axisside    "0"   // left stick X -> strafe
joy_axisforward "1"   // left stick Y -> forward/back

// No vertical movement on sticks
joy_axisup      "-1"  // disable

// RIGHT STICK → CAMERA (Profile A: common on Linux)
joy_axisyaw     "3"   // right stick X -> look left/right
joy_axispitch   "4"   // right stick Y -> look up/down

// If camera still doesn’t move with the right stick, try Profile B instead:
// joy_axisyaw   "2"
// joy_axispitch "5"

// Deadzones to stop drift/auto-rotation (tweak if needed)
joy_deadzoneyaw     "0.20"
// Kill slow downward drift on right-stick (camera pitch)
joy_deadzonepitch "0.40"
joy_sensitivitypitch "0.90"
joy_deadzoneside    "0.12"
joy_deadzoneforward "0.12"

// Sensitivity (sign controls direction)
joy_sensitivityyaw   "-1.0"  // keep negative so “stick right = turn right”
joy_sensitivitypitch  "1.0"  // set to -1.0 if you prefer inverted look

// Don’t also emit arrow-key events from axes (prevents weird doubles)
joy_axiskeyevents "0"

// Optional quick switchers (use from console: `joy_profile_a` / `joy_profile_b`)
alias joy_profile_a "joy_axisyaw 3; joy_axispitch 4"
alias joy_profile_b "joy_axisyaw 2; joy_axispitch 5"

// Log
echo "🔧 Loaded joystick fix from autoexec.cfg"
EOF

```
# Enable **Allwinner SoC Audio support V2** options in Armbian compilation with (*) - not working yet:

~~~
Device Drivers
  → Sound card support
    → Advanced Linux Sound Architecture
      → ALSA for SoC audio support
        → Allwinner SoC Audio support V2
			→ Allwinner AAUDIO support -> if you use analog audio
			→ Allwinner AHUB Support -> if you use hdmi/digital audio

~~~
# ps4/5 joystick
~~~
Device Drivers
  HID bus support ──►
    [*] HIDRAW support                      (CONFIG_HIDRAW)
    Special HID drivers ──►
      <M> PlayStation HID Driver            (CONFIG_HID_PLAYSTATION) # here is the key!

Input device support ──►
  [*] Joystick interface                    (CONFIG_INPUT_JOYDEV)
  
Device Drivers ──►
  [*] User-space I/O driver support ──►
      [*] User level driver support (UHID)  (CONFIG_UHID)
~~~
```bash
printf "hid_playstation\njoydev\nuhid\nhidraw\n" | sudo tee /etc/modules-load.d/sony.conf
```
if BRLTTY conflict:
```bash
sudo apt purge -y brltty
```

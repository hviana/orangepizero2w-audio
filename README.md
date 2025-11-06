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

# Raspberry Pi HiFi Streamer
### AirPlay 2 streaming to a hi-fi amplifier using Raspberry Pi 4 + Raspberry Pi DAC Pro

Turn a Raspberry Pi into a standalone AirPlay 2 receiver that streams Apple Music and other lossless audio to any amplifier with a line-level input. No laptop required after setup - just plug in and play.

---

## My setup

| Component | Model |
|---|---|
| Single board computer | Raspberry Pi 4 Model B |
| DAC hat | Raspberry Pi DAC Pro (PCM5242, 32-bit/384kHz) |
| Amplifier | Rega Brio |
| Connection | RCA phono cable (DAC Pro → Brio Aux/CD input) |
| Primary streaming | AirPlay 2 (Apple Music, lossless) |

---

## What you need

**Hardware**
- Raspberry Pi 4 Model B (2GB+ RAM)
- Raspberry Pi DAC Pro hat
- USB-C power supply - 5V/3A minimum (official Pi PSU recommended)
- RCA phono cable (male-to-male stereo)
- MicroSD card (16GB+, Class 10)
- Mac or Windows PC for initial setup

**Software (all free)**
- Raspberry Pi Imager
- Raspberry Pi OS Lite (64-bit, Bookworm)
- shairport-sync (AirPlay 2 receiver)

---

## Signal chain

```
iPhone / Mac  ──(AirPlay 2 over Wi-Fi)──►  Raspberry Pi 4
                                                  │
                                            40-pin GPIO
                                                  │
                                           DAC Pro hat
                                         (PCM5242 chip)
                                                  │
                                           RCA phono cable
                                                  │
                                        Amplifier line input
                                                  │
                                              Speakers
```

---

## Step 1 - Assemble the hardware

1. Attach the DAC Pro hat to the Pi's 40-pin GPIO header before first boot. Press firmly and evenly and use the supplied standoffs to secure the board.
2. Connect the RCA cable from the DAC Pro's phono outputs to a line-level input on your amplifier (Aux or CD input - **not** the phono input, which is for turntables only).
3. Insert the microSD card, and connect the USB-C power supply to the wall.

> ⚠️ Connect the Pi to a wall socket - not your laptop. Laptops don't supply enough power through USB-C.

---

## Step 2 - Flash Raspberry Pi OS

1. Download and install [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
2. Choose **Raspberry Pi OS Lite (64-bit)**
3. Click the **gear/settings icon** and configure:
   - Hostname: `hifistreamer`
   - Enable SSH: ✅
   - Username: `pi`
   - Password: *(something you'll remember)*
   - Configure Wi-Fi: your home network SSID and password *(case-sensitive)*
   - Wi-Fi country: your country code (e.g. `GB`)
4. Flash to the microSD card and insert it into the Pi

---

## Step 3 - First boot and SSH in

Power on the Pi and wait 60–90 seconds, then on your Mac:

```bash
ssh pi@hifistreamer.local
```

Type `yes` when asked about the fingerprint, then enter your password. You'll know you're on the Pi when the prompt shows `pi@hifistreamer`.

Update the system:

```bash
sudo apt update && sudo apt full-upgrade -y
```

---

## Step 4 - Configure the DAC Pro

The Raspberry Pi DAC Pro needs its own overlay and I2C enabled. Edit the boot config:

```bash
sudo nano /boot/firmware/config.txt
```

Replace the entire contents with the following (this is the clean working config):

```
dtparam=i2c_arm=on
dtparam=audio=off
camera_auto_detect=1
display_auto_detect=1
auto_initramfs=1
#dtoverlay=vc4-kms-v3d
max_framebuffers=2
disable_fw_kms_setup=1
arm_64bit=1
disable_overscan=1
arm_boost=1
dtoverlay=rpi-dacpro

[cm4]
otg_mode=1

[cm5]
dtoverlay=dwc2,dr_mode=host

[all]
enable_uart=1
```

> ⚠️ **Important:** Use `dtoverlay=rpi-dacpro` for the official Raspberry Pi DAC Pro. Do NOT use `hifiberry-dacplus` - that is for a different board and will not work.

Make i2c-dev load at boot:

```bash
echo "i2c-dev" | sudo tee -a /etc/modules
```

Install i2c tools (useful for diagnosing hardware):

```bash
sudo apt install -y i2c-tools
```

Reboot:

```bash
sudo reboot
```

---

## Step 5 - Verify the DAC is detected

SSH back in, then run:

```bash
aplay -l
```

You should see:

```
card 0: Pro [RPi DAC Pro], device 0: Raspberry Pi DAC Pro HiFi pcm512x-hifi-0
```

If you see this, the DAC is working. If not, check the troubleshooting section below.

---

## Step 6 - Set the volume

```bash
alsamixer
```

You should see `RPi DAC Pro` at the top. Use the arrow keys to set `Analogue` and `Digital` to 100. Then press `Esc` and save:

```bash
sudo alsactl store
```

> Set digital volume to 100% and control loudness only from your amplifier's volume knob - this gives the best audio quality.

Test audio is working:

```bash
speaker-test -c 2 -t sine -f 1000
```

You should hear a 1kHz tone through your speakers. Press `Ctrl+C` to stop.

---

## Step 7 - Install AirPlay 2

```bash
sudo apt install -y shairport-sync
```

Configure it:

```bash
sudo nano /etc/shairport-sync.conf
```

Set the following (find and edit the relevant sections):

```
general = {
  name = "HiFi Streamer";
};

alsa = {
  output_device = "hw:0";
};
```

Enable and start the service:

```bash
sudo systemctl enable shairport-sync
sudo systemctl restart shairport-sync
```

---

## Step 8 - Test AirPlay

1. Open Apple Music on your iPhone or Mac
2. Tap the AirPlay icon on the now playing screen
3. Select **HiFi Streamer**
4. Turn up your amplifier volume and enjoy

---

## Daily use

- **No laptop needed** after setup - the Pi runs completely independently
- **To start:** plug in power, wait 60–90 seconds, HiFi Streamer appears in AirPlay
- **To stop safely:** SSH in and run `sudo poweroff`, wait for the green LED to stop blinking, then unplug. In practice, unplugging directly is usually fine for a streaming device.
- **If unplugged by mistake:** just plug back in - everything starts automatically on boot

---

## Troubleshooting

### DAC not detected (`aplay -l` shows no HiFiBerry/DAC card)

Check the overlay is correct:
```bash
grep dtoverlay /boot/firmware/config.txt
```
Should show `dtoverlay=rpi-dacpro`.

Check I2C can see the chip:
```bash
sudo i2cdetect -y 1
```
You should see `UU` at position `4c`. If the grid is empty, the hat may not be seated properly - power off, reseat the hat firmly, and try again.

Check dmesg for errors:
```bash
dmesg | grep -i pcm512
```

### AirPlay appears but no sound

Check shairport-sync is running:
```bash
sudo systemctl status shairport-sync
```

Check the output device in `/etc/shairport-sync.conf` is set to `hw:0`.

Make sure your amplifier input selector is set to the input the RCA cable is plugged into (Aux or CD - not Phono).

### Common mistakes

- Using `hifiberry-dacplus` overlay instead of `rpi-dacpro` - these are different boards
- Plugging RCA into the amplifier's Phono input - this applies RIAA equalisation for turntables and will sound wrong
- `dtparam=audio=on` left enabled - this conflicts with the DAC and must be set to `off`
- Not rebooting after config changes - always reboot after editing `/boot/firmware/config.txt`

---

## Acknowledgements

Built with a lot of trial and error. The key breakthrough was identifying that the official Raspberry Pi DAC Pro requires the `rpi-dacpro` overlay, not the commonly documented HiFiBerry overlays.

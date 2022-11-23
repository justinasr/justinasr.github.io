# Part 1: Setup

## Common packages

Install `build-essential`, `autoconf` and `automake`:

```bash
sudo apt install build-essential autoconf automake
```

## Install devkitPro

Taken from official instructions: https://devkitpro.org/wiki/devkitPro_pacman.

To install devkitPro, another package manager - devkitPro-pacman - is needed. To install devkitPro pacman:

```bash
wget https://apt.devkitpro.org/install-devkitpro-pacman
chmod +x ./install-devkitpro-pacman
sudo ./install-devkitpro-pacman
```

Contents of script:

```bash
#!/usr/bin/env bash
if ! [ $(id -u) = 0 ]; then
  echo "Need root privilege to install!"
  exit 1
fi

# ensure apt is set up to work with https sources
apt-get install apt-transport-https

# Store devkitPro gpg key locally if we don't have it already
if ! [ -f /usr/local/share/keyring/devkitpro-pub.gpg ]; then
  mkdir -p /usr/local/share/keyring/
  wget -O /usr/local/share/keyring/devkitpro-pub.gpg https://apt.devkitpro.org/devkitpro-pub.gpg
fi

# Add the devkitPro apt repository if we don't have it set up already
if ! [ -f /etc/apt/sources.list.d/devkitpro.list ]; then
  echo "deb [signed-by=/usr/local/share/keyring/devkitpro-pub.gpg] https://apt.devkitpro.org stable main" > /etc/apt/sources.list.d/devkitpro.list
fi

# Finally install devkitPro pacman
apt-get update
apt-get install devkitpro-pacman
```

Once installed, devkitpro-pacman should be available as `dkp-pacman` command.

## GBA tools

To install GBA tools via `dkp-pacman`:

```bash
sudo dkp-pacman -S gba-dev
```

Once installed, reboot system with `sudo reboot`.

## mGBA emulator

mGBA is available via "Software" software installer.

## References:

- Making GBA Games on Linux! - https://www.youtube.com/watch?v=8v3Ci_KvMNM


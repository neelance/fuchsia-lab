# Fuchsia Lab on MacOS

## Setup

Set project and zone:
```bash
gcloud config set project my-test-project
gcloud compute project-info add-metadata --metadata google-compute-default-region=europe-west1,google-compute-default-zone=europe-west1-b
gcloud init
```

Create instance:
```bash
gcloud compute instances create fuchsia-lab \
  --machine-type=c3-highcpu-8 \
  --network-interface=network=default,network-tier=PREMIUM \
  --create-disk=boot=yes,image=projects/ubuntu-os-cloud/global/images/ubuntu-2304-lunar-amd64-v20230829,size=50 \
  --enable-nested-virtualization
```

Connect and enable KVM:
```bash
gcloud compute ssh fuchsia-lab
sudo usermod -a -G kvm ${USER}
exit

gcloud compute ssh fuchsia-lab
if [[ -w /dev/kvm ]] && grep '^flags' /proc/cpuinfo | grep -qE 'vmx|svm'; then echo 'KVM is working'; else echo 'KVM not working'; fi
```

Install tools:
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y apt-transport-https gpg wget xrdp

# install VSCode tunnel
curl -L 'https://code.visualstudio.com/sha/download?build=stable&os=cli-alpine-x64' | sudo tar -xzC /usr/local/bin

# install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# install Fuchsia SDK
git clone https://fuchsia.googlesource.com/sdk-samples/getting-started fuchsia-getting-started --recurse-submodules
cd fuchsia-getting-started
scripts/bootstrap.sh
tools/bazel build @fuchsia_sdk//:fuchsia_toolchain_sdk
tools/ffx sdk version
tools/ffx product-bundle get workstation_eng.qemu-x64 --force-repo --repository workbench-packages
```

## Run emulator

Start Fuchsia emulator and repository server:
```bash
tools/ffx emu start workstation_eng.qemu-x64 --headless
tools/ffx target default set fuchsia-emulator
tools/ffx target show

tools/ffx repository server start
tools/ffx target repository register -r workbench-packages --alias fuchsia.com --alias chromium.org
```

Inspect running Fuchsia instance:
```bash
tools/ffx component list
```

## Connect via VSCode

Enable VS Code tunnel:
```bash
sudo loginctl enable-linger $USER
code tunnel service install
```

Use "Remote Tunnels: Connect to Tunnel..." action locally.

## Graphics

Enable Tailscale tunnel:
```bash
sudo tailscale up
```

Set user password:
```bash
sudo passwd $USER
```

Use [Microsoft Remote Desktop](https://apps.apple.com/us/app/microsoft-remote-desktop/id1295203466) to connect. There will be no window manager, but that's okay.

Start emulator in your SSH session (not the remote desktop):
```bash
export DISPLAY=:10.0
tools/ffx emu start workstation_eng.qemu-x64 --gpu swiftshader_indirect
```

You should now see the graphical interface of the emulator and you should even be able to run Chromium in Fuchsia.
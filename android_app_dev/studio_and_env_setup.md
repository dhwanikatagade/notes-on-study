---
layout: default
---
# Setup studio and environment
- Setup JDK for stand alone builds
  - ```bash
    sudo apt install openjdk-21-jdk
    ```
  - Setup an environment file for JDK
    - ```bash
      export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64/
      ```
- Install Android Studio
  - Download it from Google's official [Android Studio website](https://developer.android.com/studio/install)
    - ```bash
      $ sudo tar -xvf <archive>.tar.gz -C /opt/
      $ sudo chown -R root:root /opt/android-studio/
      ```
  - Launch android studio
    - ```bash
      $ /opt/android-studio/bin/studio
      ```
    - On the studio window, click on Tools->Create Desktop Entry
    - Alternatively on the welcome splash screen, click the Options Gear Icon and click Create Desktop Entry
  - On the first launch, accept the defaults to download and install the SDK components
- Install KVM virtualization infrastructure for Linux
  - This is used by the Android Device Emulator for running virtual machines
  - Check CPU support for virtualization
    - ```bash
      $ lscpu | grep -E 'vmx|svm'
      ```
  - Install QEMU KVM binding, libvirt, and networking tools
    - ```bash
      $ sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils
      $ sudo systemctl enable --now libvirtd
      $ sudo adduser $USER kvm
      $ sudo adduser $USER libvirt
      ```


### References:
1. [Install Android Studio](https://developer.android.com/studio/install)
1. [How to add Android Studio to the launcher?](https://askubuntu.com/questions/298857/how-to-add-android-studio-to-the-launcher)
1. [Configure hardware acceleration for the Android Emulator](https://developer.android.com/studio/run/emulator-acceleration)
1. [Install KVM on Ubuntu 24.04](https://dev.to/rosgluk/install-kvm-on-ubuntu-2404-n06)


# Zeek Installation and Configuration Procedure (Ubuntu 24.04 Server Edition)

This document describes the procedures for installing and configuring the Zeek network monitoring framework on Ubuntu 24.04 Server, and for conducting operation checks by obtaining communication logs for EtherNet/IP, Modbus, and BACnet.

## 1. Preparation of VirtualBox and Ubuntu 24.04 Server

### 1.1 Software Download
1. **Download and Install VirtualBox**
   - Download the installer for Windows host from the [Official VirtualBox Website](https://www.virtualbox.org/) and install it.
2. **Obtain Ubuntu 24.04 LTS Server ISO Image**
   - Download the `Ubuntu 24.04.x LTS Server` ISO image (.iso file) from the [Official Ubuntu Download Page](https://ubuntu.com/download/server).

### 1.2 Import Ubuntu 24.04 Server
1. Launch VirtualBox and click "New".
2. Set the Name (`MonitorServer Ubuntu 24.04 Server`), ISO Image (select the previously downloaded Ubuntu ISO), check "Skip Unattended Installation (Recommended)", and click "Next".
3. **Hardware**:
   - Memory (RAM): Recommended **4096 MB** (4GB) or more
   - Processors: Recommended **2 CPU** or more
4. **Hard Disk**:
   - Create a Virtual Hard Disk: Allocate **40 GB** or more and go to "Next" -> "Finish".

### 1.3 Ubuntu 24.04 Server Adapter Settings (via GUI)
Allocate adapters from the VirtualBox GUI.
(This example assumes connection to intnet7.1. Modify according to your environment.)

1. On the main VirtualBox screen, select the target VM (`MonitorServer Ubuntu 24.04 Server`), and open "Settings" > "Network".
2. Open the **"Adapter 1"** tab and check "Enable Network Adapter".
   - **Attached to:** `Internal Network`
   - **Name:** `intnet7.1`
3. Open the **"Adapter 2"** tab and check "Enable Network Adapter".
   - **Attached to:** `NAT`
   - Port Forwarding
     Name: SSH, Protocol: TCP, Host Port: 7132, Guest Port: 22
   *Port forwarding is used for internet connection and SSH access from the host OS. Enable only during development.*
4. Click "OK" to save.

### 1.4 Assigning DHCP Server to the Internal Network (intnet7.1)
Setting up the DHCP server for internal networks (`intnet7.1`, `intnet7.2`, `intnet7.3`) should be done in each environment.

<Reference> For `intnet7.1`, addresses from `10.7.1.128` to `10.7.1.191` are assigned via DHCP. The range from 10.7.1.3 to 10.7.1.127 is reserved for static IP assignment. The following command should have been executed during the setup of each environment:

```powershell
.\VBoxManage dhcpserver add --netname intnet7.1 --ip 10.7.1.2 --netmask 255.255.255.0 --lowerip 10.7.1.128 --upperip 10.7.1.191 --enable
```

### 1.5 Ubuntu 24.04 Server Installation
1. "Start" the virtual machine.
2. Select `English` for the language. (For the Server edition, installation and operation in English rather than Japanese is strongly recommended.)
3. Follow the on-screen instructions to configure the keyboard layout (Japanese), etc.
4. **Network connections**: Leave as default and select `Done`.
5. **Storage configuration**: Leave as default and select `Done`.
6. **Profile setup**: Set up username (e.g., `kali`), password, etc.
7. **SSH Setup**: **Make sure to press the spacebar to check ([X]) "Install OpenSSH server".**
8. **Featured Server Snaps**: Since Docker will be installed from the official repository later, leave everything unchecked here and select `Done`.
9. Once installation is complete, select `Reboot Now`. (*If "Please remove the installation medium" appears, press Enter to reboot.*)

### 1.6 SSH Server Auto-Start Configuration
Configure SSH to start automatically when the OS boots. This ensures SSH is enabled upon subsequent VM startups.

```bash
# Enable auto-start
sudo systemctl enable ssh
# Start the service
sudo systemctl start ssh
# Check status
sudo systemctl status ssh
```

### 1.7 Static IP Address Configuration for MonitorServer
Configure the static IP address for the MonitorServer for each protocol environment.
See sections 4.1, 5.1, and 6.1.

### 1.8 Timezone Configuration (Asia/Tokyo)
Set the timezone to `Asia/Tokyo` to align the log timestamps with Japan Standard Time (JST).

```bash
sudo timedatectl set-timezone Asia/Tokyo
```

---
## 2. Zeek Installation

Zeek is installed using the official package provided via the openSUSE Build Service.

### 2.1 Add the Repository's GPG Key
```bash
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
```

### 2.2 Add the Repository
```bash
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list
```

### 2.3 Execute Installation
```bash
sudo apt update
sudo apt install zeek
```
> **Note**: During installation, a Postfix (mail configuration) prompt may appear. If using this purely as a monitoring sensor, select "No configuration" or "Local only" to proceed.

## 3. Basic Startup Check and Plugin Preparation

### 3.1 Interface Check
Identify the interface to be monitored (e.g., `eth0`, `enp0s3`).

```bash
ip addr show
```

### 3.2 Start Zeek
Start Zeek by specifying the interface to monitor.

```bash
sudo /opt/zeek/bin/zeek -i <interface_name> -C local
```
`local` specifies the standard set of communication scripts. The scripts are located in `/opt/zeek/share/zeek/scripts/`.
> **Note**: In a virtual machine (VM) environment, TCP/UDP/IP checksums often do not have correct values. Therefore, add the `-C` option to ignore checksum validation by Zeek.

### 3.3 Install Necessary Development Modules (Libraries)
To monitor control system communication protocols like ENIP, dedicated plugins must be introduced in addition to the standard Zeek installation to enable detailed analysis and log output.
Install the C/C++ compiler and Zeek development header files required to compile Zeek plugins.

```bash
sudo apt update
sudo apt install cmake make gcc g++ zeek-core-dev zeek-spicy-dev
```

### 3.4 Initialize ZKG (Zeek Package Manager)
When using the package manager for the first time, auto-generate the configuration file.

```bash
sudo /opt/zeek/bin/zkg autoconfig
```

---

## 4. EtherNet/IP Log Acquisition Procedure

When monitoring EtherNet/IP (ENIP/CIP) communications, introduce the dedicated plugin for detailed analysis.

### 4.1 Configure MonitorServer IP Address (for ENIP Environment)
Adapt the MonitorServer's IP to the ENIP environment (10.7.1.0/24). Edit `/etc/netplan/99-netcfg.yaml` as follows:

```yaml
    enp0s3:
      dhcp4: false
      addresses:
        - 10.7.1.32/24
```
Then, apply the changes.

```bash
sudo netplan apply
```

### 4.2 Install EtherNet/IP Analysis Plugin
Install the ENIP plugin provided by CISA.

```bash
sudo /opt/zeek/bin/zkg install https://github.com/cisagov/icsnpp-enip
```
- If an error occurs during the test and you need to force the installation, use the `--skiptests` option.

### 4.3 EtherNet/IP Operation Check Procedure

**[Step 1] Start Monitoring with Zeek (Guest OS)**
Enable promiscuous mode on the target interface, start Zeek, and load the ENIP plugin.

```bash
sudo ip link set dev enp0s3 promisc on
cd ~/zeek_logs
rm -rf *.log
sudo /opt/zeek/bin/zeek -i enp0s3 -C local icsnpp-enip
```

**[Step 2] Confirm Zeek Log Output (Zeek Guest OS)**
After communication occurs, verify that the following logs have been newly created in `~/zeek_logs`.
- `enip.log`
- `enip_list_identity.log` (If List Identity communication occurred)
- `cip.log` (If CIP communication is included over ENIP)

If these files exist and their contents are recorded, it indicates that EtherNet/IP communications are correctly recognized.

---

## 5. Modbus Log Acquisition Procedure

Zeek supports basic analysis of the Modbus TCP protocol by default, but for detailed monitoring (function codes, register read/write, etc.), introduce the dedicated plugin provided by CISA.

### 5.1 Configure MonitorServer IP Address (for Modbus Environment)
Change the MonitorServer's IP address to match the Modbus environment (10.7.2.0/24). On Ubuntu 24.04 Server edition, edit `/etc/netplan/99-netcfg.yaml`.

```yaml
    enp0s3:
      dhcp4: false
      addresses:
        - 10.7.2.32/24
```
After making the changes, apply the settings with the following command and confirm they are reflected.

```bash
sudo netplan apply
ip addr show
```

### 5.2 Install Modbus Plugin
Install the Modbus plugin provided by CISA.

```bash
sudo /opt/zeek/bin/zkg install https://github.com/cisagov/icsnpp-modbus
```
- If an error occurs during the test and you need to force the installation, use the `--skiptests` option.

### 5.3 Modbus Operation Check Procedure
Link Zeek with each Modbus component, test communication analysis, and log output.

**[Step 1] Start Modbus Server (Simulator)**
On the guest OS where the Modbus server is running, open the Modbus TCP port (default: 502) and put it in a listening state. In the `modbus-lab` environment, executing `sudo docker compose up -d` automatically launches the server/client and generates regular communication.

**[Step 2] Generate Communication from Modbus Client**
Send requests from another guest OS to the Modbus server to generate communication events. You can confirm the actual communication (port 502) with the following command. Once confirmed, stop tcpdump with Ctrl-C.

```bash
sudo tcpdump -i enp0s3 -n -A tcp port 502
```

**[Step 3] Start Monitoring with Zeek (Guest OS)**
Enable promiscuous mode on the monitored interface, start Zeek, and load the Modbus plugin.

```bash
sudo ip link set dev enp0s3 promisc on
cd ~/zeek_logs
rm -rf *.log
sudo /opt/zeek/bin/zeek -i enp0s3 -C local icsnpp-modbus
```

**[Step 4] Confirm Zeek Log Output (Zeek Guest OS)**
Confirm that the following logs are generated in `~/zeek_logs`.
- `modbus.log`
- `modbus_detailed.log`

---

## 6. BACnet Log Acquisition Procedure

### 6.1 Configure MonitorServer IP Address (for BACnet Environment)
Change the MonitorServer's IP address to match the BACnet environment (10.7.3.0/24). On Ubuntu 24.04 Server edition, edit `/etc/netplan/99-netcfg.yaml`.

```yaml
    enp0s3:
      dhcp4: false
      addresses:
        - 10.7.3.32/24
```
After making the changes, apply the settings with the following command and confirm they are reflected.

```bash
sudo netplan apply
ip addr show
```

### 6.2 Install BACnet Plugin
Download, build, and install the BACnet plugin provided by CISA.

```bash
sudo /opt/zeek/bin/zkg install https://github.com/cisagov/icsnpp-bacnet
```
- If an error occurs during the test and you need to force the installation, use the `--skiptests` option.

### 6.3 BACnet Operation Check Procedure
Link the Zeek and BACnet components (server, client) within the virtual network, generate actual communication, and confirm log output.

> **Note**: Modify the network interface name `enp0s3` and the bacnet-stack-0.8.2 installation path `~/bacnet/bacnet-stack-0.8.2` mentioned in the steps below as appropriate for your actual environment.

**[Step 1] Execute on BACnet Server (Guest OS)**
Execute the following commands on the guest OS where the server runs, putting the BACnet server in a listening state with device ID `1234`.

```bash
export BACNET_IFACE=enp0s3
~/bacnet/bacnet-stack-0.8.2/bin/bacserv 1234
```

**[Step 2] Start Monitoring with Zeek (Guest OS)**
Enable promiscuous mode on the guest OS where Zeek is running, delete any old logs in the specified directory, and then start monitoring.

```bash
sudo ip link set dev enp0s3 promisc on
cd ~/zeek_logs
rm -rf *.log
sudo /opt/zeek/bin/zeek -i enp0s3 -C local icsnpp-bacnet
```

**[Step 3] Generate Communication from BACnet Client (Guest OS)**
From a guest OS used for the client, execute the following commands to discover BACnet devices on the network (Who-Is broadcast) and generate communication.

```bash
export BACNET_IFACE=enp0s3
~/bacnet/bacnet-stack-0.8.2/bin/bacwi -1
```

**[Step 4] Confirm Zeek Log Output (Zeek Guest OS)**
After generating communication, verify that the following log files have been newly created in the `~/zeek_logs` directory.
- `bacnet.log`
- `bacnet_discovery.log`

If these files exist, Zeek has successfully detected and analyzed the BACnet protocol, indicating that the setup is complete.

---

## 7. Advanced Settings

### 7.1 Concurrent Monitoring of Multiple Protocols
If you wish to monitor multiple protocols simultaneously on the same network interface (e.g., `enp0s3`), specify the plugin names separated by half-width spaces in the Zeek startup command.

```bash
sudo /opt/zeek/bin/zeek -i enp0s3 -C local icsnpp-enip icsnpp-modbus icsnpp-bacnet
```
With this command, the respective logs (`enip.log`, `modbus.log`, `bacnet.log`, etc.) will be generated simultaneously when communication for each protocol occurs.

## 8. Supplementary Information

### 8.1 Effects of Checksum Offloading and the `-C` Option
In virtual environments or with certain physical NICs, TCP/UDP/IP checksums may be captured as "0" or with incorrect values. This is due to the OS's "Checksum Offloading" feature, which delegates checksum calculations to the hardware (NIC).

1. **Offloading Feature**: To reduce calculation load, the OS delegates checksum calculation to the NIC.
2. **Capture Timing**: Since Zeek intercepts the packet before it reaches the NIC (before calculation), it is detected as an error.
3. **Zeek Behavior**: By default, packets with invalid checksums are discarded.

Given that this phenomenon is particularly notable in VirtualBox internal networks and similar environments, it is recommended to add the `-C` option to skip checksum validation and forcibly perform the analysis.

### 8.2 Zeek internally sets the monitored interface to promiscuous mode
Therefore, it is unnecessary to set promiscuous mode manually for the monitored interface before running Zeek.

### 8.3 Troubleshooting During Installation
During package installation (`zkg install`), the automated tests for plugins may result in an error (`Fail`). Even in such cases, appending the `--skiptests` option to force installation often results in the plugin working normally in practice.

### 8.4 Troubleshooting During Zeek Execution
If Zeek log files are not being generated, confirm whether communications can be captured using the following tcpdump command.

```bash
sudo tcpdump -i <interface_name> -n port <port_number>
```

---
End of Document

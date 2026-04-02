# Zeek Installation and Configuration Procedure (Ubuntu 24.04 Server Edition)

This document describes the procedures for installing and configuring the Zeek network monitoring framework on Ubuntu 24.04 Server, and for conducting operation checks by obtaining communication logs for EtherNet/IP, Modbus, and BACnet.

## 1. Zeek Installation

Zeek is installed using the official package provided via the openSUSE Build Service.

### 1-1. Add the Repository's GPG Key
```bash
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
```

### 1-2. Add the Repository
```bash
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list
```

### 1-3. Execute Installation
```bash
sudo apt update
sudo apt install zeek
```
> **Note**: During installation, a Postfix (mail configuration) prompt may appear. If using this purely as a monitoring sensor, select "No configuration" or "Local only" to proceed.

## 2. Basic Startup Check and Plugin Preparation

### 2-1. Interface Check
Identify the interface to be monitored (e.g., `eth0`, `enp0s3`).
```bash
ip addr show
```

### 2-2. Start Zeek
Start Zeek by specifying the interface to monitor.
```bash
sudo /opt/zeek/bin/zeek -i <interface_name> -C local
```
`local` specifies the standard set of communication scripts. The scripts are located in `/opt/zeek/share/zeek/scripts/`.
> **Note**: In a Virtual Machine (VM) environment, TCP/UDP/IP checksums often do not have correct values. Therefore, add the `-C` option to ignore checksum validation by Zeek.

### 2-3. Install Necessary Development Modules (Libraries)
To monitor control system communication protocols like ENIP, dedicated plugins must be introduced in addition to the standard Zeek installation to enable detailed analysis and log output.
Install the C/C++ compiler and Zeek development header files required to compile Zeek plugins.
```bash
sudo apt update
sudo apt install cmake make gcc g++ zeek-core-dev zeek-spicy-dev
```

### 2-4. Initialize ZKG (Zeek Package Manager)
When using the package manager for the first time, auto-generate the configuration file.
```bash
sudo /opt/zeek/bin/zkg autoconfig
```

---

## 3. EtherNet/IP Log Acquisition Procedure

When monitoring EtherNet/IP (ENIP/CIP) communications, introduce the dedicated plugin for detailed analysis.

### 3-1. Change MonitorServer IP Address (for ENIP Environment)
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

### 3-2. Install EtherNet/IP Analysis Plugin
Install the ENIP plugin provided by CISA.
```bash
sudo /opt/zeek/bin/zkg install https://github.com/cisagov/icsnpp-enip
```
- If an error occurs during the test and you need to force the installation, use the `--skiptests` option.

### 3-3. EtherNet/IP Operation Check Procedure
Link Zeek with the ENIP components and confirm the output of analysis logs.

**[Step 1] Start EtherNet/IP Server (Device) and Check Connectivity**
On the EtherNet/IP server side, open ports 44818 (TCP/UDP) and 2222 (UDP) and put them in a listening state.
For `enip-lab`, execute `sudo docker-compose up -d` in the `enip-lab` directory.

Confirm from MonitorServer that Pings reach the monitored devices (for enip-lab: dockerhost 10.7.1.36, enip-plc 10.7.1.37, enip-mes 10.7.1.38).
```bash
ping 10.7.1.36
ping 10.7.1.37
ping 10.7.1.38
```

**[Step 2] Generate Communication from EtherNet/IP Client**
For `enip-lab`, execute `PLC_IP=10.7.1.37 python robot_sim.py`.
Confirm that EtherNet/IP packets are being captured using the following tcpdump command. Once confirmed, stop tcpdump with Ctrl-C.

```bash
sudo tcpdump -i enp0s3 -n -A port 44818 or port 2222
```

**[Step 3] Start Monitoring with Zeek (Guest OS)**
Enable promiscuous mode on the target interface, start Zeek, and load the ENIP plugin.
```bash
sudo ip link set dev enp0s3 promisc on
cd ~/zeek_logs
rm -rf *.log
sudo /opt/zeek/bin/zeek -i enp0s3 -C local icsnpp-enip
```

**[Step 4] Confirm Zeek Log Output (Zeek Guest OS)**
After generating communication, verify that the following logs have been newly created in `~/zeek_logs`.
- `enip.log`
- `enip_list_identity.log` (If List Identity communication occurred)
- `cip.log` (If CIP communication is included over ENIP)

If these files exist and their contents are recorded, it indicates that EtherNet/IP communications are correctly recognized.

---

## 4. Modbus Log Acquisition Procedure

Zeek supports basic analysis of the Modbus TCP protocol by default, but for detailed monitoring (function codes, register read/write, etc.), introduce the dedicated plugin provided by CISA.

### 4-1. Change MonitorServer IP Address (for Modbus Environment)
Change the MonitorServer's IP address to match the Modbus environment (10.7.2.0/24). On Ubuntu 24.04 Server edition, edit `/etc/netplan/99-netcfg.yaml`.
```yaml
    enp0s3:
      dhcp4: false
      addresses:
        - 10.7.2.32/24
```
After making the changes, apply the settings with the following commands and confirm they are reflected.
```bash
sudo netplan apply
ip addr show
```

### 4-2. Install Modbus Plugin
Install the Modbus plugin provided by CISA.
```bash
sudo /opt/zeek/bin/zkg install https://github.com/cisagov/icsnpp-modbus
```
- If an error occurs during the test and you need to force the installation, use the `--skiptests` option.

### 4-3. Modbus Operation Check Procedure
Link Zeek with each Modbus component, test communication analysis, and log output.

**[Step 1] Start Modbus Server (Simulator)**
On the guest OS where the Modbus server is running, open the Modbus TCP port (default: 502) and put it in a listening state. In the `modbus-lab` environment, starting `docker-compose.yml` automatically launches the server/client and generates regular communication.

**[Step 2] Generate Communication from Modbus Client**
Send requests from another guest OS (etc.) to the Modbus server to generate communication events. You can confirm the actual communication (port 502) with the following command. Once confirmed, stop tcpdump with Ctrl-C.
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

## 5. BACnet Log Acquisition Procedure

### 5-1. Install BACnet Plugin
Download, build, and install the BACnet plugin provided by CISA.
```bash
sudo /opt/zeek/bin/zkg install https://github.com/cisagov/icsnpp-bacnet
```
- If an error occurs during the test and you need to force the installation, use the `--skiptests` option.

### 5-2. BACnet Operation Check Procedure
Link the Zeek and BACnet components (server, client) within the virtual network, generate actual communication, and confirm log output.

> **Note**: Modify the network interface name `enp0s3` and the bacnet-stack-0.8.2 installation path `~/bacnet/bacnet-stack-0.8.2` mentioned in the steps below as appropriate for your actual environment.

**[Step 1] Execute on BACnet Server (Guest OS)**
Execute the following commands on the guest OS where the server acts, putting the BACnet server in a listening state with device ID `1234`.
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

## 6. Advanced Settings

### 6-1. Concurrent Monitoring of Multiple Protocols
If you wish to monitor multiple protocols simultaneously on the same network interface (e.g., `enp0s3`), specify the plugin names separated by half-width spaces in the Zeek startup command.
```bash
sudo /opt/zeek/bin/zeek -i enp0s3 -C local icsnpp-enip icsnpp-modbus icsnpp-bacnet
```
With this command, the respective logs (`enip.log`, `modbus.log`, `bacnet.log`, etc.) will be generated simultaneously when communication for each protocol occurs.

## Supplementary Information

### 6-2. Effects of Checksum Offloading and the `-C` Option
In virtual environments or with certain physical NICs, TCP/UDP/IP checksums may be captured as "0" or with incorrect values. This is due to the OS's "Checksum Offloading" feature, which delegates checksum calculations to the hardware (NIC).

1. **Offloading Feature**: To reduce calculation load, the OS delegates checksum calculation to the NIC.
2. **Capture Timing**: Since Zeek intercepts the packet before it reaches the NIC (before calculation), it is detected as an error.
3. **Zeek Behavior**: By default, packets with invalid checksums are discarded.

Given that this phenomenon is particularly notable in VirtualBox internal networks and similar environments, it is recommended to add the `-C` option to skip checksum validation and forcibly perform the analysis.

### 6-3. Troubleshooting During Installation
During package installation (`zkg install`), the automated tests for plugins may result in an error (`Fail`). Even in such cases, appending the `--skiptests` option to force installation often results in the plugin working normally in practice.

### 6-4. Troubleshooting During Zeek Execution
If Zeek log files are not being generated, confirm whether communications can be captured using the following tcpdump command.

```bash
sudo tcpdump -i <interface_name> -n port <port_number>
```

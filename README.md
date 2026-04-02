# Monitor Server (Zeek Environment Setup)

This repository contains the installation, configuration, and operation check procedures for setting up a network monitoring server using [Zeek](https://zeek.org/) on Ubuntu 24.04 Server.

The documentation specifically focuses on capturing and analyzing Industrial Control Systems (ICS) and Operational Technology (OT) network protocols such as:
- **EtherNet/IP** (ENIP/CIP)
- **Modbus TCP**
- **BACnet**

## 📄 Documentation

We provide step-by-step guides in both Japanese and English. Please refer to the document of your preferred language:

* 🇬🇧 **English**: [Zeek_Installation_Configuration_and_Operation_Check_Procedure_EN](Zeek_Installation_Configuration_and_Operation_Check_Procedure_EN.md)
* 🇯🇵 **Japanese**: [モニターサーバ(Zeek)のインストール・設定・動作確認手順書](モニターサーバ(Zeek)のインストール・設定・動作確認手順書.md)

## 🔧 Environment & Prerequisites

* **OS**: Ubuntu 24.04 Server
* **Network**: A network interface configured to capture the target traffic (requires promiscuous mode).
* **Dependencies**: C/C++ build tools (`cmake`, `make`, `gcc`, `g++`), Zeek core headers, and ZKG (Zeek Package Manager).

## 🚀 Key Topics Covered in the Guides

1. **Zeek Core Installation**: How to install the Zeek framework using the official openSUSE repository.
2. **ICS Plugin Installation**: Adding CISA-provided `icsnpp` plugins (for ENIP, Modbus, BACnet) using the Zeek Package Manager.
3. **Verification**: How to launch protocol simulators, generate communication, and verify that Zeek outputs the correct logs (e.g., `enip.log`, `modbus.log`, `bacnet.log`).

---

## 🇯🇵 リポジトリ概要 (Japanese Overview)

本リポジトリは、Ubuntu 24.04 Server をベースとした **Zeek ネットワーク監視サーバ** の構築および動作確認手順を管理・共有するためのものです。
主に、産業用制御システム（ICS）向けの通信プロトコル（EtherNet/IP, Modbus, BACnet）の詳細なログを取得・解析するためのプラグイン導入手順に焦点を当てています。
各言語のMarkdownドキュメントを参照して構築を進めてください。

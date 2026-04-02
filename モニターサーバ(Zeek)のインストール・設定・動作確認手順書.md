# Zeek インストールおよび設定手順書 (Ubuntu 24.04 Server版)

本書では、Ubuntu 24.04 Server に対してネットワーク監視フレームワーク Zeek をインストールおよび設定し、動作確認としてEtherNet/IP、Modbus、BACnetの通信ログを取得する手順を記載する。

## 1. Zeekのインストール

Zeekは、openSUSE Build Service 経由で提供される公式パッケージを利用する。

### 1-1. リポジトリのGPGキー追加
```bash
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
```

### 1-2. リポジトリの追加
```bash
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list
```

### 1-3. インストールの実行
```bash
sudo apt update
sudo apt install zeek
```
> **注意**: インストール中に Postfix (メール設定) のプロンプトが表示される場合がある。監視専用のセンサーとして利用する場合は、「No configuration（設定なし）」または「Local only（ローカルのみ）」を選択して進める。

## 2. 基本的な起動確認とプラグインの準備

### 2-1. インターフェースの確認
監視するインターフェース（例: `eth0`, `enp0s3`など）を確認する。
```bash
ip addr show
```

### 2-2. Zeekの起動
監視するインターフェースを指定してZeekを起動する。
```bash
sudo /opt/zeek/bin/zeek -i <interface_name> -C local
```
`local` は標準的な通信用のスクリプト群の指定を意味する。スクリプト群は `/opt/zeek/share/zeek/scripts/` に格納されている。
> **注意**: 仮想マシン(VM)環境ではTCP/UDP/IPのチェックサムが正しい値にならない。そこで、Zeekによるチェックサムのバリデーションを無視する `-C` オプションを追加する。

### 2-3. 必要な開発モジュール（ライブラリ）のインストール
ENIP等の制御システム向け通信プロトコルは、Zeekの標準インストールに加えて専用のプラグインを導入することで、詳細な解析やログの出力が可能になる。
Zeek用プラグインのコンパイルに必要な C/C++ コンパイラや、Zeekの開発用ヘッダーファイルをインストールする。
```bash
sudo apt update
sudo apt install cmake make gcc g++ zeek-core-dev zeek-spicy-dev
```

### 2-4. ZKG（Zeek Package Manager）の初期設定
初めてパッケージマネージャーを利用する場合は、設定ファイルを自動生成する。
```bash
sudo /opt/zeek/bin/zkg autoconfig
```

---

## 3. EtherNet/IPのログ取得手順

EtherNet/IP（ENIP/CIP）通信を監視する場合、専用のプラグインを導入して詳細な解析を行う。

### 3-1. MonitorServerのIPアドレスの変更（ENIP環境用）
MonitorServerのIPをENIP環境（10.7.1.0/24）に適応させる。`/etc/netplan/99-netcfg.yaml` を以下の通り変更する。
```yaml
    enp0s3:
      dhcp4: false
      addresses:
        - 10.7.1.32/24
```
その後、変更を適用する。
```bash
sudo netplan apply
```

### 3-2. EtherNet/IP解析プラグインのインストール
CISA提供のENIPプラグインをインストールする。
```bash
sudo /opt/zeek/bin/zkg install https://github.com/cisagov/icsnpp-enip
```
- testでエラーが発生しインストールを強制する場合は `--skiptests` オプションを使用する。

### 3-3. EtherNet/IP動作確認手順
ZeekとENIPコンポーネントを連携させ、解析ログの出力を確認する。

**【手順1】EtherNet/IPサーバ（デバイス）の起動と導通確認**
EtherNet/IPサーバ側でポート 44818 (TCP/UDP) および 2222 (UDP) を開放して待機させる。
enip-labの場合、enip-labディレクトリで sudo docker-compose up -d を実行する。

MonitorServerから監視対象デバイス（enip-labの場合、dockerhost 10.7.1.36, enip-plc 10.7.1.37, enip-mes 10.7.1.38）へPingが届くことを確認する。
```bash
ping 10.7.1.36
ping 10.7.1.37
ping 10.7.1.38
```

**【手順2】EtherNet/IPクライアントによる通信発生**
enip-labの場合、PLC_IP=10.7.1.37 python robot_sim.pyを実行する。
EtherNet/IPのパケットがキャプチャできていることを以下のtcpdumpコマンドで確認する。確認したらCtl-Cでtcpdumpを停止する。

```bash
sudo tcpdump -i enp0s3 -n -A port 44818 or port 2222
```

**【手順3】Zeek（ゲストOS）での監視開始**
対象インターフェースでプロミスキャスモードを有効にし、Zeekを起動してENIPプラグインを読み込ませる。
```bash
sudo ip link set dev enp0s3 promisc on
cd ~/zeek_logs
rm -rf *.log
sudo /opt/zeek/bin/zeek -i enp0s3 -C local icsnpp-enip
```

**【手順4】Zeekログの出力確認（ZeekゲストOS）**
通信発生後、`~/zeek_logs` 内に以下のログが新しく生成されていることを確認する。
- `enip.log`
- `enip_list_identity.log` （List Identity通信が行われた場合）
- `cip.log` （ENIP上でCIP通信が含まれている場合）

これらのファイルが存在し、内容が記録されていればEtherNet/IP通信が正しく認識できている状態である。

---

## 4. Modbusのログ取得手順

Zeekは標準でModbus TCPプロトコルの基本的な解析に対応しているが、詳細な監視（機能コードやレジスタの読み書き等）を行う場合は、CISA提供の専用プラグインを導入する。

### 4-1. MonitorServerのIPアドレスの変更（Modbus環境用）
MonitorServerのIPアドレスをModbus環境（10.7.2.0/24）に合わせて変更する。Ubuntu 24.04 Server版では、`/etc/netplan/99-netcfg.yaml` を編集する。
```yaml
    enp0s3:
      dhcp4: false
      addresses:
        - 10.7.2.32/24
```
変更後、以下のコマンドで設定を適用し、反映を確認する。
```bash
sudo netplan apply
ip addr show
```

### 4-2. Modbusプラグインのインストール
CISA提供のModbusプラグインをインストールする。
```bash
sudo /opt/zeek/bin/zkg install https://github.com/cisagov/icsnpp-modbus
```
- testでエラーが発生しインストールを強制する場合は `--skiptests` オプションを使用する。

### 4-3. Modbus動作確認手順
ZeekとModbus各コンポーネントを連携させ、通信解析とログ出力をテストする。

**【手順1】Modbusサーバ（シミュレータ）の起動**
Modbusサーバが稼働するゲストOSで、Modbus TCPポート（デフォルト: 502）を開放して待機させる。`modbus-lab` 環境では、`docker-compose.yml` を起動することでサーバ/クライアントが自動起動し、定期的な通信が発生する。

**【手順2】Modbusクライアントからの通信発生**
別のゲストOS等からModbusサーバへリクエストを送信し、通信イベントを発生させる。以下のコマンドで実際の通信（ポート502）を併せて確認できる。確認したらCtl-Cでtcpdumpを停止する。
```bash
sudo tcpdump -i enp0s3 -n -A tcp port 502
```

**【手順3】Zeek（ゲストOS）での監視開始**
監視対象インターフェースでプロミスキャスモードを有効にし、Zeekを起動してModbusプラグインを読み込ませる。
```bash
sudo ip link set dev enp0s3 promisc on
cd ~/zeek_logs
rm -rf *.log
sudo /opt/zeek/bin/zeek -i enp0s3 -C local icsnpp-modbus
```

**【手順4】Zeekログの出力確認（ZeekゲストOS）**
`~/zeek_logs` 内に以下のログが生成されていることを確認する。
- `modbus.log`
- `modbus_detailed.log`

---

## 5. BACnetのログ取得手順

### 5-1. BACnetプラグインのインストール
CISAが提供している BACnet プラグインをダウンロードし、ビルド・インストールする。
```bash
sudo /opt/zeek/bin/zkg install https://github.com/cisagov/icsnpp-bacnet
```
- testでエラーが発生しインストールを強制する場合は `--skiptests` オプションを使用する。

### 5-2. BACnet動作確認手順
ZeekとBACnetの各コンポーネント（サーバ、クライアント）を仮想ネットワーク内で連携させ、実際に通信を発生させてログの出力を確認する。

> **注意**: 以下の手順に記載されているネットワークインターフェース名 `enp0s3`、 およびbacnet-stack-0.8.2のインストールパス名 `~/bacnet/bacnet-stack-0.8.2` は、適宜実環境に合わせて変更する。

**【手順1】BACnetサーバ（ゲストOS）での実行**
サーバが稼働するゲストOSで以下のコマンドを実行し、BACnetサーバをデバイスID `1234` で待機させる。
```bash
export BACNET_IFACE=enp0s3
~/bacnet/bacnet-stack-0.8.2/bin/bacserv 1234
```

**【手順2】Zeek（ゲストOS）での監視開始**
Zeekが稼働するゲストOSでプロミスキャスモードを有効にし、指定ディレクトリ内に古いログがあれば削除してから監視をスタートさせる。
```bash
sudo ip link set dev enp0s3 promisc on
cd ~/zeek_logs
rm -rf *.log
sudo /opt/zeek/bin/zeek -i enp0s3 -C local icsnpp-bacnet
```

**【手順3】BACnetクライアント（ゲストOS）での通信発生**
クライアント用のゲストOSから以下のコマンドを実行し、ネットワーク内のBACnetデバイスを探索（Who-Isブロードキャスト）して通信を発生させる。
```bash
export BACNET_IFACE=enp0s3
~/bacnet/bacnet-stack-0.8.2/bin/bacwi -1
```

**【手順4】Zeekログの出力確認（ZeekゲストOS）**
通信発生後、`~/zeek_logs` ディレクトリ内に、以下のログファイルが新しく生成されていることを確認する。
- `bacnet.log`
- `bacnet_discovery.log`

これらのファイルが存在すれば、Zeekが正常にBACnetプロトコルを検知・解析し、構築が完了していることになる。


---

## 6. 応用設定

### 6-1. 複数プロトコルの同時監視について
同一のネットワークインターフェース（例: `enp0s3`）で、複数のプロトコルを同時に監視させたい場合は、Zeekの起動コマンドでプラグイン名を半角スペース区切りで指定する。
```bash
sudo /opt/zeek/bin/zeek -i enp0s3 -C local icsnpp-enip icsnpp-modbus icsnpp-bacnet
```
このコマンドにより、各プロトコルの通信が発生した際に、それぞれ対応するログ（`enip.log`, `modbus.log`, `bacnet.log` 等）が同時に生成される。

## 補足

### 6-2. チェックサム・オフロードの影響と `-C` オプションについて
仮想環境や一部の物理NIC環境では、TCP/UDP/IPのチェックサムが「0」または不正確な値のままキャプチャされることがある。これはOSが計算をハードウェア（NIC）に任せる「チェックサム・オフロード」機能によるものである。

1. **オフロード機能**: OSは計算負荷を減らすため、チェックサム計算をNICに任せる。
2. **キャプチャのタイミング**: ZeekがNICに届く前の（未計算の）パケットを横取りするため、エラーとして検出される。
3. **Zeekの挙動**: デフォルトでは不正なチェックサムを持つパケットを破棄する。

特にVirtualBoxの内部ネットワーク等ではこの現象が顕著であるため、`-C` オプションを付与してチェックサム検証をスキップし、強制的に解析を行うアプローチが推奨される。

### 6-3. インストール時のトラブルシューティング
パッケージのインストール（zkg install）中にプラグインの自動テストでエラー（Fail）となる場合がある。その場合でも、`--skiptests` オプションを付けて強制的にインストールすることで、実際には正常に動作することが多い。

### 6-4. Zeek実行時のトラブルシューティング
Zeekでlogファイルが生成されない場合、通信がキャプチャできるかどうかを以下のtcpdumpコマンドで確認する。

```bash
sudo tcpdump -i <インターフェース名> -n port <ポート番号>
```
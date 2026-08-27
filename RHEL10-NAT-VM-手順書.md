# RHEL 10 NAT VM 構築手順

このホストは上流回線が Wi-Fi のみのため、ブリッジ接続は物理的に成立しません。
libvirt の NAT ネットワーク `default` 上に RHEL 10 ゲストを DVD ISO からテキストモードで構築します。
管理は CLI（`virsh` / `virt-install`）のみ。

## ホスト情報（実測値）

| 項目 | 値 |
|---|---|
| Host OS | RHEL 10.2 (Coughlan) / kernel 6.12.0-211.49.1.el10_2 |
| CPU | AMD Ryzen 5 3600 / 6 コア 12 スレッド |
| KVM | `kvm_amd` ロード済み、`/dev/kvm` あり |
| Memory | 31 GiB（空き 27 GiB） |
| Uplink | `wlp7s0f3u1` (Wi-Fi) 192.168.1.24/24 → GW 192.168.1.1 |
| `enp4s0` | リンクダウン（ケーブル未接続） |
| `/` 空き | 63 GB |
| `/home` 空き | 816 GB（別 xfs マウント） |
| SELinux | Enforcing |
| firewalld | active / default zone `public`（`wlp7s0f3u1` が所属） |
| 操作ユーザー | `redhat`（wheel 所属、libvirt グループは未所属） |

---

## NAT で何ができて、何ができないか

libvirt の `default` ネットワークは、ホスト上に `virbr0` という内部専用ブリッジを作り、
`192.168.122.0/24` を dnsmasq で配ります。ゲストから外へ出る通信は nftables の
masquerade でホストの IP に書き換えられて Wi-Fi から出ていきます。

```
 ┌──────────────┐  vnet0    ┌──────────────┐ nftables  ┌──────────────┐          ┌──────────────┐
 │  ゲスト VM   │ virtio-net│    virbr0    │ masquerade│  wlp7s0f3u1  │          │  ルーター    │
 │192.168.122.x │ ─────────▶│192.168.122.1 │ ─────────▶│ 192.168.1.24 │ ────────▶│ 192.168.1.1  │
 │  DHCP 払出し │           │dnsmasq DHCP  │  送信元IPを│  Wi-Fi (STA) │          │ → インターネット│
 └──────────────┘           └──────────────┘  .1.24に変換└──────────────┘          └──────────────┘
        ▲                                                                                 │
        └─────────────────────────────── ✕ ───────────────────────────────────────────────┘
            外部 → VM の着信は届かない（ポート転送を足せば個別に開通）
```

NAT は片方向です。ゲストの外向き通信はホストの IP に変換されて成立しますが、
LAN 上の他の機器からゲストの `192.168.122.x` へは到達できません。

### この構成で諦めること

- ゲストが LAN の DHCP（192.168.1.0/24）から IP を取ることはできません。
- LAN 上の他機から VM へ直接アクセスできません（ポート転送で個別に開通は可能）。
- ゲスト発の mDNS / ブロードキャストは LAN に届きません。

これらが必要になったら、`enp4s0` に LAN ケーブルを挿してブリッジ `br0` を作る構成へ
切り替えてください（Wi-Fi は STA モードのため 802.11 の 3 アドレスフレーム制約で
ブリッジできません。macvtap も同じ理由で不可）。

---

## 作業用の変数

以降のコマンドはこの変数を前提にしています。**シェル変数は新しいシェルに引き継がれません。**
特に手順 3 の `newgrp libvirt` は新しいシェルを起動するため、そこで定義が消えます。
毎回書き直さずに済むよう、ファイルに置いて `source` する形にします。

```bash
mkdir -p ~/.config
cat > ~/.config/rhel10-vm.env <<'EOF'
VM_NAME=rhel10-vm
VM_RAM=8192          # MiB
VM_VCPUS=4
VM_DISK=60           # GiB
POOL_DIR=/home/vm-images
ISO=$POOL_DIR/iso/rhel-10.2-x86_64-dvd.iso
EOF
```

**各手順を始める前に、毎回これを流してください。**

```bash
source ~/.config/rhel10-vm.env
```

新しいターミナル、`newgrp` の直後、`sudo -i` から戻ったあと——シェルが変わったら
その都度必要です。次のチェックを癖にしておくと事故が防げます。

```bash
: "${VM_NAME:?変数が未設定です。source ~/.config/rhel10-vm.env を実行してください}"
: "${POOL_DIR:?変数が未設定です。source ~/.config/rhel10-vm.env を実行してください}"
: "${ISO:?変数が未設定です。source ~/.config/rhel10-vm.env を実行してください}"
```

> **変数が空のまま実行すると何が起きるか**
> `sudo` を付けてもシェル側で先に展開されるため、未設定の変数は**空文字に化けたまま
> root 権限のコマンドに渡ります**。実例として `POOL_DIR` が空だと手順 5 の
> `semanage fcontext -a -t virt_image_t "${POOL_DIR}(/.*)?"` が
> `"(/.*)?"`——**全パスにマッチするルール**になり、SELinux ポリシー本体が
> `virt_image_t` に化けて systemd がファイルを読めなくなります
> （→ トラブルシューティングの `can't connect to virtlogd`）。
> `virt-install` でも `--name` が空になり、ディスクが `/.qcow2` に作られます。

> **前提**
> ゲスト側の RHEL 10 を `subscription-manager` で登録するには、有効なサブスクリプション
> （Developer Subscription for Individuals でも可）が必要です。

---

## 1. 仮想化パッケージのインストール

RHEL 10.2 のリポジトリ（BaseOS / AppStream）に必要なものは揃っています。
`libvirt` はメタパッケージで、各モジュラードライバを引き込みます。

```bash
sudo dnf install -y \
  qemu-kvm \
  libvirt \
  libvirt-client \
  libvirt-daemon-config-network \
  virt-install \
  libosinfo \
  guestfs-tools \
  libvirt-nss
```

**確認**

```bash
rpm -q qemu-kvm libvirt virt-install
```

```
qemu-kvm-10.1.0-*.el10_2
libvirt-11.10.0-*.el10_2
virt-install-5.1.0-*.el10
```

`libvirt-daemon-config-network` が `default` NAT ネットワークの定義を持ち込みます。
これを入れ忘れると `virbr0` が作られません。

---

## 2. モジュラーデーモンの起動

RHEL 10 では一枚岩の `libvirtd` ではなく、ドライバごとに分かれた**モジュラーデーモン**
（`virtqemud` / `virtnetworkd` …）を使います。ソケット活性化なので、ソケットを
有効にすれば本体は必要時に起動します。

これは RHEL 10 の既定です。`/usr/lib/systemd/system-preset/90-default.preset` は
モジュラー側だけを `enable` 対象にしており、`libvirtd` の記載は一行もありません。
つまり `dnf install libvirt` した時点でこの構成が有効になっているはずですが、
確実にするため明示的に流します。

```bash
for drv in qemu network nodedev nwfilter secret storage interface; do
  sudo systemctl enable --now virt${drv}d.socket \
                              virt${drv}d-ro.socket \
                              virt${drv}d-admin.socket
done
sudo systemctl enable --now virtlogd.socket

# ホスト起動時のオートスタートには .service 側の有効化が必要
sudo systemctl enable --now virtqemud.service
sudo systemctl enable --now virtnetworkd.service
```

> **`.socket` だけでは足りない場面**
> ソケット活性化は「クライアントが接続してきたら起動する」仕組みです。手順 8 で
> `virsh autostart` を設定した VM をホスト起動時に自動で立ち上げるには、まだ誰も
> 接続していない段階で `virtqemud.service` が動いている必要があります。preset は
> `enable virtqemud.service` を含むので通常は既に有効です（起動後 2 分で終了し、
> 以後はソケット活性化に戻る挙動）。
>
> 一方 `virtnetworkd.service` は preset に**含まれていません**。VM のオートスタートに
> 引きずられて起動するので実害は出ませんが、明示的に有効にしておくと VM の有無に
> かかわらずホスト起動直後から `virbr0` が存在します。

**確認**

```bash
systemctl list-units --type=socket 'virt*d*.socket' --no-pager
systemctl is-enabled virtqemud.service virtnetworkd.service
```

> **任意 — libvirtd との併用は不可**
> RHEL 10 にも `libvirtd.service` は同梱されています（unit の Description は
> `libvirt legacy monolithic daemon`）が、モジュラーデーモンとは**同時に動かせません**。
> `libvirtd.socket` と、preset で有効化される `virtproxyd.socket` が同じ
> `/run/libvirt/libvirt-sock` を取り合うためです。誤って起動しないよう塞いでおけます。
>
> ```bash
> sudo systemctl mask libvirtd.service libvirtd.socket \
>                     libvirtd-ro.socket libvirtd-admin.socket
> ```
>
> これは必須ではありません。将来 libvirtd に切り替える可能性を残すなら飛ばして構いません
> （→ 付録 A）。

---

## 3. 操作ユーザーを libvirt グループへ

`qemu:///system` を `sudo` なしで触れるようにします。既定の接続先も固定しておくと、
`virsh` がユーザーセッション（`qemu:///session`）に迷い込む事故を防げます。

```bash
sudo usermod -aG libvirt "$USER"
echo 'export LIBVIRT_DEFAULT_URI=qemu:///system' >> ~/.bashrc

# グループ反映のため再ログイン、または現シェルだけ切り替える
newgrp libvirt

# ★ newgrp は新しいシェルを起動するため、ここで作業用の変数が消えています。
#    必ず読み直してください
source ~/.config/rhel10-vm.env
export LIBVIRT_DEFAULT_URI=qemu:///system
```

**確認**

```bash
virsh uri
virsh list --all
```

```
qemu:///system
 Id   名前   状態
--------------------
```

---

## 4. NAT ネットワーク default の起動

ここが本題です。`default` を起動して自動起動を付け、`virbr0` が期待どおり
出来上がったことを確認します。

```bash
virsh net-list --all
sudo virsh net-start default        # すでに active なら不要
sudo virsh net-autostart default
```

**確認**

```bash
virsh net-list --all
virsh net-dumpxml default
ip -br addr show virbr0
sysctl net.ipv4.ip_forward
```

```
 名前      状態   自動起動   永続
----------------------------------------
 default   活動中  はい      はい

virbr0  DOWN  192.168.122.1/24
net.ipv4.ip_forward = 1
```

> **`DOWN` で正常です**
> `ip -br` の 2 列目は operstate です。Linux ブリッジの operstate は配下ポートの
> carrier で決まるため、VM の tap（`vnet0`）がまだ 1 つも刺さっていないブリッジは
> `NO-CARRIER` → `DOWN` になります。手順 8 で VM を起動すると `UP` に変わります。
>
> 管理状態そのものは `ip -br link show virbr0` の**フラグ**側で確認します。
> ここに `UP` があれば ifup 済みです。
>
> ```
> virbr0  DOWN  52:54:00:...  <NO-CARRIER,BROADCAST,MULTICAST,UP>
>                              ^^^^^^^^^^ ポートが空       ^^ 管理上は UP
> ```
>
> 実際に使える状態かは、次の 3 つで判断してください。
>
> ```bash
> ping -c3 192.168.122.1                      # ブリッジまで届く
> ss -lnup | grep -E '192.168.122.1:53|:67'   # dnsmasq が DNS/DHCP を張っている
> ls /sys/class/net/virbr0/brif/              # VM 起動後は vnet0 が現れる
> ```

`net.ipv4.ip_forward` は現在 `0` ですが、`default` ネットワークの起動時に libvirt が
`1` にします。手で `sysctl` を書く必要はありません。

### firewalld との関係

libvirt は `virbr0` を専用の `libvirt` ゾーンに入れ、NAT 用の nftables ルールを
自分で管理します。Wi-Fi 側の `public` ゾーンには触らないので、既存の設定に影響はありません。

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --get-zone-of-interface=virbr0
sudo firewall-cmd --info-zone=libvirt
```

```
virbr0 → libvirt
（services: dhcp dhcpv6 dns ssh tftp / forward: yes）
```

---

## 5. ストレージプールを /home 上に作る

既定のプール `/var/lib/libvirt/images` は `/`（空き 63 GB）にあります。VM イメージと
ISO を置くと手狭なので、空き 816 GB の `/home` に作ります。`/home/redhat` は `0700` で
qemu プロセスが辿れないため、**`/home` の直下**に置くのが要点です。

```bash
source ~/.config/rhel10-vm.env

# ★ ガードは必須です。POOL_DIR が空だと下の semanage が全パスにマッチする
#    ルールを作り、SELinux ポリシーごと壊れます（復旧手順は末尾を参照）
if [ -z "$POOL_DIR" ] || [ "${POOL_DIR#/}" = "$POOL_DIR" ]; then
  echo "POOL_DIR が不正です: '$POOL_DIR' — source ~/.config/rhel10-vm.env を実行してください" >&2
else
  sudo mkdir -p "$POOL_DIR/iso"
  sudo chown root:root "$POOL_DIR"
  sudo chmod 0755 "$POOL_DIR"

  # SELinux: qcow2 と ISO を読めるようラベルを付ける
  sudo semanage fcontext -a -t virt_image_t "${POOL_DIR}(/.*)?"
  sudo restorecon -Rv "$POOL_DIR"
fi
```

**登録内容を必ず目視してください。** 意図した 1 行だけがあることを確認します。

```bash
cat /etc/selinux/targeted/contexts/files/file_contexts.local
```

```
# This file is auto-generated by libsemanage
# Do not edit directly.

/home/vm-images(/.*)?    system_u:object_r:virt_image_t:s0
```

ここに `(/.*)?` だけの行があったら**全ファイルにマッチする不正ルール**です。
直ちに削除してください（→ トラブルシューティング）。

**プールの定義**

```bash
sudo virsh pool-define-as vmpool dir --target "$POOL_DIR"
sudo virsh pool-build vmpool
sudo virsh pool-start vmpool
sudo virsh pool-autostart vmpool
```

**確認**

```bash
virsh pool-list --all
virsh pool-info vmpool
ls -ldZ "$POOL_DIR"
```

```
system_u:object_r:virt_image_t:s0  /home/vm-images
容量: 832.xx GiB / 利用可能: 816.xx GiB
```

> **SELinux**
> `semanage fcontext` を飛ばすと、インストール時に
> `Could not open '...qcow2': Permission denied` で止まります。
> ラベルが怪しいときは `sudo ausearch -m AVC -ts recent` で確認してください。

---

## 6. インストール ISO を配置する

`~/Downloads` に取得済みです。使うのは **DVD ISO**（BaseOS と AppStream の両リポジトリを
含むため、ネットワーク越しのリポジトリ指定なしでインストールできます）。

| ファイル | サイズ | 用途 |
|---|---|---|
| `rhel-10.2-x86_64-dvd.iso` | 11 GB | **本手順で使用。** 全パッケージ同梱 |
| `rhel-10.2-x86_64-boot.iso` | 994 MB | ネットインストール用。`inst.repo=` で外部リポジトリ指定が必要 |
| `rhel-10.2-x86_64-kvm.qcow2` | 1.1 GB | クラウドイメージ。cloud-init 方式に切り替える場合の素材（本手順では未使用） |

### 配置

`~/Downloads` と `/home/vm-images` は同じ `/home`（xfs）上なので、`mv` は一瞬で終わり
追加の空き容量も消費しません。ラベルは `mv` で引き継がれてしまうため、
`restorecon` が必須です。

```bash
sudo mv ~/Downloads/rhel-10.2-x86_64-dvd.iso "$POOL_DIR/iso/"
sudo chown root:root "$ISO"
sudo chmod 0644 "$ISO"
sudo restorecon -v "$ISO"
```

**確認**

```bash
ls -lhZ "$POOL_DIR/iso/"
virsh pool-refresh vmpool
osinfo-query os | grep -i rhel10
```

```
system_u:object_r:virt_image_t:s0  rhel-10.2-x86_64-dvd.iso
（user_home_t のままなら restorecon が効いていない）
```

### ISO の健全性チェック

ダウンロードが途中で切れていないかは、ISO 内部が宣言するボリュームサイズと
実ファイルサイズの比較で分かります。

```bash
file "$ISO"
SZ=$(stat -c %s "$ISO")
BLOCKS=$(od -An -tu4 -j $((32768+80)) -N4 "$ISO" | tr -d ' ')
echo "$SZ vs $((BLOCKS*2048))"    # 実サイズ >= 宣言サイズ なら切り詰めなし
```

検証済みの結果:

```
ISO 9660 CD-ROM filesystem data (DOS/MBR boot sector) 'RHEL-10-2-BaseOS-x86_64' (bootable)
11059986432 vs 11059652608   → OK
```

より厳密に見るなら、RHEL の ISO に埋め込まれたチェックサムを検証します
（11 GB の読み込みで数分かかります）。

```bash
sudo dnf install -y isomd5sum
checkisomd5 --verbose "$ISO"
```

> **osinfo に rhel10.2 が無かったら**
> 次の手順の `--osinfo` には、`osinfo-query` に出た中で一番近いもの（例 `rhel10.0`）を
> 指定してください。どうしても見つからない場合は `--osinfo detect=on,require=off` で
> 進められます（virtio ドライバの既定値が最適化されないだけで、動作はします）。

---

## 7. VM を作成してインストールする

GUI を使わないので、シリアルコンソール上でテキストモードの Anaconda を動かします。
`--location` は ISO から kernel / initrd を取り出して直接起動するため、
`--extra-args` でカーネル引数を渡せるのがポイントです。

```bash
source ~/.config/rhel10-vm.env

# 変数が空なら virt-install を実行させない（&& で繋ぐのが要点）
: "${VM_NAME:?未設定}" "${POOL_DIR:?未設定}" "${ISO:?未設定}" &&
sudo virt-install \
  --name "$VM_NAME" \
  --osinfo rhel10.2 \
  --memory "$VM_RAM" \
  --vcpus "$VM_VCPUS" \
  --cpu host-passthrough \
  --disk path=$POOL_DIR/$VM_NAME.qcow2,size=$VM_DISK,format=qcow2,bus=virtio,discard=unmap \
  --network network=default,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --location "$ISO" \
  --extra-args 'console=ttyS0,115200n8 inst.text'
```

コマンドがそのままコンソールに繋がり、テキストインストーラが始まります。
**コンソールから抜けるのは `Ctrl` + `]`**、戻るのは `virsh console $VM_NAME` です。

インストーラでは最低限、以下を設定してください。

- **Installation Destination** — `vda` を自動構成で確定
- **Software Selection** — Minimal Install または Server
- **Network & Host Name** — `enp1s0` を on（DHCP で 192.168.122.x を取得）
- **Root Password / User Creation** — 管理ユーザーを作り、wheel に入れる

完了すると VM は自動的に再起動し、ログインプロンプトが同じコンソールに出ます。

### 無人インストールにする場合（任意）

Kickstart を initrd に注入すれば対話なしで完了します。

```bash
cat > /tmp/ks.cfg <<'EOF'
text
lang ja_JP.UTF-8
keyboard jp
timezone Asia/Tokyo --utc
network --bootproto=dhcp --device=link --activate --hostname=rhel10-vm
rootpw --lock
user --name=admin --groups=wheel --password=CHANGE_ME --plaintext
bootloader --append="console=ttyS0,115200n8"
clearpart --all --initlabel
autopart --type=lvm
firewall --enabled --service=ssh
selinux --enforcing
services --enabled=sshd
reboot
%packages
@^minimal-environment
%end
EOF

: "${VM_NAME:?未設定}" "${POOL_DIR:?未設定}" "${ISO:?未設定}" &&
sudo virt-install \
  --name "$VM_NAME" --osinfo rhel10.2 \
  --memory "$VM_RAM" --vcpus "$VM_VCPUS" --cpu host-passthrough \
  --disk path=$POOL_DIR/$VM_NAME.qcow2,size=$VM_DISK,format=qcow2,bus=virtio,discard=unmap \
  --network network=default,model=virtio \
  --graphics none --console pty,target_type=serial \
  --location "$ISO" \
  --initrd-inject /tmp/ks.cfg \
  --extra-args 'inst.ks=file:/ks.cfg console=ttyS0,115200n8 inst.text'
```

> **パスワードは平文で残さない**
> 上の `--plaintext` は検証用です。実運用では `openssl passwd -6` で作ったハッシュを
> `--iscrypted` で渡し、`ks.cfg` は使用後に削除してください。

---

## 8. ゲスト側の仕上げ

対話インストールした場合、ゲストの GRUB にシリアルコンソール設定が入っていないことが
あります。ゲストにログインして次を実行しておくと、以後 `virsh console` で
ブートログまで見られます。

**ゲスト内で実行**

```bash
sudo grubby --update-kernel=ALL --args="console=ttyS0,115200n8"
sudo systemctl enable --now serial-getty@ttyS0.service

# サブスクリプション登録（必要な場合）
sudo subscription-manager register
sudo dnf -y update
```

**ホスト側**

```bash
virsh autostart "$VM_NAME"      # ホスト起動時に自動起動
virsh dominfo "$VM_NAME"
virsh domifaddr "$VM_NAME"
```

```
名前       MAC アドレス         プロトコル  アドレス
-------------------------------------------------------
vnet0      52:54:00:xx:xx:xx    ipv4        192.168.122.x/24
```

### VM 名で SSH できるようにする（任意）

手順 1 で入れた `libvirt-nss` を有効にすると、ホストから `ssh rhel10-vm` で届きます。

```bash
sudo sed -i 's/^hosts:\(.*\)files\(.*\)/hosts:\1files libvirt libvirt_guest\2/' /etc/nsswitch.conf
grep '^hosts:' /etc/nsswitch.conf
getent hosts "$VM_NAME"
```

---

## 9. 疎通確認

**ゲスト内で実行**

```bash
ip -br addr                        # 192.168.122.x/24 が付いているか
ping -c3 192.168.122.1             # ホスト(virbr0)まで
ping -c3 192.168.1.1               # LAN のルーターまで
getent hosts access.redhat.com     # dnsmasq 経由の名前解決
curl -sI https://access.redhat.com | head -1
```

**ホストで実行**

```bash
ssh admin@$(virsh -q domifaddr "$VM_NAME" | awk '{print $4}' | cut -d/ -f1)
sudo nft list table ip libvirt_network | grep -i masquerade
```

### チェックリスト

- [ ] ゲストが `192.168.122.0/24` の IP を DHCP で取得している
- [ ] ゲストからインターネットに出られる（名前解決込み）
- [ ] ホストからゲストへ SSH できる
- [ ] LAN の別端末からゲストへは届かない ← NAT なので正常
- [ ] ホスト再起動後も `default` と VM が自動起動する

---

## 10. 固定 IP とポート転送（任意）

### DHCP 予約でゲストの IP を固定する

ポート転送先を安定させるため、MAC に対して IP を予約します。

```bash
MAC=$(virsh -q domiflist "$VM_NAME" | awk '{print $5}')
echo "$MAC"

sudo virsh net-update default add ip-dhcp-host \
  "<host mac='$MAC' name='$VM_NAME' ip='192.168.122.10'/>" \
  --live --config

virsh net-dumpxml default | grep dhcp -A5
```

反映にはゲストのリース更新が必要です。`virsh reboot $VM_NAME` が手早いです。

### LAN からゲストの SSH に届かせる

Wi-Fi 側（`public` ゾーン）の TCP 2222 をゲストの 22 に転送します。

```bash
sudo firewall-cmd --permanent --zone=public \
  --add-forward-port=port=2222:proto=tcp:toport=22:toaddr=192.168.122.10
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --reload
sudo firewall-cmd --info-zone=public
```

LAN の別端末から `ssh -p 2222 admin@192.168.1.24` で接続できます。

> **注意**
> firewalld で別ホスト宛にポート転送するには `--add-masquerade` が必須です。
> これは `public` ゾーンを出る通信全般に SNAT を適用するため、このホストが他の用途で
> ルーティングを担っている場合は影響を確認してから入れてください。

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `virsh` が `Permission denied` / 何も表示されない | libvirt グループ未反映、または `qemu:///session` に繋がっている | 再ログイン後 `id` で確認。`export LIBVIRT_DEFAULT_URI=qemu:///system` |
| `Could not access KVM kernel module` | `/dev/kvm` の権限、または kvm_amd 未ロード | `ls -l /dev/kvm` / `sudo modprobe kvm_amd` |
| `can't connect to virtlogd` で `virt-install` が失敗する | `virtlogd.service` が `status=226/NAMESPACE` で起動できない。ほぼ SELinux ポリシーの誤ラベル | 下の「virtlogd が起動しない」を参照 |
| `virtlogd.service: Failed to set up mount namespacing: /dev: Permission denied` | systemd が `file_contexts` を読めず `PrivateDevices=true` の構築に失敗 | 同上。`ls -Z /etc/selinux/targeted/policy/policy.35` を確認 |
| `virbr0` が `DOWN` に見える | **正常**。VM 未起動でブリッジにポートが無く `NO-CARRIER` | 対処不要。`ip -br link show virbr0` のフラグに `UP` があり `ping 192.168.122.1` が通れば問題なし。VM 起動後に `UP` になる |
| `virbr0` が存在しない | `libvirt-daemon-config-network` 未導入、または `virtnetworkd` 未起動 | 手順 1・2 をやり直す。`virsh net-define /usr/share/libvirt/networks/default.xml` |
| `Network not found: no network with matching name 'default'` | ネットワーク定義が消えている | 上記 XML から `net-define` → `net-start` → `net-autostart` |
| qcow2 が `Permission denied` で開けない | SELinux ラベル（手順 5 の `semanage` 漏れ） | `ls -Z` で確認 → `restorecon -Rv $POOL_DIR` / `ausearch -m AVC -ts recent` |
| ゲストから外に出られない | ホストの Wi-Fi が切れている、または `ip_forward=0` | `nmcli device status` / `sysctl net.ipv4.ip_forward` / `virsh net-list` |
| 名前解決だけ失敗する | ゲストの DNS が 192.168.122.1 を向いていない | ゲストで `resolvectl status`。dnsmasq が上流を引き継ぐので通常は自動 |
| `--location` がエラーになる | ISO が読めない／boot 用ツリーを含まない | DVD ISO であることを確認。回避策として `--cdrom "$ISO"`（ただしテキストモードは使えない） |
| `virsh` の接続先が起動のたびに変わる／VM が見えたり消えたりする | libvirtd とモジュラーデーモンを両方有効にしている | `sudo ss -lx \| grep libvirt-sock` で確認し、どちらか一方に統一する（→ 付録 A） |
| ホスト再起動後に VM が自動起動しない | `virtqemud.service` が無効 | `systemctl is-enabled virtqemud.service` → `sudo systemctl enable --now virtqemud.service` |
| `virsh console` に何も出ない | ゲストの GRUB に `console=ttyS0` が無い | 手順 8 の `grubby --update-kernel=ALL --args=...` |
| コンソールから抜けられない | — | `Ctrl` + `]` |

---

### virtlogd が起動しない（`226/NAMESPACE`）

`virt-install` が次で止まる場合です。

```
ERROR    can't connect to virtlogd: Cannot recv data: Connection reset by peer
```

`journalctl -b -u virtlogd.service` に次が出ていれば、このケースです。

```
Failed to initialize SELinux labeling handle: Permission denied
virtlogd.service: Failed to set up mount namespacing: /dev: Permission denied
virtlogd.service: Failed at step NAMESPACE spawning /usr/sbin/virtlogd: Permission denied
```

**原因**は手順 5 を `POOL_DIR` が空のまま実行したことです。
`semanage fcontext -a -t virt_image_t "(/.*)?"` が全パスにマッチするルールとして
登録され、その後 libsemanage がポリシーを再生成した際に**生成物自身が
`virt_image_t` になります**。systemd は `init_t` で動くため
`virt_image_t` のポリシーファイルを読めず、`PrivateDevices=true` の
名前空間構築に失敗します。

```bash
# 診断: virt_image_t に化けていないか
ls -Z /etc/selinux/targeted/policy/policy.35 \
      /etc/selinux/targeted/contexts/files/file_contexts \
      /etc/selinux/targeted/seusers
cat /etc/selinux/targeted/contexts/files/file_contexts.local
```

**復旧は順番が重要です。先に不正ルールを消してから relabel してください。**
逆順だと再生成で再び壊れます。

```bash
# 1. 全マッチの不正ルールを削除
sudo semanage fcontext -d "(/.*)?"

# 2. (/.*)? の行が消えたことを確認
cat /etc/selinux/targeted/contexts/files/file_contexts.local

# 3. 壊れたラベルを修復
sudo restorecon -Rv /etc/selinux

# 4. virt_image_t が出なくなったことを確認
ls -Z /etc/selinux/targeted/policy/policy.35 \
      /etc/selinux/targeted/contexts/files/file_contexts \
      /etc/selinux/targeted/seusers

# 5. 失敗状態をリセットして再起動
sudo systemctl reset-failed virtlogd.service virtlogd.socket virtlogd-admin.socket
sudo systemctl start virtlogd.socket
systemctl is-active virtlogd.socket
```

手順 1 が失敗する場合は直接編集して再ビルドします。

```bash
sudo sed -i '/^(\/\.\*)?[[:space:]]/d' /etc/selinux/targeted/contexts/files/file_contexts.local
sudo semodule -B
sudo restorecon -Rv /etc/selinux
```

期待される relabel 結果です。

```
Relabeled .../policy/policy.35              ... to system_u:object_r:semanage_store_t:s0
Relabeled .../contexts/files/file_contexts  ... to system_u:object_r:file_context_t:s0
Relabeled .../seusers                       ... to system_u:object_r:selinux_config_t:s0
```

> **被害範囲**
> `restorecon -R /` を実行していなければ、壊れるのは libsemanage が再生成した
> `/etc/selinux/targeted/` 配下の数ファイルだけです。全体再ラベルは不要です。
> なお `/var/lib/libvirt/images` の `virt_image_t` は**既定どおりで正常**です。

---

## 撤収

作り直したいとき、VM とディスクをまとめて破棄します。
**`--remove-all-storage` は qcow2 を実際に削除します。**

```bash
virsh destroy "$VM_NAME"                 # 電源断（停止済みならエラーで無害）
virsh undefine "$VM_NAME" --remove-all-storage --nvram
```

ホスト側の設定ごと戻す場合。

```bash
sudo virsh pool-destroy vmpool
sudo virsh pool-undefine vmpool
sudo semanage fcontext -d "${POOL_DIR}(/.*)?"
sudo virsh net-autostart default --disable
```

---

## 付録 A: libvirtd（legacy monolithic daemon）を使う場合

**libvirtd は禁止ではありません。** RHEL 10.2 にも同梱されており、普通に動作します。
ただし既定ではないため、使うには明示的な切り替えが必要です。

### RHEL 10 での位置づけ（実機で確認した事実）

| 確認項目 | 結果 |
|---|---|
| パッケージ | `libvirt-daemon-11.10.0-12.4.el10_2` に同梱（削除されていない） |
| unit の Description | `libvirt legacy monolithic daemon` |
| 同梱 unit | `libvirtd.service` / `.socket` / `-ro` / `-admin` / `-tcp` / `-tls` |
| systemd preset | `enable libvirtd.*` の記載は**一行もない** |
| 旧クライアント互換 | preset は代わりに `virtproxyd.socket` を enable |

### 併用できない理由

| unit | ListenStream |
|---|---|
| `libvirtd.socket` | `/run/libvirt/libvirt-sock` |
| `virtproxyd.socket`（preset で有効） | 同じ `/run/libvirt/libvirt-sock` |

両方を有効にすると先に起動した方が勝ち、`virsh` の接続先が状況次第で変わります。
加えて libvirtd と virtqemud が同じ VM の状態を二重に管理することになるため、
どちらか一方に決める必要があります。

### 切り替え手順（手順 2 の代替）

```bash
# モジュラー側を全部止める（proxy を含めるのが要点）
for drv in qemu network nodedev nwfilter secret storage interface proxy; do
  sudo systemctl disable --now virt${drv}d.service \
                               virt${drv}d.socket \
                               virt${drv}d-ro.socket \
                               virt${drv}d-admin.socket 2>/dev/null
done

# mask 済みなら解除
sudo systemctl unmask libvirtd.service libvirtd.socket \
                      libvirtd-ro.socket libvirtd-admin.socket

# libvirtd を有効化（libvirtd は virtlogd.socket を Requires している）
sudo systemctl enable --now virtlogd.socket
sudo systemctl enable --now libvirtd.socket
sudo systemctl enable --now libvirtd.service
```

**確認**

```bash
virsh uri
systemctl status libvirtd --no-pager
sudo ss -lx | grep libvirt-sock
```

手順 3 以降はまったく同じです。`virsh` / `virt-install` の使い方も、NAT ネットワークの
挙動も変わりません。トラブル時に見るログが `journalctl -u libvirtd` に集約される点だけが
違います。

### 選択の目安

| | libvirtd | モジュラーデーモン |
|---|---|---|
| RHEL 10 での位置づけ | legacy・非推奨（動作はする） | **既定**（preset で有効） |
| 設定の手間 | 切り替え作業が必要 | インストールしただけで済む |
| 障害の分離 | 1 プロセス。再起動すると全ドライバ停止 | ドライバ単位で再起動できる |
| ログ | `journalctl -u libvirtd` に集約 | ドライバごとに分かれる |
| 将来 | 削除予定 | 継続 |

新規構築なら既定のまま（モジュラー）を推奨します。RHEL 7/8 時代の運用手順や
スクリプトが libvirtd 前提になっている、といった事情がある場合に上の切り替えを
検討してください。

---

*ホスト実測値（RHEL 10.2 / Ryzen 5 3600 / wlp7s0f3u1 192.168.1.24）に基づいて作成 · 2026-08-26*
*Web 版: https://claude.ai/code/artifact/1020a5b2-6198-45b1-9992-c93caab8294b*

# RaspberryPi2台でKuberenetesを使えるようになるまで
備忘録です。

- [RaspberryPi2台でKuberenetesを使えるようになるまで](#raspberrypi2台でkuberenetesを使えるようになるまで)
  - [下準備](#下準備)
    - [必要な機材などを用意](#必要な機材などを用意)
    - [RaspberryPiにUbuntuをクリーンインストール](#raspberrypiにubuntuをクリーンインストール)
    - [初期ログイン、IPアドレス固定、拡張モジュールのインストール](#初期ログインipアドレス固定拡張モジュールのインストール)
  - [コンテナランタイムのインストール](#コンテナランタイムのインストール)
  - [Kuberenetesのインストール](#kuberenetesのインストール)
    - [マスターノード、ワーカーノードでの共通作業](#マスターノードワーカーノードでの共通作業)
    - [マスターノードでの作業](#マスターノードでの作業)
    - [ワーカーノードでの作業](#ワーカーノードでの作業)
  - [CNIのインストール](#cniのインストール)
  - [metrics-serverのインストール](#metrics-serverのインストール)

## 下準備
### 必要な機材などを用意
- RaspberryPi4 ModelB/8GB（中古、マスターノード）
- RaspberryPi4 ModelB/4GB（中古、ワーカーノード）
- PoE Hat x2
- TP-Link TL-SG1005P（スイッチングハブ）
- KIOXIA microSD 16GB x4
- LANケーブル CAT6 20cm x10
- RaspberryPi用ラック

### RaspberryPiにUbuntuをクリーンインストール
RaspberryPiImagerを使って、microSDにUbuntuイメージ書き込む。バージョンは`Ubuntu Server 22.04 LTS`を選択する。特にカスタム設定などする必要はなく、そのままmicroSDに書き込む。

### 初期ログイン、IPアドレス固定、拡張モジュールのインストール
書き込み済みのmicroSDをRaspberryPiに挿入し、スイッチングハブ経由でLANに接続し、RaspberryPiの電源を入れる。RaspberryPiのIPアドレスを調べる。SSHでRaspberryPiに接続を試みる。
```
ssh ubuntu@192.168.11.xxx
```
初期接続はユーザ名とパスワードはデフォルトの`ubuntu`で問題ない。パスワードの変更を求められるので、任意のものに変更する。
<br>

変更後再接続し、固定IPアドレスを設定するために、`/etc/netplan/`にカスタムの設定ファイルを追加する。
```
sudo vim /etc/netplan/99-custom-conf.yaml
```
マスターノードには内容は以下の通り設定する。ワーカーノードにはIPアドレスは`192.168.11.101`を設定する。
```yaml
network:
  ethernets:
    eth0:
      dhcp4: false
      addresses:
      - 192.168.11.100/24
      gateway4: 192.168.11.1
      nameservers:
        addresses:
        - 192.168.11.1
        search: []
  version: 2
```
ホスト名も変更する。マスターノードは`k8s-master`、ワーカーノードは`k8s-worker01`と設定する。
```
sudo vim /etc/hostname
```
システム時刻のタイムゾーンをAsia/Tokyoに設定する。
```
sudo timedatectl set-timezone Asia/Tokyo
```
RaspberryPi用のVXLAN拡張モジュールをインストールする。これがないとCNIが動かない。
```
sudo apt update
sudo apt install linux-modules-extra-raspi -y
```
その後、再起動し新しいIPアドレスで再接続する。
```
sudo reboot
```

## コンテナランタイムのインストール
今回はCRI-Oをコンテナランタイムとして選択した。
<br>

IPv4フォワーディングを有効化し、iptablesからブリッジされたトラフィックを見えるようにする。
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
aptアップデート、アップグレードを行い、再起動する。
```
sudo apt update
sudo apt upgrade -y
```
```
sudo reboot
```
SSHで再接続する。環境変数を以下の通り設定する。今回は、CRI-Oのバージョンは1.28を選択する。
```
export OS=xUbuntu_22.04
export CRIO_VERSION=1.28
```
CRI-Oのkubicリポジトリを追加する。
```
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /"| sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$CRIO_VERSION/$OS/ /"|sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$CRIO_VERSION.list
```
GPGキーをaptのパッケージリポジトリに追加する。
```
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$CRIO_VERSION/$OS/Release.key | sudo apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key add -
```
aptのアップデートを行う。
```
sudo apt update
```
cri-oとcri-o-runcをインストールする。
```
sudo apt install cri-o cri-o-runc -y
```
crioを起動し、サービスを有効にする。
```
sudo systemctl start crio
sudo systemctl enable crio
sudo systemctl status crio
```
CRI-OにCNIプラグインはインストールされておらず、設定もされていない。CNIプラグインをインストールする。
```
sudo apt install containernetworking-plugins -y
```
インストール完了後、cri-oの設定ファイルを編集する。
```
sudo vim /etc/crio/crio.conf
```
`network_dir`と`plugin_dirs`の行のコメントアウトを外し、`network_dir`内に`"/usr/lib/cni/",`を追記する。
```
# Path to the directory where CNI configuration files are located.
network_dir = "/etc/cni/net.d/"
# Paths to directories where CNI plugin binaries are located.
plugin_dirs = [
        "/opt/cni/bin/",
        "/usr/lib/cni/",
]
```
crio-bridgeの設定ファイルを削除し、最新のものをダウンロードする。
```
sudo rm -f /etc/cni/net.d/100-crio-bridge.conf
sudo curl -fsSLo /etc/cni/net.d/11-crio-ipv4-bridge.conflist https://raw.githubusercontent.com/cri-o/cri-o/main/contrib/cni/11-crio-ipv4-bridge.conflist
```
crioを再起動し、設定を反映させる。
```
sudo systemctl restart crio
```
crictlもインストールする。
```
sudo apt install cri-tools -y
```
systemdがinitシステムに採用されているか確認する。
```
ps -p 1 -o comm=
```
systemdだったが、CRI-Oはcgroupドライバーをデフォルトでsystemdにしているので、特に設定変更はしなかった。

## Kuberenetesのインストール
### マスターノード、ワーカーノードでの共通作業
aptのパッケージ一覧を更新し、K8sのaptリポジトリを利用するのに必要なパッケージをインストールする。
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
GoogleCloudの公開鍵をダウンロードする。
```
curl -fsSL https://dl.k8s.io/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
```
K8sのaptリポジトリを追加する。
```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
aptのパッケージ一覧を更新し、kubelet、kubeadm、kubectlをインストールする。
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
kubeadmはkubelet向けcgroupドライバーを検出するが、systemdの場合は、以下のファイルを作成する。

```
sudo vim /etc/default/kubelet
```
記載内容は以下である。`kubeadm init`および`kubeadm join`を実行するときに、このユーザー定義引数が取得される。
```
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd
```
kubeletを再起動する。
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### マスターノードでの作業
kubeadmでマスターノード化する。PodのCIDRは後でインストールするCalicoに合わせる。
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
root権限以外でもkubectlを使えるようにする。
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
マスターノードにもPodをスケジューリングできるようにするため、マスターノードのtaintを削除する。今回はノード2台構成でリソースが少ないためである。
```
kubectl taint node k8s-master node-role.kubernetes.io/control-plane:NoSchedule-
```
`.bashrc`にコマンド省略と補完機能設定を書き込み、再読み込みする。 
```
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -F __start_kubectl k" >> ~/.bashrc
source ~/.bashrc
```

### ワーカーノードでの作業
マスターノードでの`sudo kubeadm init`実行時に、標準出力の最下行に`kubeadm join ~~`のコマンドが表示される。これをそのままワーカーノードのroot権限で実行する。

## CNIのインストール
今回はCalicoを採用する。NetworkPolicyを利用したいためである。インストールはマスターノードで行う。
<br>

TigeraCalicoオペレータとカスタムリソース定義をインストールする。
```
k create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
```
インストール用のリソースをスケジューリングする。
```
k create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

## metrics-serverのインストール
マスターノードでmetrics-serverをデプロイする。
```
k apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
そのままだとmetrics-serverが起動しないので、metrics-serverのDeploymentを編集する。
```
k edit deploy metrics-server -n kube-system
```
`args:`セクションに`--kubelet-insecure-tls`を追記する。
```
- args:
  - --cert-dir=/tmp
  - --secure-port=10250
  - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
  - --kubelet-use-node-status-port
  - --metric-resolution=15s
  - --kubelet-insecure-tls
```
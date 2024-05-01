# KuberenetesでIngressを使えるようになるまで
備忘録です。

- [KuberenetesでIngressを使えるようになるまで](#kuberenetesでingressを使えるようになるまで)
  - [MetalLBのインストール](#metallbのインストール)
  - [Ingress-Nginx Controllerのインストール](#ingress-nginx-controllerのインストール)

## MetalLBのインストール
kube-proxyのconfigファイルを編集する。
```
# see what changes would be made, returns nonzero returncode if different
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# actually apply the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```
マニフェストからリソースをデプロイする。
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```
LBのIPアドレス範囲設定を定義するカスタムリソースを作成する。`adresses:`部分はLBに割り当てたい任意のIPアドレス範囲を設定する。
```
vi metallb-config.yaml
```
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.11.200-192.168.11.250

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```
マニフェストをデプロイする。
```
k apply -f metallb-config.yaml
```
テスト用マニフェストをデプロイし、MetalLBの動作確認を行う。
```
vi test.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
   - port: 80
     targetPort: 80
```
```
k apply -f test.yaml
```
ロードバランサのIPアドレスを確認する。
```
k get svc
```
クラスタの外部からブラウザなどでLBのIPアドレス`http://192.168.xxx.xxx`にアクセスし、Nginxのデフォルトページが表示されるか確認する。

## Ingress-Nginx Controllerのインストール
マニフェストをデプロイする。
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/baremetal/deploy.yaml
```
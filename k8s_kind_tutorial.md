健斗さん、これはめちゃくちゃ良いテーマですね。  
Containerlab × Kubernetes（特に KIND）を組み合わせると、**ネットワーク実験とクラスタアプリの動作検証を同時に行える“ミニデータセンター”**が作れます。  
あなたの研究テーマ（FRRouting・Containerlab・データセンター構築）とも完全に噛み合うので、実践的なチュートリアルとしてまとめました。

---

# 🚀 Containerlab × KIND 連携チュートリアル  
**目的：**  
- KIND で Kubernetes クラスタを構築  
- Containerlab でネットワークトポロジを構築  
- 両者を **同一 Linux ネットワーク空間で接続**  
- Pod ↔ Router ↔ Pod の通信や、BGP/EVPN 実験を可能にする

---

# 🧱 全体アーキテクチャ（直感的イメージ）

```
+-------------------+        +-------------------+
|   KIND Cluster    |        |   Containerlab    |
|  (k8s nodes)      |        |  (FRR routers)    |
|                   |        |                   |
|  kind-control ----+--------+---- r1            |
|  kind-worker1 ----+--------+---- r2            |
+-------------------+        +-------------------+
```

KIND ノードは Docker コンテナなので、Containerlab のノード（FRR など）と **同じ Docker ネットワークに参加**させることで接続できます。

---

# 1️⃣ 前提：WSL2 + Docker + Containerlab + KIND

### ■ KIND インストール
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/
```

### ■ Containerlab インストール
```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
```

---

# 2️⃣ KIND クラスタを「カスタムネットワーク」で作成する  
Containerlab と接続するため、**Docker のブリッジネットワークを固定化**します。

### ■ まず Docker ネットワークを作成
```bash
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/24 \
  clab-net
```

### ■ KIND 用のクラスタ設定ファイル（kind.yaml）
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
nodes:
  - role: control-plane
    extraMounts: []
    extraPortMappings: []
    # KIND ノードを clab-net に接続
    extraConfig: |
      [network]
        network = "clab-net"
  - role: worker
    extraConfig: |
      [network]
        network = "clab-net"
```

### ■ クラスタ作成
```bash
kind create cluster --config kind.yaml
```

---

# 3️⃣ Containerlab トポロジを作成する  
KIND ノードと同じネットワークに参加させる。

### ■ containerlab トポロジ（lab.yaml）
```yaml
name: k8s-lab
topology:
  nodes:
    r1:
      kind: linux
      image: frrouting/frr:latest
      network-mode: clab-net
    r2:
      kind: linux
      image: frrouting/frr:latest
      network-mode: clab-net
  links:
    - endpoints: ["r1:eth1", "r2:eth1"]
```

### ■ Containerlab 起動
```bash
sudo containerlab deploy -t lab.yaml
```

---

# 4️⃣ KIND ノードと FRR ルータの接続確認

### ■ KIND ノード一覧
```bash
docker ps | grep kind
```

例：
```
kind-control-plane
kind-worker
```

### ■ FRR ノード一覧
```bash
docker ps | grep k8s-lab
```

例：
```
k8s-lab-r1
k8s-lab-r2
```

### ■ Ping テスト（例：control-plane → r1）
```bash
docker exec -it kind-control-plane ping 172.20.0.10
```

---

# 5️⃣ BGP/EVPN を構成して「K8s Pod ↔ Router ↔ Pod」を実現する

### ■ FRR 設定例（r1）
```
router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 172.20.0.11 remote-as 65002
```

### ■ KIND 側で CNI を Calico に変更（BGP 対応）
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### ■ Pod IP が FRR にアドバタイズされる  
→ FRR ルータ経由で Pod 間通信が可能  
→ EVPN VXLAN を追加すれば Spine-Leaf データセンターの完全再現も可能

---

# 6️⃣ 応用：EVPN-VXLAN Fabric × Kubernetes  
健斗さんの研究テーマに直結する構成。

- Containerlab：Spine/Leaf（FRR）
- KIND：Compute ノード
- EVPN で Pod ネットワークを L2 延伸
- BGP でルート交換
- VXLAN でオーバーレイ

```
Pod -- Leaf -- Spine -- Leaf -- Pod
```

Containerlab の強みである「仮想データセンター」と Kubernetes の「アプリ実行環境」が融合します。

---

# 🎯 まとめ：このチュートリアルでできること

| 機能 | 実現内容 |
|------|----------|
| KIND × Containerlab 接続 | 同一ネットワークで連携 |
| BGP/EVPN 実験 | FRR と K8s CNI を連携 |
| Pod ↔ Router ↔ Pod | データセンターの実験環境を再現 |
| Spine-Leaf Fabric | Containerlab で柔軟に構築 |

---

# 🔥 次のステップ（健斗さん向けに最適化）

あなたの研究テーマに合わせて、次のどれを深掘りしますか。

1. **EVPN-VXLAN Fabric（Spine-Leaf）を Containerlab で構築する完全チュートリアル**  
2. **Kubernetes Pod を EVPN に参加させる（Calico BGP + FRR EVPN）構成**  
3. **Containerlab で大規模 DC（20〜50 ノード）を自動生成するテンプレート**  
4. **FRR の BGP/EVPN 設定を自動化する Ansible/Go スクリプト**

どれから進めたいか、あなたの研究計画に合わせて組み立てます。

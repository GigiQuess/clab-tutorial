# データセンターネットワーク構築・学習計画 (Containerlab)

データセンター (DC) を構成する最新のネットワーク技術（Spine-Leaf アーキテクチャ、BGP、VXLAN、EVPN など）を、`containerlab` を用いて実践的に学ぶための詳細な学習計画です。

## 学習の目的
* モダンなデータセンターネットワークアーキテクチャ (Spine-Leaf Topology) を理解する。
* アンダーレイ (Underlay) ネットワークとオーバーレイ (Overlay) ネットワークの概念と実装を学ぶ。
* BGP EVPN/VXLAN を深く理解し、手元で構築・パケットレベルで検証できるスキルを身につける。

## 必要な環境・前提知識
* **ツール**: Docker, Containerlab, Wireshark (パケットキャプチャ用)
* **OS (NOS)**: ネットワーク機器の OS として、FRRouting (FRR), Nokia SR Linux, Arista cEOS, Juniper cPTX などを利用します。
  * 本計画では無償で扱いやすく、モダンな API や CLI を備える **Nokia SR Linux** または **FRR** での学習を想定しています。
* **前提知識**: 基本的な TCP/IP、L2 スイッチング、OSPF/BGP の基礎知識。

---

## 🛑 Phase 1: 環境構築と Containerlab の基礎
まずは検証環境を整理し、ツールと仮想ネットワークの扱いに慣れます。

1. **環境セットアップ**
   * Linux マシン (Ubuntu 推奨) 上に Docker と Containerlab をインストール・設定する。
2. **Containerlab の基本動作**
   * 2 台のルーター（または Linux ホスト）を直結するシンプルな YAML トポロジ (`.clab.yml`) を作成する。
   * トポロジのデプロイ (`clab deploy`)、破棄 (`clab destroy`)、状態確認 (`clab inspect`) を実行し、ライフサイクルを理解する。
   * コンテナ内へのアクセス (`docker exec` または組み込みの SSH) 方法の確認。
3. **パケットキャプチャの習得**
   * リンク間のパケットをホスト側からキャプチャし、Wireshark で閲覧する方法を確認する（Containerlab 標準のパケットキャプチャ機能の活用）。

---

## 🏗️ Phase 2: Spine-Leaf アーキテクチャの構築 (アンダーレイ)
モダンな DC の物理トポロジ (Clos ネットワーク) を構築します。

1. **Spine-Leaf トポロジの作成**
   * Spine スイッチ 2 台、Leaf スイッチ 3 台、各 Leaf に接続されるホストを用意する構成 YAML を作成する。
2. **IP アドレス設計**
   * P2P リンク（Spine-Leaf 間）の IP アドレス設計 (例: `/31` ボンディング または IPv6 Link-Local を用いた BGP Unnumbered)。
   * ルーター ID (Router-ID) および VTEP (VXLAN Tunnel Endpoint) の発信元となる Loopback アドレス設計。
3. **設定の流し込み**
   * 各ノードにインターフェース設定を行い、Ping で隣接（直結）ノードと通信できることを確認する。

---

## 🌐 Phase 3: BGP によるアンダーレイ・ルーティング
DC 内の経路制御として、現在主流となっている eBGP を用いたルーティングを設計・実装します。

1. **eBGP の設定**
   * RFC 7938 (Use of BGP for Routing in Large-Scale Data Centers) に基づくプライベート AS 番号 (Private ASN) の割り当て設計。
   * Spine と Leaf の間で eBGP ピアを確立する。
2. **Loopback アドレスの広報**
   * 各スイッチの Loopback アドレスを BGP テーブルに注入し、ネットワーク全体に広報する。
3. **ECMP (Equal-Cost Multi-Path) の確認**
   * Leaf から別の Leaf の Loopback に Ping を打つ。
   * インターフェースを 1 本ダウンさせても通信が継続できることや、ルーティングテーブルにトラフィックの等コスト分散 (ECMP) が表示されることを確認する。

---

## 🚇 Phase 4: VXLAN によるオーバーレイ (データプレーン)
物理ネットワーク (アンダーレイ) の上に、論理的な L2 ネットワーク (オーバーレイ) を構築します。

1. **VXLAN の概念理解**
   * VTEP と VNI (VXLAN Network Identifier) の役割を理解する。VXLAN ヘッダの構造も抑える。
2. **静的 VXLAN の設定 (Static VXLAN)**
   * BGP 等のコントロールプレーンに依存せず、手動で宛先 VTEP IP を指定して静的に VXLAN トンネルを構築する。
   * 別々の Leaf 配下にいる同一サブネットのホスト同士で L2 通信 (Ping) が通ることを確認する。
3. **トラフィックの可視化**
   * Spine-Leaf 間でパケットキャプチャを行い、ホスト間の Ping が UDP ヘッダ (UDP over IPv4) で VXLAN カプセル化されていることを確認する。

---

## 🧠 Phase 5: BGP EVPN の導入 (コントロールプレーン)
MAC アドレスの学習や通信をフラッディング (Data-Plane Learning) に頼らず、BGP (Control-Plane) で効率的に経路交換・学習します。

1. **MP-BGP EVPN Address Family の有効化**
   * Spine-Leaf 間で L2VPN EVPN アドレスファミリの BGP ピアリングを確立。
   * （Spine を Route Reflector として機能させる iBGP 構成か、eBGP ベースの EVPN かを選択・設定する）
2. **EVPN L2 モード設定 (Type 2 / Type 3 Route)**
   * MAC アドレス情報 (EVPN Type-2) と BUM トラフィック転送用 (EVPN Type-3) の経路情報をやり取りする設定を行う。
   * Leaf の BGP EVPN ルーティングテーブルを確認し、対向ホストの MAC アドレスが BGP で学習・広報されていることを確認する。
3. **EVPN IRB (Integrated Routing and Bridging) - L3VPN (Type 5 Route)**
   * 異なる VNI (サブネット) 間を Leaf スイッチ上でルーティングする仕組み (Distributed Default Gateway/Anycast Gateway) を構築する。
   * Symmetric IRB と Asymmetric IRB の違いを座学で理解し、Symmetric IRB モデルを設定・検証する (EVPN Type-5 または Type-2 拡張を利用)。

---

## 🚀 Phase 6: 実践的機能と応用 (Advanced)
より実際の DC 環境に近い高度な要件を実装します。

1. **EVPN Multi-Homing (Type 1 / Type 4 Route)**
   * サーバー (ホスト) を 2 台の Leaf スイッチに冗長接続する。従来の MLAG/MC-LAG に代わる EVPN ベースのマルチホーミング技術。
   * ESI (Ethernet Segment Identifier) を設定し、All-Active モードでトラフィックが両 Leaf に分散・保護されるかを検証する。
2. **テナントの分離 (VRF / Multi-tenancy)**
   * 共通のインフラ上で、組織やシステム(テナント)ごとにルーティングインスタンス (VRF/Network Instance) を分割し、L3 レベルでのアイソレーション (L3 ネットワークの完全分離) を検証する。
3. **インテントベース・ネットワーク自動化の体験**
   * 構成が複雑化してきたタイミングで Ansible、Nornir (Python)、または Terraform (プロバイダがあれば) を用い、Phase 2〜5 のコンフィグを自動生成してルーターへ流し込む。
   * Infrastructure as Code (IaC) 的なネットワーク運用を手元で学習する。

---

## まとめ・次のステップ
本学習計画を一つ一つクリアしていくことで、現在主流となっている Spine-Leaf アーキテクチャと EVPN-VXLAN ファブリックの設計・構築をゼロから深く理解できます。

まずは簡単な Containerlab トポロジを書き始める Phase 1/2 に着手してみてください。「このツールの使い方を知りたい」「Phase 2のYAML書き方から始めたい」など、次のステップにおけるサポートが必要であればお知らせください。

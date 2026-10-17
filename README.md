# SideStore-Tailscale

SideStoreのアプリ更新・インストールを、Tailscaleを切断・切り替えずに実行するためのDocker Compose構成じゃ。

新しいSideStore（0.6.3以降の新しい `idevice` / Swift minimuxer実装）向けに、[shiqi917/sidestore-vpn-tailscale](https://github.com/shiqi917/sidestore-vpn-tailscale) をDocker Composeでビルド・実行する。元の [xddxdd/sidestore-vpn](https://github.com/xddxdd/sidestore-vpn) の `10.7.0.1/32` を配布する旧方式は使わない。

## 必要なもの

- Docker EngineとDocker Compose（Git URLからのビルドに対応するもの）
- ホストで利用できる `/dev/net/tun`
- 同一のTailnetに接続しているiPhone / iPadのTailscale
- 対応するSideStoreとペアリングファイル（新しい実装ではRPPairing形式）
- **iOS端末のWi-Fi接続**（モバイル通信のみは開発元によると非対応）

## 初期設定

1. このリポジトリを取得する。
2. `.env.example` を `.env` にコピーし、必要に応じて有効なTailscale認証キーを入力する。既存の `./tailscale-state` が認証済みなら、その状態を維持する。**認証キーや状態ディレクトリはGitHubにアップロードしないこと。**
3. Composeを起動する。

```bash
cp .env.example .env
docker compose config -q
docker compose up -d --build
docker compose logs --tail=100 sidestore-vpn
```

ログの `reflector: node address 100.x.y.z` がSideStore側に設定する **Device IP** じゃ。これは**Dockerコンテナ自身のTailscale IP**であり、iPhone自身のIPに1を加えた値ではない。

## SideStore側の設定

1. iOS端末をWi-Fiへ接続し、Tailscaleを有効にする。
2. SideStore → Settings → Connection Config を開く。
3. **Use Local VPN をオフ**（Remote Endpointモード）。
4. **Device IP** にログの `reflector: node address` に表示されたIPを入力。
5. **RemotePair Port** は空欄のまま、Confirm。
6. SideStoreの Health Check で **Device Reachability** を確認し、アプリ更新を試す。

開発元はSideStore 0.7.0でRemote Endpointモードの動作を報告している。従来のLocal VPNモードは同じ設定で動くとは限らない。

## 従来構成からの移行

- `./tailscale-state` は**削除しない**。既存のTailscaleノードの認証とIPを維持するためじゃ。
- コンテナ名 `sidestore-tailscale`、ホスト名 `sidestore`、Docker BridgeとIPv6設定は従来どおり。
- `ghcr.io/xddxdd/sidestore-vpn:tailscale` のイメージは使用せず、新実装をGitHubからビルドする。
- 旧方式の `10.7.0.1/32` サブネットルートは新方式では不要。旧コンテナが作ったTailscaleのルート広告が管理画面に残っていれば、その承認を解除する。
- 新実装ではコンテナ自身のTailscale IPを宛先に使い、DNAT/SNATでiOS端末自身へ折り返す。
- `--snat-subnet-routes=false` は折り返し通信に必要。コンテナのTailscale起動オプションで指定済み。

## 再ビルド・更新

```bash
docker compose up -d --build
docker compose logs --tail=100 sidestore-vpn
```

ビルド元は新実装の `main` ブランチじゃ。ビルドにはGitへのアクセスとRustのコンパイル時間が必要になる。上流の互換性変更で不具合が出る場合はビルド対象コミットの固定を検討する。

## 接続できない場合

- **SideStore側**: Remote Endpointモード、Device IP、Wi-Fi接続とTailscale接続を確認。
- **Docker側**: `docker compose logs --tail=100 sidestore-vpn` でリフレクターとTailscaleの起動を確認。
- **ネットワーク側**: ホストへのUDP 41642の転送を設定すると、DERP経由より直接接続しやすくなる（必須ではない）。`tailscale ping <コンテナのTailscale IP>` で接続経路を確認。
- **反応が遅い場合**: SideStore Developer Options → Device (TCP) Probe Timeoutを2000msへ増やすことを開発元が推奨。

## 参考

- [新実装: shiqi917/sidestore-vpn-tailscale](https://github.com/shiqi917/sidestore-vpn-tailscale)
- [旧実装: xddxdd/sidestore-vpn](https://github.com/xddxdd/sidestore-vpn)
- [SideStore](https://github.com/SideStore/SideStore)
- [Tailscale](https://tailscale.com/)

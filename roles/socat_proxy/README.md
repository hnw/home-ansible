# Role: socat_proxy

Cloudflare Tunnel などの背後に配置し、受信した TCP 443 通信を特定のバックエンドホスト（Target）へ中継するプロキシロール。
Systemd Socket Activation を利用し、アクセスがあるまでプロセスを起動しない。
また、中継経路となる親プロキシ（Mac 等）への WOL（Wake-on-LAN）送信機能をもつ。

## Requirements

- ターゲット（Raspberry Pi）が `wakeonlan`, `socat`, `nc` をインストール可能であること。

## Role Variables

| Variable                  | Description                                    | Default / Example |
| ------------------------- | ---------------------------------------------- | ----------------- |
| `socat_proxy_wol_mac`     | （Optional） 起動させる親プロキシのMACアドレス | `00:11:22:33:44:55` |
| `socat_proxy_via_host`    | 中継する親プロキシのホスト名 （Parent Proxy）  | `proxy-gateway.lan` |
| `socat_proxy_via_port`    | 親プロキシのポート番号                         | `8888`            |
| `socat_proxy_target_host` | 最終的な接続先ホスト名 （SNI送信用）           | `upstream.example.com` |
| `socat_proxy_target_port` | 最終的な接続先ポート番号                       | `443`             |

## Verification （動作確認方法）

このプロキシは単なる TCP 土管であり、SSL 証明書を持たないため、通常の `curl` やブラウザでは証明書エラーまたは接続切断となる。
正しく検証するには、`--resolve` オプションを使用して **SNI （Server Name Indication）** を偽装する必要がある。

### コマンド例

Raspberry Pi の IP が `192.0.2.10`、ターゲットが `upstream.example.com` の場合:

```bash
# フォーマット: --resolve <TargetHost>:<Port>:<Pi_IP> <TargetUrl>
curl -v -k --resolve upstream.example.com:443:192.0.2.10 https://upstream.example.com
```

# SwitchBot照明のローカル制御調査

## 調査日

2026-10-14

## 目的

既存のSwitchBot LEDシーリングライトを実機として、自宅LAN内だけで制御できる経路があるかを確認する。実用性よりも、クラウドを使わないスマートホームの仕組みを理解することを優先する。

ATOM Liteの購入は、この調査でWi-FiまたはBLEを使ったローカルPoCの方向性を決めてから判断する。

## 公開仕様から分かること

### Wi-Fiとハブ機能

SwitchBotの公式サポートでは、シーリングライトは2.4GHz Wi-Fiに接続でき、壁スイッチがOFFになると本体が無給電になるため、ネットワーク経由の制御もできなくなると説明されている。通常モデルのハブ機能は、他のSwitchBot Bluetooth製品がSwitchBotサーバーへ接続するための経路として説明されている。

- [SwitchBot公式: シーリングライトのWi-Fi接続](https://support.switch-bot.com/hc/ja/articles/7776495040791)
- [SwitchBot公式: ハブ機能を正しく使うには](https://support.switch-bot.com/hc/ja/articles/10080910238487)
- [SwitchBot公式: シーリングライトのハブ機能](https://support.switch-bot.com/hc/ja/articles/10080840319383)
- [SwitchBot LEDシーリングライト取扱説明書（PDF）](https://m.media-amazon.com/images/I/B1uEgWu46GL.pdf)

この説明から、照明のWi-Fi接続が存在することは分かる。一方で、照明がLAN上にローカルHTTPサーバーを公開していることまでは分からない。ハブ機能も、ローカルAPIを提供する機能とは説明されていない。

### Open API

公式のSwitchBot Open APIは、APIホスト、トークン、署名、タイムスタンプ、nonceを使うクラウドAPIとして公開されている。公式ドキュメントにはシーリングライトのデバイス種別・ステータス・コマンドが追加されているが、ローカルネットワークだけで完結するAPI仕様ではない。

- [OpenWonderLabs/SwitchBotAPI](https://github.com/OpenWonderLabs/SwitchBotAPI)
- [ローカルAPI対応を求める公式リポジトリのIssue](https://github.com/OpenWonderLabs/SwitchBotAPI/issues/19)

したがって、今回のPoCではOpen APIを採用せず、ローカルWi-FiまたはBLEが実際に成立するかを先に確認する。

### BLE

公式のBLE資料には、Ceiling Lightのデバイスタイプとして `q (0x71)` と `Q (0x51)` が記載されている。また、デバイスタイプはBLEのスキャンレスポンスに含まれると説明されているため、Macの一般的なBluetooth機器一覧に人間向けの名前で表示されないことだけでは、非対応とは判断できない。

ただし、公式資料だけで照明の全コマンドペイロードを再現できるとは確認していない。BLEを採用する場合は、まず探索・接続・読み書きの各段階を分けてPoCする。

- [OpenWonderLabs/SwitchBotAPI-BLE](https://github.com/OpenWonderLabs/SwitchBotAPI-BLE)
- [OpenWonderLabs/node-switchbot](https://github.com/OpenWonderLabs/node-switchbot)
- [homebridge-switchbot: BLE対応と接続方式](https://github.com/OpenWonderLabs/homebridge-switchbot)

## 自宅ネットワークでの調査結果

### 構成

- ルーターとアクセスポイントが別の機器である。
- Macはアクセスポイント側のWi-Fiに接続していた。
- 照明はSwitchBotアプリでWi-Fi設定済みで、ユーザーが確認した照明のIPアドレスを使って到達性を調査した。
- 公開記録には、個別のIPアドレス、MACアドレス、SSID、パスワード、SwitchBotトークンを残さない。

### Mac側で確認できたこと

- MacのWi-Fiインターフェースは有効で、プライベートIPv4アドレスと `/24` 相当のネットマスクが設定されていた。
- デフォルトゲートウェイはルーター側にあり、対象アドレスへの経路は同一サブネットへの直結経路になる構成だった。
- ARP一覧にはルーターや複数の他端末が現れていたため、MacのWi-Fiインターフェース全体が停止していたわけではない。
- 照明のアドレスは、途中でARPが `incomplete` になり、最終の読み取り確認ではARPエントリ自体が確認できなかった。
- よく使われるTCPポートを少数確認した範囲では、応答は得られなかった。これはローカルHTTP APIが存在しない可能性を示す材料ではあるが、非標準ポート、UDP、BLE、クラウド経由を否定するものではない。
- mDNSの探索は実行環境側のサービス起動エラーで完了しなかったため、照明がmDNS非対応だという結論には使わない。

### 暫定結論

現時点で確定しているのは、Mac側のネットワーク設定とルーティングが大きく矛盾していないことまでである。照明との端末間通信は成立しておらず、次の候補を切り分ける必要がある。

1. アクセスポイントがブリッジ/APモードではなく、NATや別DHCPを提供している。
2. アクセスポイントのAP isolation / Client isolationが有効になっている。
3. Guest Wi-FiまたはIoT VLANに分離されている。
4. 照明がオフライン、または確認したIPアドレスから変わっている。
5. ネットワーク到達性は成立しているが、照明がローカルWi-Fi APIを提供していない。

## 次回の調査手順

1. 照明の壁スイッチをONのままにし、照明側のWi-Fi設定は変更しない。
2. Macをルーター本体の通常Wi-Fiへ接続する。Guest/IoT用SSIDは使わない。
3. MacのIPアドレス、サブネット、デフォルトゲートウェイを確認する。機微な値はリポジトリへ保存しない。
4. ルーターのDHCP接続端末一覧で、照明の現在のリース、MACアドレス、接続先を確認する。
5. アクセスポイントの管理画面で、AP/ブリッジモード、DHCP無効、端末間隔離無効、通常SSIDへの接続を確認する。
6. ルーター本体のWi-Fiから照明へ到達できるかを確認する。
7. 到達性が成立した場合だけ、ローカルサービスの有無を狭い範囲で再確認する。見つからない場合はMac上のBLE探索へ進む。

## 分岐と購入判断

| 結果 | 次の判断 |
|---|---|
| ルーター本体のWi-Fiから照明へ到達できる | アクセスポイントの隔離・VLAN・ブリッジ設定を修正候補にする |
| 照明はDHCP一覧にあるが到達できない | APの端末間隔離またはVLAN分離を優先して確認する |
| 照明がDHCP一覧にもない | SwitchBotアプリで照明のWi-Fi状態と現在IPを確認する |
| LAN到達後もローカルサービスが見つからない | Wi-Fi直結を前提にせず、BLE探索を次のPoCにする |
| MacでBLE探索・通信が成立する | ATOM LiteをBLEゲートウェイとして使う価値を評価する |
| Wi-Fi/BLEのどちらも成立しない | ATOM Lite購入前に、ローカル完結という目的と対象機器を再評価する |

## 制約と注意

- SwitchBotのクラウドOpen APIを使えば制御できる可能性はあるが、今回の目的であるローカル完結から外れるため採用しない。
- 公開仕様だけでは、通常のSwitchBotシーリングライトのローカルWi-Fiコマンドを保証できない。
- BLEは公式資料にデバイスタイプがあるが、機種・ファームウェアごとの互換性とコマンド詳細は実機で確認する。
- 実機固有の赤外線データ、IPアドレス、MACアドレス、SSID、認証情報は公開リポジトリにコミットしない。

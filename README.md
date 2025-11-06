# STM32RC - BLE Belt Conveyor Controller

STM32WB55を使用したBluetooth Low Energy（BLE）ベルトコンベアコントローラー

## 目次

- [概要](#概要)
- [主な機能](#主な機能)
- [ハードウェア要件](#ハードウェア要件)
- [ソフトウェア要件](#ソフトウェア要件)
- [BLE仕様](#ble仕様)
- [ビルド方法](#ビルド方法)
- [使用方法](#使用方法)
- [技術仕様](#技術仕様)
- [システムアーキテクチャ](#システムアーキテクチャ)
- [ディレクトリ構成](#ディレクトリ構成)
- [カスタマイズ方法](#カスタマイズ方法)
- [デバッグ方法](#デバッグ方法)
- [トラブルシューティング](#トラブルシューティング)
- [FAQ](#faq)
- [参考資料](#参考資料)
- [コントリビューション](#コントリビューション)
- [今後の拡張予定](#今後の拡張予定)
- [セキュリティ](#セキュリティ)
- [開発履歴](#開発履歴)
- [ライセンス](#ライセンス)

## 概要

このプロジェクトは、STM32WB55マイコンを使用して、左右2つのベルトコンベアをBLE経由でワイヤレス制御するシステムです。Web Bluetooth APIを使用したHTML制御画面により、スマートフォンやPCから直接操作できます。

## 主な機能

- 左右独立したベルトコンベア制御
- 前進・後進の方向制御
- 速度制御（0-255段階）
- BLE通信による低消費電力動作
- Web Bluetooth API対応のブラウザから制御可能

## ハードウェア要件

- STM32WB55CGUx マイコン
- モータードライバ
- ベルトコンベア機構（左右2系統）
- 電源回路

## ソフトウェア要件

### 開発環境
- STM32CubeIDE
- STM32CubeMX
- STM32WB Copro Wireless Binary（BLE Stack）

### 制御用ブラウザ
- Chrome、Edge、Opera など Web Bluetooth API対応ブラウザ
- iOS: Bluefy（Web Bluetooth対応ブラウザアプリ）

## BLE仕様

### サービスUUID
```
0000fe40-cc7a-482a-984a-7f2ed5b3e58f
```

### キャラクタリスティックUUID（制御用）
```
0000fe41-8e22-4541-9d4c-21edae82ed19
```

### データフォーマット
4バイトのデータで制御します：

| バイト | 内容 | 値の範囲 |
|--------|------|----------|
| 0 | 右ベルト方向 | 0: 前進, 1: 後進 |
| 1 | 右ベルト速度 | 0-255 |
| 2 | 左ベルト方向 | 0: 前進, 1: 後進 |
| 3 | 左ベルト速度 | 0-255 |

## ビルド方法

1. STM32CubeIDEでプロジェクトをインポート
2. プロジェクトをクリーンビルド
3. STM32WBの無線スタックをフラッシュ（初回のみ）
4. プログラムをフラッシュ

```bash
# STM32_Programmer_CLIを使用する場合
STM32_Programmer_CLI -c port=SWD -w build/01-BLE_Test.hex
```

## 使用方法

### 1. デバイスの起動
STM32WB55ボードに電源を供給すると、自動的にBLEアドバタイズを開始します。

デバイス名: `WeAct`

### 2. Web UIによる制御

1. `index.html`をWeb Bluetooth API対応ブラウザで開く
2. 「デバイスに接続」ボタンをクリック
3. BLEデバイス一覧から「WeAct」を選択
4. スライダーを操作してベルトコンベアを制御
   - 中央（0）: 停止
   - プラス方向: 前進（速度0-100）
   - マイナス方向: 後進（速度0-100）

### 省電力機能

アドバタイジング間隔を最適化し、低消費電力動作を実現しています。

## 技術仕様

### ハードウェアピン配置

#### モータードライバ制御（右ベルト - A系統）
- `IN_A1`: 方向制御ピン1
- `IN_A2`: 方向制御ピン2
- `TIM1_CH3`: PWM出力（速度制御）

#### モータードライバ制御（左ベルト - B系統）
- `IN_B1`: 方向制御ピン1
- `IN_B2`: 方向制御ピン2
- `TIM2_CH1`: PWM出力（速度制御）

#### その他の制御ピン
- `STANDBY`: モータードライバスタンバイ制御
- `GPIO_PE4`: LED表示用

#### センサー入力
- `ADC`: アナログ入力（将来の拡張用）
- `RTC`: リアルタイムクロック（実装済み）

### モーター制御ロジック

#### 方向制御
```
前進: IN_X1=HIGH, IN_X2=LOW
後進: IN_X1=LOW, IN_X2=HIGH
停止: 速度=0
```

#### 速度制御
- PWMデューティ比: 0-255 (8ビット)
- PWM周波数: TIM1/TIM2設定に依存
- 速度マッピング: Web UIの0-100 → STM32の0-255

### BLE通信仕様

#### アドバタイジング設定
- デバイス名: "WeAct"
- 省電力モード対応
- カスタムサービスUUID使用

#### データ転送方式
- Write Without Response方式を採用
- デバウンス処理: 10ms（Web UI側）
- リアルタイム制御に最適化

## システムアーキテクチャ

### データフロー

```
[Web Browser]
    ↓ Web Bluetooth API
    ↓ (BLE通信)
[STM32WB55 - BLE Stack]
    ↓
[p2p_server_app.c] ← BLE Write イベント受信
    ↓ 変数更新 (A_dir, A_speed, B_dir, B_speed)
[main.c - LoopMain()]
    ↓ GPIO制御 (方向)
    ↓ PWM制御 (速度)
[モータードライバ]
    ↓
[ベルトコンベア]
```

### 主要ファイル

| ファイル | 役割 |
|----------|------|
| `STM32_WPAN/App/p2p_server_app.c` | BLE通信処理、データ受信 |
| `Core/Src/main.c` | メインループ、モーター制御 |
| `Core/Src/app_entry.c` | アプリケーション初期化 |
| `STM32_WPAN/App/app_ble.c` | BLEスタック初期化 |
| `index.html` | Web制御UI |

## ディレクトリ構成

```
.
├── Core/               # メインアプリケーションコード
│   ├── Inc/           # ヘッダーファイル
│   └── Src/           # ソースファイル（main.c等）
├── Drivers/            # STM32 HALドライバ
├── Middlewares/        # STM32WB ミドルウェア
├── STM32_WPAN/         # BLEスタック関連
│   └── App/           # BLEアプリケーション層
├── Utilities/          # ユーティリティ
├── Bsp/                # ボードサポートパッケージ
├── index.html          # Web制御UI
└── 01-BLE_Test.ioc     # STM32CubeMX設定ファイル
```

## カスタマイズ方法

### BLE UUIDの変更

`index.html`とSTM32側のBLEサービス定義の両方を変更する必要があります：

```javascript
// index.html
const SERVICE_UUID = 'your-service-uuid';
const CHARACTERISTIC_UUID = 'your-characteristic-uuid';
```

### PWM周波数の変更

STM32CubeMXで`TIM1`および`TIM2`の設定を変更してください。

### デバイス名の変更

BLEアドバタイジング設定でデバイス名を変更できます：
- `STM32_WPAN/App/app_ble.c`内のデバイス名設定を編集

### 速度範囲の調整

`index.html`のマッピング関数を変更：

```javascript
function mapTo255(value) {
  return Math.min(255, Math.round((value / 100) * 255));
}
```

## 開発履歴

- 省電力アドバタイジングを実装
- 初期動作確認版
- プロジェクト初期化

## ライセンス

このプロジェクトは、STM32のサンプルコードをベースに開発されています。

## 注意事項

- Web Bluetooth APIはHTTPS環境またはlocalhostでのみ動作します
- iOS SafariはWeb Bluetooth APIに対応していません（Bluefyなどのアプリが必要）
- BLE接続が不安定な場合は、デバイスを再起動してください

## デバッグ方法

### STM32側のデバッグ

1. **SWDデバッガー接続**
   - ST-LINK/V2やJ-Linkを使用
   - STM32CubeIDEのデバッグ機能を利用

2. **変数の監視**
   - `A_dir`, `A_speed`, `B_dir`, `B_speed`の値を確認
   - `p2p_server_app.c:94-97`にブレークポイントを設置

3. **GPIO状態の確認**
   - IN_A1, IN_A2, IN_B1, IN_B2のピン状態を確認
   - PWMデューティ比を確認

### Web UI側のデバッグ

1. **ブラウザコンソールの確認**
   ```javascript
   // index.htmlを開き、F12でデベロッパーツールを起動
   // コンソールに「書き込み成功」メッセージが表示されるか確認
   ```

2. **BLE接続状態の確認**
   - `beltCharacteristic`がnullでないことを確認
   - 接続エラーメッセージを確認

## トラブルシューティング

### BLEデバイスが見つからない
- STM32WBの無線スタックが正しくフラッシュされているか確認
- デバイスがアドバタイズモードになっているか確認
- ブラウザがWeb Bluetooth APIに対応しているか確認
- Bluetoothが有効になっているか確認（OS設定）

### 接続できるが制御できない
- UUIDが正しく設定されているか確認
- キャラクタリスティックのプロパティ（Write Without Response）を確認
- Web UIのコンソールでエラーメッセージを確認
- STM32側で`P2PS_STM_WRITE_EVT`が受信されているか確認

### モーターが動かない
- モータードライバの電源を確認
- PWM出力が正しく設定されているか確認
- GPIOピンの接続を確認
- STANDBYピンがHIGHになっているか確認
- モータードライバのENABLEピンの状態を確認

### 速度制御が不安定
- デバウンス時間（10ms）が適切か確認
- PWM周波数が適切か確認
- 電源容量が十分か確認
- BLE通信の遅延を確認

## FAQ

**Q: 他のBLEデバイスと同時に使用できますか？**
A: はい、BLEは複数デバイスの接続に対応していますが、ブラウザの制限により一度に接続できるデバイス数には限りがあります。

**Q: 通信可能距離はどのくらいですか？**
A: 理論値で約10-50m（障害物なし）、実際の使用環境では5-10m程度を想定してください。

**Q: バッテリー駆動は可能ですか？**
A: 可能です。省電力モードを実装しており、適切なバッテリー容量があれば長時間動作します。

**Q: 複数のユーザーから同時に制御できますか？**
A: BLEの仕様上、1対1接続が基本ですが、複数の端末から順次接続・切断することは可能です。

**Q: Web UI以外の制御方法はありますか？**
A: Web Bluetooth APIが使える環境であれば、独自のアプリケーションを開発可能です。また、BLEに対応したPythonやNode.jsからも制御できます。

## 参考資料

### STM32WB関連
- [STM32WB55 製品ページ](https://www.st.com/ja/microcontrollers-microprocessors/stm32wb55rg.html)
- [STM32WB BLE スタックドキュメント](https://www.st.com/resource/en/user_manual/dm00550659.pdf)
- [STM32CubeWB ファームウェアパッケージ](https://github.com/STMicroelectronics/STM32CubeWB)

### Web Bluetooth API
- [Web Bluetooth API 仕様](https://webbluetoothcg.github.io/web-bluetooth/)
- [MDN Web Bluetooth API ドキュメント](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API)
- [Google Developers - Web Bluetooth](https://developers.google.com/web/updates/2015/07/interact-with-ble-devices-on-the-web)

### 開発ツール
- [STM32CubeIDE](https://www.st.com/ja/development-tools/stm32cubeide.html)
- [STM32CubeMX](https://www.st.com/ja/development-tools/stm32cubemx.html)
- [STM32CubeProgrammer](https://www.st.com/ja/development-tools/stm32cubeprog.html)

## コントリビューション

プロジェクトへの貢献を歓迎します！

### 貢献方法

1. このリポジトリをフォーク
2. 新しいブランチを作成 (`git checkout -b feature/amazing-feature`)
3. 変更をコミット (`git commit -m 'Add some amazing feature'`)
4. ブランチにプッシュ (`git push origin feature/amazing-feature`)
5. プルリクエストを作成

### コーディング規約

- STM32 HALのコーディングスタイルに準拠
- コメントは日本語または英語で記述
- 新機能追加時は動作確認を実施
- 可能な限りドキュメントも更新

### バグ報告

Issuesセクションでバグ報告を受け付けています。以下の情報を含めてください：

- 発生した問題の詳細な説明
- 再現手順
- 期待される動作
- 実際の動作
- 使用環境（ブラウザ、OS、STM32ファームウェアバージョンなど）

## 今後の拡張予定

- [ ] 速度プリセット機能
- [ ] タイマー機能（指定時間後に自動停止）
- [ ] センサーフィードバック機能
- [ ] 複数デバイスの同時制御
- [ ] スマートフォン専用アプリの開発
- [ ] ログ記録機能

## セキュリティ

このプロジェクトは認証なしのBLE通信を使用しています。産業用途や機密性の高い用途では、以下の対策を検討してください：

- BLEペアリング・ボンディングの実装
- 暗号化通信の追加
- アクセス制御の実装
- 通信範囲の物理的制限

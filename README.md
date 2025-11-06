# STM32RC - BLE Belt Conveyor Controller

STM32WB55を使用したBluetooth Low Energy（BLE）ベルトコンベアコントローラー

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

## ディレクトリ構成

```
.
├── Core/               # メインアプリケーションコード
├── Drivers/            # STM32 HALドライバ
├── Middlewares/        # STM32WB ミドルウェア
├── STM32_WPAN/         # BLEスタック関連
├── Utilities/          # ユーティリティ
├── Bsp/                # ボードサポートパッケージ
├── index.html          # Web制御UI
└── 01-BLE_Test.ioc     # STM32CubeMX設定ファイル
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

## トラブルシューティング

### BLEデバイスが見つからない
- STM32WBの無線スタックが正しくフラッシュされているか確認
- デバイスがアドバタイズモードになっているか確認
- ブラウザがWeb Bluetooth APIに対応しているか確認

### 接続できるが制御できない
- UUIDが正しく設定されているか確認
- キャラクタリスティックのプロパティ（Write Without Response）を確認

### モーターが動かない
- モータードライバの電源を確認
- PWM出力が正しく設定されているか確認
- GPIOピンの接続を確認

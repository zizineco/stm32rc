# ドメインモデル - STM32RC BLE Belt Conveyor Controller

このドキュメントでは、STM32RC BLE Belt Conveyor Controllerのドメインモデルを定義します。

## 目次

- [システムコンテキスト図](#システムコンテキスト図)
- [ドメインモデル図](#ドメインモデル図)
- [レイヤーアーキテクチャ](#レイヤーアーキテクチャ)
- [シーケンス図](#シーケンス図)
- [状態遷移図](#状態遷移図)
- [コンポーネント図](#コンポーネント図)

---

## システムコンテキスト図

システム全体の境界とアクターを示します。

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

title システムコンテキスト図 - STM32RC BLE Belt Conveyor Controller

Person(user, "ユーザー", "ベルトコンベアを操作したい人")
System(webapp, "Web UI", "Web Bluetooth APIを使用した制御画面")
System_Boundary(stm32_system, "STM32WB55システム") {
    System(stm32, "STM32WB55", "BLE対応マイコン")
}
System(motorDriver, "モータードライバ", "TB6612FNGなど")
System(conveyor, "ベルトコンベア", "左右2系統の搬送装置")

Rel(user, webapp, "スライダー操作", "タッチ/マウス")
Rel(webapp, stm32, "制御コマンド送信", "BLE")
Rel(stm32, motorDriver, "PWM/GPIO信号", "電気信号")
Rel(motorDriver, conveyor, "モーター駆動", "電力")

@enduml
```

![システムコンテキスト図](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/zizineco/stm32rc/claude/gum-dissolving-feature-011CUrnDysn79g8sZmzffeph/doc/diagrams/context.puml)

---

## ドメインモデル図

システムの中核となるドメインオブジェクトとその関係を示します。

```plantuml
@startuml
title ドメインモデル図

package "Web UI Domain" {
    class WebBrowserUI {
        + connectButton: Button
        + rightSlider: Slider
        + leftSlider: Slider
        + connect(): void
        + updateSpeed(side: string, value: int): void
    }

    class BLEController {
        - serviceUUID: string
        - characteristicUUID: string
        - beltCharacteristic: BluetoothCharacteristic
        + connect(): Promise<void>
        + sendCommand(data: Uint8Array): Promise<void>
    }
}

package "BLE Protocol" {
    class BLEService {
        + serviceUUID: 0000fe40-cc7a-482a-984a-7f2ed5b3e58f
        + deviceName: "WeAct"
    }

    class BLECharacteristic {
        + characteristicUUID: 0000fe41-8e22-4541-9d4c-21edae82ed19
        + properties: Write Without Response
        + dataFormat: [rightDir, rightSpeed, leftDir, leftSpeed]
    }
}

package "STM32 Application Domain" {
    class P2PServerApp {
        - A_dir: uint8_t
        - A_speed: uint8_t
        - B_dir: uint8_t
        - B_speed: uint8_t
        + handleWriteEvent(data: uint8_t[]): void
        + updateMotorVariables(): void
    }

    class MotorController {
        + updateDirection(motor: Motor, dir: Direction): void
        + updateSpeed(motor: Motor, speed: uint8_t): void
    }
}

package "Domain Entities" {
    class BeltConveyor {
        - side: Side
        - motor: Motor
        + setDirection(dir: Direction): void
        + setSpeed(speed: uint8_t): void
        + getStatus(): ConveyorStatus
    }

    class Motor {
        - direction: Direction
        - speed: uint8_t
        - pwmChannel: PWMChannel
        - directionPins: GPIOPins
        + forward(): void
        + backward(): void
        + stop(): void
        + setSpeed(speed: uint8_t): void
    }

    enum Direction {
        FORWARD
        BACKWARD
    }

    enum Side {
        RIGHT
        LEFT
    }

    class GPIOPins {
        - in1: GPIOPin
        - in2: GPIOPin
        + setForward(): void
        + setBackward(): void
    }

    class PWMChannel {
        - timer: Timer
        - channel: int
        - dutyCycle: uint8_t
        + setDutyCycle(value: uint8_t): void
    }
}

package "Infrastructure" {
    class GPIOHardware {
        + writePin(port: Port, pin: Pin, state: State): void
    }

    class TimerHardware {
        + setCompare(timer: Timer, channel: int, value: uint16_t): void
    }
}

' Relationships
WebBrowserUI --> BLEController: uses
BLEController --> BLEService: connects to
BLEService *-- BLECharacteristic: contains
BLECharacteristic --> P2PServerApp: writes data to
P2PServerApp --> MotorController: updates
MotorController --> BeltConveyor: controls
BeltConveyor *-- Motor: has
Motor --> Direction: has
Motor *-- GPIOPins: controls direction with
Motor *-- PWMChannel: controls speed with
BeltConveyor --> Side: identified by
GPIOPins --> GPIOHardware: uses
PWMChannel --> TimerHardware: uses

@enduml
```

![ドメインモデル図](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/zizineco/stm32rc/claude/gum-dissolving-feature-011CUrnDysn79g8sZmzffeph/doc/diagrams/domain-model.puml)

---

## レイヤーアーキテクチャ

システムのレイヤー構造を示します。

```plantuml
@startuml
title レイヤーアーキテクチャ

package "Presentation Layer" {
    [index.html]
    [Web Bluetooth API]
}

package "Communication Layer" {
    [BLE GATT Protocol]
    [STM32 BLE Stack]
}

package "Application Layer" {
    [app_ble.c]
    [p2p_server_app.c]
    [app_entry.c]
}

package "Domain Layer" {
    [Motor Control Logic]
    [Belt Conveyor Control]
}

package "Infrastructure Layer" {
    [GPIO HAL]
    [Timer HAL]
    [PWM Driver]
    [STM32WB55 Hardware]
}

[index.html] --> [Web Bluetooth API]
[Web Bluetooth API] --> [BLE GATT Protocol]
[BLE GATT Protocol] --> [STM32 BLE Stack]
[STM32 BLE Stack] --> [app_ble.c]
[app_ble.c] --> [p2p_server_app.c]
[p2p_server_app.c] --> [Motor Control Logic]
[Motor Control Logic] --> [Belt Conveyor Control]
[Belt Conveyor Control] --> [GPIO HAL]
[Belt Conveyor Control] --> [Timer HAL]
[GPIO HAL] --> [PWM Driver]
[Timer HAL] --> [PWM Driver]
[PWM Driver] --> [STM32WB55 Hardware]

@enduml
```

![レイヤーアーキテクチャ図](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/zizineco/stm32rc/claude/gum-dissolving-feature-011CUrnDysn79g8sZmzffeph/doc/diagrams/layer-architecture.puml)

---

## シーケンス図

### 1. デバイス接続シーケンス

```plantuml
@startuml
title デバイス接続シーケンス

actor User
participant "Web UI" as UI
participant "Web Bluetooth API" as BLE_API
participant "STM32 BLE Stack" as BLE_Stack
participant "app_ble.c" as AppBLE
participant "p2p_server_app.c" as P2PServer

User -> UI: 「デバイスに接続」ボタンをクリック
UI -> BLE_API: navigator.bluetooth.requestDevice()
BLE_API -> BLE_Stack: デバイススキャン
BLE_Stack --> BLE_API: デバイスリスト返却
BLE_API --> UI: デバイス選択画面表示
User -> UI: "WeAct"を選択
UI -> BLE_API: device.gatt.connect()
BLE_API -> BLE_Stack: GATT接続要求
BLE_Stack -> AppBLE: 接続イベント通知
AppBLE -> P2PServer: 接続完了通知
P2PServer --> AppBLE: 初期化完了
AppBLE --> BLE_Stack: 接続確立
BLE_Stack --> BLE_API: 接続成功
BLE_API -> BLE_Stack: getPrimaryService(SERVICE_UUID)
BLE_Stack --> BLE_API: サービス返却
BLE_API -> BLE_Stack: getCharacteristic(CHARACTERISTIC_UUID)
BLE_Stack --> BLE_API: キャラクタリスティック返却
BLE_API --> UI: 接続成功
UI --> User: 制御可能状態

@enduml
```

![デバイス接続シーケンス図](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/zizineco/stm32rc/claude/gum-dissolving-feature-011CUrnDysn79g8sZmzffeph/doc/diagrams/sequence-connection.puml)

### 2. モーター制御シーケンス

```plantuml
@startuml
title モーター制御シーケンス（スライダー操作）

actor User
participant "Web UI" as UI
participant "Web Bluetooth API" as BLE_API
participant "p2p_server_app.c" as P2PServer
participant "main.c" as Main
participant "Motor Controller" as MotorCtrl
participant "GPIO/PWM" as Hardware

User -> UI: 右スライダーを操作 (値: 50)
UI -> UI: デバウンス処理 (10ms待機)
UI -> UI: 値を変換\n(50 -> direction:0, speed:127)
UI -> BLE_API: writeValueWithoutResponse([0, 127, leftDir, leftSpeed])
BLE_API -> P2PServer: BLE Write Event
P2PServer -> P2PServer: データ受信\nA_dir = 0\nA_speed = 127
note right: STM32_WPAN/App/\np2p_server_app.c:94-97
P2PServer -> Main: 変数更新通知
Main -> Main: LoopMain() 実行
Main -> MotorCtrl: 方向制御\nIN_A1=HIGH, IN_A2=LOW
Main -> MotorCtrl: 速度制御\n__HAL_TIM_SET_COMPARE(htim1, CH3, 127)
MotorCtrl -> Hardware: GPIO書き込み
MotorCtrl -> Hardware: PWMデューティ比設定
Hardware -> Hardware: モーター駆動
Hardware --> User: ベルトコンベア動作

@enduml
```

![モーター制御シーケンス図](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/zizineco/stm32rc/claude/gum-dissolving-feature-011CUrnDysn79g8sZmzffeph/doc/diagrams/sequence-motor-control.puml)

### 3. 停止シーケンス

```plantuml
@startuml
title 停止シーケンス

actor User
participant "Web UI" as UI
participant "BLE" as BLE
participant "STM32" as STM32
participant "Motor" as Motor

User -> UI: スライダーを中央(0)に移動
UI -> UI: デバウンス (10ms)
UI -> BLE: writeValue([0, 0, leftDir, leftSpeed])
BLE -> STM32: データ受信
STM32 -> STM32: A_speed = 0
STM32 -> Motor: PWMデューティ比 = 0
Motor -> Motor: モーター停止
Motor --> User: ベルトコンベア停止

@enduml
```

![停止シーケンス図](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/zizineco/stm32rc/claude/gum-dissolving-feature-011CUrnDysn79g8sZmzffeph/doc/diagrams/sequence-stop.puml)

---

## 状態遷移図

### BLE接続状態

```plantuml
@startuml
title BLE接続状態遷移図

[*] --> Disconnected: 電源投入

Disconnected --> Advertising: アドバタイズ開始
Advertising --> Connected: 接続要求受信
Connected --> Disconnected: 切断

Connected --> DataTransfer: Write要求受信
DataTransfer --> Connected: データ処理完了

Advertising --> Advertising: タイムアウト\n(再アドバタイズ)

note right of Advertising
  デバイス名: "WeAct"
  Service UUID: 0000fe40-...
end note

note right of DataTransfer
  4バイトデータ受信:
  [rightDir, rightSpeed,
   leftDir, leftSpeed]
end note

@enduml
```

![BLE接続状態遷移図](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/zizineco/stm32rc/claude/gum-dissolving-feature-011CUrnDysn79g8sZmzffeph/doc/diagrams/state-ble.puml)

### モーター動作状態

```plantuml
@startuml
title モーター動作状態遷移図

[*] --> Stopped: 初期化

Stopped --> Forward: speed > 0, direction = 0
Stopped --> Backward: speed > 0, direction = 1
Forward --> Stopped: speed = 0
Backward --> Stopped: speed = 0
Forward --> Forward: speed変更 (direction = 0)
Backward --> Backward: speed変更 (direction = 1)
Forward --> Backward: direction = 1, speed > 0
Backward --> Forward: direction = 0, speed > 0

note right of Stopped
  IN_X1: 任意
  IN_X2: 任意
  PWM: 0
end note

note right of Forward
  IN_X1: HIGH
  IN_X2: LOW
  PWM: 0-255
end note

note right of Backward
  IN_X1: LOW
  IN_X2: HIGH
  PWM: 0-255
end note

@enduml
```

![モーター動作状態遷移図](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/zizineco/stm32rc/claude/gum-dissolving-feature-011CUrnDysn79g8sZmzffeph/doc/diagrams/state-motor.puml)

---

## コンポーネント図

### STM32アプリケーションコンポーネント

```plantuml
@startuml
title STM32アプリケーション コンポーネント図

package "STM32_WPAN" {
    [app_ble.c] as AppBLE
    [p2p_server.c] as P2PServer_Service
    [p2p_server_app.c] as P2PServer_App
}

package "Core" {
    [main.c] as Main
    [app_entry.c] as AppEntry
    [gpio.c] as GPIO
    [tim.c] as Timer
}

package "Drivers" {
    [stm32wbxx_hal_gpio.c] as HAL_GPIO
    [stm32wbxx_hal_tim.c] as HAL_TIM
}

package "Middlewares" {
    [ble_stack] as BLE_Stack
    [sequencer] as Sequencer
}

cloud "Hardware" {
    [STM32WB55] as MCU
}

' Application flow
AppEntry --> AppBLE: 初期化
AppBLE --> P2PServer_Service: サービス登録
AppBLE --> BLE_Stack: BLE初期化
P2PServer_Service --> P2PServer_App: イベント通知
P2PServer_App --> Main: 変数更新

' Main loop
Main --> GPIO: 方向制御
Main --> Timer: 速度制御 (PWM)

' HAL layer
GPIO --> HAL_GPIO: HAL API
Timer --> HAL_TIM: HAL API

' Hardware
HAL_GPIO --> MCU: レジスタ操作
HAL_TIM --> MCU: レジスタ操作

' BLE Stack
BLE_Stack --> Sequencer: タスク管理
Sequencer --> MCU: 無線制御

@enduml
```

![STM32アプリケーション コンポーネント図](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/zizineco/stm32rc/claude/gum-dissolving-feature-011CUrnDysn79g8sZmzffeph/doc/diagrams/component-stm32.puml)

### データフローコンポーネント

```plantuml
@startuml
title データフローコンポーネント図

component "Web Browser" as Browser {
    [HTML UI] as HTML
    [JavaScript] as JS
    [Web Bluetooth API] as WebBLE
}

component "BLE Protocol" as BLEProto {
    [GATT Profile] as GATT
    [ATT Protocol] as ATT
}

component "STM32WB55" as STM32 {
    component "BLE Stack" as BLEStack {
        [Link Layer] as LL
        [HCI] as HCI
        [GATT Server] as GATTServer
    }

    component "Application" as App {
        [P2P Server] as P2P
        [Motor Control] as MotorCtrl
    }

    component "HAL" as HAL {
        [GPIO Driver] as GPIODrv
        [Timer Driver] as TimerDrv
    }
}

component "Hardware" as HW {
    [Motor Driver IC] as MotorDrv
    [Belt Conveyor] as Belt
}

' Data flow
HTML --> JS: User Input
JS --> WebBLE: Control Command
WebBLE -down-> GATT: BLE Write
GATT -down-> ATT: ATT Request
ATT -down-> GATTServer: Write Request
GATTServer --> P2P: Characteristic Write
P2P --> MotorCtrl: Update Variables
MotorCtrl --> GPIODrv: Direction Control
MotorCtrl --> TimerDrv: Speed Control (PWM)
GPIODrv --> MotorDrv: GPIO Signals
TimerDrv --> MotorDrv: PWM Signals
MotorDrv --> Belt: Power

note right of GATT
  Service UUID:
  0000fe40-cc7a-482a-
  984a-7f2ed5b3e58f

  Characteristic UUID:
  0000fe41-8e22-4541-
  9d4c-21edae82ed19
end note

note right of P2P
  4 bytes data:
  [rightDir, rightSpeed,
   leftDir, leftSpeed]
end note

@enduml
```

![データフローコンポーネント図](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/zizineco/stm32rc/claude/gum-dissolving-feature-011CUrnDysn79g8sZmzffeph/doc/diagrams/component-dataflow.puml)

---

## ドメイン用語集

### エンティティ

| 用語 | 説明 |
|------|------|
| **BeltConveyor** | ベルトコンベアを表すドメインエンティティ。左右2つ存在する。 |
| **Motor** | DCモーターを制御するエンティティ。方向と速度を持つ。 |
| **Direction** | モーターの回転方向（前進/後進）を表す値オブジェクト。 |
| **Speed** | モーターの速度（0-255）を表す値オブジェクト。 |

### サービス

| 用語 | 説明 |
|------|------|
| **MotorController** | モーターを制御するドメインサービス。GPIO/PWMを操作する。 |
| **BLEController** | BLE通信を管理するアプリケーションサービス。 |
| **P2PServerApp** | BLE P2Pサービスのアプリケーション層実装。 |

### 値オブジェクト

| 用語 | 説明 |
|------|------|
| **PWMDutyCycle** | PWMのデューティ比（0-255）。 |
| **GPIOState** | GPIOピンの状態（HIGH/LOW）。 |
| **BLECharacteristicData** | BLEで送信される4バイトのデータ。 |

### インフラストラクチャ

| 用語 | 説明 |
|------|------|
| **GPIOHardware** | STM32のGPIOペリフェラルを操作するインフラ層。 |
| **TimerHardware** | STM32のTimerペリフェラルを操作するインフラ層。 |
| **BLEStack** | STM32WB BLEプロトコルスタック。 |

---

## ユビキタス言語（Ubiquitous Language）

プロジェクト内で共通に使用される用語の定義：

- **ベルトコンベア (Belt Conveyor)**: 物体を搬送する装置。左右2系統存在する。
- **右ベルト / A系統**: 右側のベルトコンベア。TIM1_CH3でPWM制御。
- **左ベルト / B系統**: 左側のベルトコンベア。TIM2_CH1でPWM制御。
- **前進 (Forward)**: ベルトが順方向に動作する状態。direction = 0。
- **後進 (Backward)**: ベルトが逆方向に動作する状態。direction = 1。
- **速度 (Speed)**: モーターの回転速度。0（停止）〜255（最大速度）。
- **デバウンス (Debounce)**: 連続した入力を一定時間後に1回だけ処理する仕組み。Web UI側で10ms。
- **Write Without Response**: BLE通信方式の一つ。応答を待たずに書き込む。リアルタイム制御に適している。
- **P2P (Peer to Peer)**: STM32WBのBLEサンプルアプリケーション名。カスタムサービスを実装する際のベース。
- **STANDBY**: モータードライバのスタンバイ制御ピン。HIGHで動作可能。
- **IN_X1, IN_X2**: モータードライバの方向制御ピン。組み合わせで前進/後進を切り替え。

---

## アーキテクチャ上の制約

### 技術的制約

1. **BLE通信**
   - 1対1接続のみサポート
   - データサイズ: 4バイト固定
   - 通信方式: Write Without Response

2. **PWM制御**
   - 分解能: 8ビット (0-255)
   - 使用タイマー: TIM1_CH3 (右), TIM2_CH1 (左)

3. **GPIO制御**
   - 方向制御: 2ピン/モーター
   - スタンバイ制御: 共通1ピン

### ビジネス制約

1. **安全性**
   - 接続切断時は自動的に停止すべき（要実装）
   - 最大速度制限が必要な場合がある

2. **リアルタイム性**
   - スライダー操作からモーター動作まで100ms以内の応答が望ましい

3. **消費電力**
   - BLEアドバタイジング間隔を調整して省電力動作を実現

---

## まとめ

このドメインモデルは、STM32RC BLE Belt Conveyor Controllerの以下の側面を明確にします：

1. **責務の分離**: プレゼンテーション層、アプリケーション層、ドメイン層、インフラ層の明確な分離
2. **通信フロー**: Web UIからハードウェアまでのデータフロー
3. **ドメインロジック**: ベルトコンベアとモーター制御の中核ロジック
4. **状態管理**: BLE接続とモーター動作の状態遷移
5. **コンポーネント依存関係**: 各モジュール間の依存関係

このモデルを基に、機能追加や修正を行う際の指針として活用してください。

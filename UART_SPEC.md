# UART I/O Specification

このドキュメントは、[src/main.c](/Users/jinaoyagi/workspace/ares/basestation-downlink/src/main.c) に実装されている UART 入出力仕様を整理したものです

## 概要

- `USART1` は XBee からの受信入力として使用する
- `USART2` は rover 向けの送信出力として使用する
- `USART6` は ARM 向け `PacketJF` の送信出力として使用する
- `USART3` は初期化されるが、現状の `main.c` では送受信処理に使用していない

## UART 割り当て

| UART | 論理名 | 方向 | 用途 | ボーレート |
| --- | --- | --- | --- | --- |
| `USART1` | `XBEE_UART` | 入力 | XBee からの受信ストリーム | `57600` |
| `USART2` | `ROVER_UART` | 出力 | rover 向けデータ送信 | `115200` |
| `USART3` | なし | 未使用 | 初期化のみ | `57600` |
| `USART6` | `ARM_PACKET_JF_UART` | 出力 | ARM 向け `PacketJF` 送信 | `115200` |

すべての UART 設定は以下で共通です

- データ長: `8bit`
- ストップビット: `1`
- パリティ: `None`
- フロー制御: `None`

## 受信仕様

- 受信元は `XBEE_UART` のみ
- `HAL_UART_Receive_IT(&XBEE_UART, &rx_char, 1)` により 1 バイトずつ割り込み受信する
- 受信完了時は `HAL_UART_RxCpltCallback()` から `FilterXBeeByte()` を呼び出してストリームを解析する

## データ振り分け仕様

XBee から受信したデータは、以下の 2 種類として扱う

1. rover 向けのテキスト形式
2. ARM 向けの `PacketJF` 固定長バイナリ形式

### 1. rover 向け既存形式

- 改行終端のテキストデータとして扱う
- `'\r'` は無視する
- `'\n'` を受信した時点で 1 パケット完了とみなす
- 完了したデータは `SendRoverPacket()` で `USART2` に送信する
- 送信時は受信した本文の後ろに `"\r\n"` を付加する
- バッファ上限は `64` バイト
- 上限を超えた場合は、その rover パケットの蓄積を破棄して先頭からやり直す

### 2. ARM 向け `PacketJF`

- `0x4A 0x46` (`'J' 'F'`) をヘッダとして検出する
- ヘッダ検出後は合計 `16` バイトを `PacketJF` として受信する
- CRC 対象はオフセット `0` から `13` までの `14` バイト
- CRC は `CRC16-CCITT-FALSE`
- CRC が正しい場合は `SendArmPacketJf()` で `USART6` にそのまま送信する
- 送信時に改行や追加整形は行わない

## `PacketJF` 判定仕様

- ヘッダ:
  `0x4A 0x46` (`'J' 'F'`)
- 全長:
  `16` バイト
- CRC:
  `CRC16-CCITT-FALSE`
- 初期値:
  `0xFFFF`
- 多項式:
  `0x1021`
- 反転:
  なし
- Final XOR:
  なし

## `PacketJF` 判定失敗時の扱い

- `'J' 'F'` で始まっていても CRC が不正な場合は、有効な ARM パケットとしては扱わない
- `printf()` で不正な `PacketJF` としてログ出力する
- ログには受信した CRC 値と計算した CRC 値を 16 進数で出力する
- 不正な `PacketJF` は UART へ転送せず破棄する

## 送信仕様

### `ROVER_UART` への送信

- 送信関数: `SendRoverPacket()`
- 送信先: `USART2`
- データ形式:
  rover 本文 + `"\r\n"`
- 送信 API:
  `HAL_UART_Transmit()`

### `ARM_PACKET_JF_UART` への送信

- 送信関数: `SendArmPacketJf()`
- 送信先: `USART6`
- データ形式:
  `PacketJF` 16 バイトをそのまま送信
- 送信 API:
  `HAL_UART_Transmit()`

## 状態管理

- `xbee_rx_mode`
  現在の XBee 受信解析状態を保持する
- `XBEE_RX_MODE_ROVER`
  rover 向け既存形式を受信中
- `XBEE_RX_MODE_ARM`
  ARM 向け `PacketJF` を受信中
- `rover_pending_j`
  `'J'` を 1 バイト先読みし、次バイトが `'F'` かどうかを判定するために使用する

## 補足

- `main.c` 内では `USART1`, `USART2`, `USART6` のみが実際のデータ経路に関与する
- `USART3` は初期化されているが、現時点では入出力先として未接続
- `printf()` の出力先は `src/printf.c` の `_write()` 実装に従い ITM であり、UART には出力されない

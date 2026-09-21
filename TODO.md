# 残件

moNa2 側の未確認・未着手事項。片付いたら消すこと。

**現状: v0.4 移行はビルドまで完了。実機確認の途中。**

---

## A. いますぐ確認すること（焼いた直後に見る）

### A-1. トラックボールの軸の向き ✅ 実機確認済み

`config/mona2_r.conf`:

```conf
CONFIG_PMW3610_INVERT_X=n
CONFIG_PMW3610_INVERT_Y=y
```

**実機でカーソルの向きが正しいことを確認済み。**

チェーン上の `&zip_xy_transform (X_INVERT | Y_INVERT)` が X も Y も反転させるので、
ドライバの Kconfig はその打ち消しとして働く。正味の向きは

    X = transform(反転) XOR INVERT_X
    Y = transform(反転) XOR INVERT_Y

まだ逆なら **DYA Studio のセンサータブから実行時に変更できる**（意図的に
ドライバの Kconfig 側に置いてある）。正しい組み合わせが分かったら conf に反映すること。

- 両方まだ逆 → `INVERT_X=y` / `INVERT_Y=n` に戻す
- 片方だけ逆 → その軸だけ反転

### A-2. スクロールが効くか ✅ 動作確認済み（縦の向きも修正）

include の重複を直してある。**スクロールレイヤー（index 5 = `NUM-SCROLL`）**で
ボールを転がしてスクロールするか。到達は `&lt 5 ENTER` または MOUSE レイヤーの `&mo 5`。

慣性スクロールとスクロールスナップも、スクロールが動いて初めて効く。

### A-3. ランタイムマクロのスロット0を記録する

このキーは **`&rmacro 0`**（ランタイムマクロのスロット0）に割り当ててある。
**DYA Studio のランタイムマクロパネルでスロット0に記録するまで、
そのキーは何もしない。**

マクロの中身はリポジトリに置かず、キーボードの Flash に持たせる方式。
書き換えは Studio から行う。

---

## B. reset で消えたので設定し直しが必要

`mona2_reset` を焼いた時点で Flash の保存値が全部消えている。

- BLE のペアリング
- **スクロール倍率**（DYA Studio の入力プロセッサパネル、`scroll` の multiplier / divisor）。
  拡大縮小の調整に効く。既定は 1/60
- ジェスチャーの RPC 設定（`&mg_set 0..2` に割り当てる内容）
- トラックボールのセンサー設定（CPI など）

**これらはリポジトリに残らない。** `settings_reset` するたびに消える。

---

## C. 未確認の機能

| 項目 | 見かた |
|---|---|
| iPad レイヤー + OS 判定 | BLE で iPad に繋ぎ、DYA Studio の「OS ごとのデフォルトレイヤー」で index 1 の `iPad` を割り当てる。修飾キーの入れ替えと英数/かなが効くか |
| BLE の安定性 | `ZMK_BLE_EXPERIMENTAL_CONN` / `BT_CTLR_PHY_2M=n` / `BT_GATT_ENFORCE_SUBSCRIPTION=n` を **v0.4 で新たに有効化**した。動き出しがもたつくなら `BT_PERIPHERAL_PREF_MIN_INT/MAX_INT`（コメントアウト中）を戻す |
| **ヘッドセットの通話遅延** | `BT_CTLR_PHY_2M=n` は v0.3 ではコメントアウトされていた。電波占有時間に効くので、**v0.3 より悪化していないか要確認**。悪化していたらこの行を消すのが最初の一手 |
| ジェスチャー | DT 6セット（レイヤー 6, 8〜12）と RPC 3セット（`&mg_set 0..2`） |
| DYA Studio の各タブ | トラックボールセンサー / kscan 診断 / ランタイムマクロ・コンボ |

---

## D. 実機 OK になってからやること

### D-1. `main` ブランチを作る

torabo と同じ運用にする。`main` = いま焼いてあるもの。焼く uf2 は必ず `main` の
artifacts から取る。**ファームには config リポジトリのバージョンが埋まらない**ので、
実験ブランチを焼いて気に入ったらその場で `main` を進めること。後から確認する手段は無い。

### D-2. 慣性スクロール — torabo でやった調整は反映されていない

モジュールは両機とも `mjmjm0101/zmk-input-processor-scroll-inertia`（`revision: main`）で同じ。
**基本の値は torabo から移してあるが、その後 torabo で入れた多段減衰は入っていない。**

差分はちょうど6プロパティ（`boards/shields/mona2/mona2_r.overlay`）。

| プロパティ | torabo `main` | moNa2 | 既定 |
|---|---|---|---|
| `fast` | 250 | **未設定** | 0（＝多段オフ） |
| `decay-fast` | 998 | **未設定** | 990 |
| `decay-slow` | 988 | **未設定** | 990 |
| `slow` | 60 | **未設定** | 0（＝テールゾーンオフ） |
| `decay-tail` | 985 | **未設定** | 990 |
| `span` | 12000 | **未設定** | 6000 |

その他（`axis` `layer` `scale` `scale-div` `tick` `release` `start` `move`
`min-events` `gain` `blend` `friction` `limit` `stop`）は完全に一致している。

**つまり moNa2 は「全速度域が同じ減衰率で落ちる」状態。** 弾きの強弱で転がり方が変わらない。

#### そのまま移してはいけない理由

`fast` と `slow` は**速度の絶対値**。両機はセンサーも CPI も違う。

| | センサー | CPI | レート |
|---|---|---|---|
| torabo | PAW3222 | 未設定（センサー既定のまま） | 約67Hz |
| moNa2 | PMW3610 | **600** | 約67Hz |

同じ転がしでも速度の数値が変わるので、`fast = 250` の判定ラインがずれる。
**まず素の慣性の感触を見てから、moNa2 用に別ブランチを切って実機で詰めること。**

#### 詰めるときに知っておくこと

以下は torabo の `TODO.md` の「6. 慣性スクロール」に詳細がある。要点だけ再掲。

- **速度の「増幅」はできない。** `decay-*` は減衰率（千分率/tick、常に 1000 未満）で、
  速度に応じて倍率を上げる仕組みは存在しない。できるのは
  「速い間は減りにくくする」＝伸びる、という形だけ
- **「ゆっくりは細かく、弾いたら強く」**を作るには、
  スクロール倍率（Studio）を下げて、**慣性の `scale` / `scale-div` を上げる**。
  慣性の `scale` は `accumulate_and_emit()` 経由で**転がり出力にだけ**掛かり、
  アクティブスクロールには掛からない。torabo でも未実施
- **モジュールの既定値は 1000 CPI / 125Hz 前提**（binding に明記）。
  こちらは約67Hz、`tick = 17`（既定 8）なので、実時間あたりの減衰が既定の感覚と違う。
  990 なら1秒で ×0.55（既定 tick=8 なら ×0.29）
- **未使用の機能**: `swap-mod` / `unlock-mod`（1レイヤーのまま修飾キーで軸切替）、
  `decel-samples` / `decel-ratio` / `peak-decay`（弾き終わりの検出）、`track-remainders`
- **軸ロックの思想が逆**: 作者は自動判定を否定しているが、こちらは
  `axis = <0>` + kot149 の `zip_scroll_snap`（＝自動判定）という構成。
  斜めや切り返しで引っかかるならここ

---

---

## E. 記録

### このリポジトリには2つの系統があり、合流していない

設定が「本家にはあるのにこちらに無い」と見えたとき、まずこれを疑うこと。

```
公開側   … 20abb82 (5/1) → dya-studio (8/16) → my-dya-studio (8/18)
              ╲
               ╲ 合流していない
                ╲
v0.4 の中身  … v0.3 系 (4月) ──────────→ v0.4 移行 (9/13〜) → いまの main
```

v0.4 移行は**稼働していた v0.3 系を土台に組み直した**もので、公開側の
`dya-studio` / `my-dya-studio`（2026年8月の線）は取り込んでいない。
そのため8月の線にだけ入った設定は「消えた」のではなく**最初から継いでいない**。

実例として、次の3つがこの経緯で欠けていた（2026-09-21 に補った）。

| 設定 | 側 |
|---|---|
| `CONFIG_EC11_TRIGGER_OWN_THREAD` ＋ `_THREAD_PRIORITY` / `_THREAD_STACK_SIZE` | 左 |
| `CONFIG_ZMK_WATCHDOG` | 左 |
| `CONFIG_USB_HID_POLL_INTERVAL_MS` | 右 |

一方 `CONFIG_BT_PERIPHERAL_PREF_MIN_INT` / `MAX_INT` は**意図して外したもの**。
`CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y` が接続パラメータを自前で扱うため重複する。
本家は EXPERIMENTAL_CONN を使わず明示指定する方式なので、そこだけ設計が違う。
**戻さないこと。**

本家 main は現在 `dya-studio` を統合済みで、それ以降はドキュメントのみ。
モジュール構成も追跡先も一致しているので、取り込むべき機能差は無い。

### v0.3 から引き継いだ値で、根拠を残してあるもの

- `CONFIG_PMW3610_REPORT_INTERVAL_MIN=15` — **上げないこと。**
  レートを上げると Bluetooth ヘッドセットの通話に遅延が出る。
  torabo の PAW3222 が 15ms 固定（`src/paw3222.c:451`）なのに合わせてある
- `CONFIG_ZMK_SPLIT_BLE_CENTRAL_SPLIT_RUN_STACK_SIZE=3096` — **根拠なし。**
  ZMK 既定 512 / 参考リポジトリ未設定 / torabo 768 に対して6倍。
  RAM が足りなくなったら真っ先にここを削ること

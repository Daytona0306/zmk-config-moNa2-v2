# moNa2 v2 用 ZMK ファームウェア（DYA Studio 構成）

ZMK v0.4 系（Zephyr 4.1 / HWMv2）で、DYA Studio の機能を一通り有効化した構成です。

* `mona2_right` がトラックボール側（central）、`mona2_left` が反対側
* v0.3 から上げる場合は、先に `mona2_reset` を左右に焼いてください。設定の保存形式が変わっています
* キーマップは keymap-editor および ZMK Studio / DYA Studio で編集できます

---

## モジュールのリビジョンは SHA で固定しています

`config/west.yml` の `revision:` はブランチ名ではなく SHA です。追跡先は
行末の `# track: <ref>` コメントに残してあります。

```yaml
- name: zmk
  remote: cormoran
  revision: e5c9b6915b56801193e359dd9bad4a167ce0d1b8  # track: main+dya
```

### なぜ固定するのか

浮動のままだと、**自分では何も変えていないのにビルドが落ちます。** 上流の
モジュールが動くと `west update` が別物を引いてくるためです。加えて

* GitHub Actions の **artifact は 90 日で消える**
* ファームには config のバージョンが埋まらない

この2つが重なると、「いま快調に使っているファーム」を二度と作れなくなります。
同じコミットをビルドしても結果が変わるからです。

### 更新のしかた

```sh
tools/west-pins.py            # 追跡先の先頭に固定し直す
tools/west-pins.py --unpin    # ブランチ名に戻す
tools/west-pins.py --check    # 追跡先が進んでいるか見るだけ（書き換えない）
```

上流に追随したくなったら、`--unpin` した枝で CI を通してから、`main` 側で
`tools/west-pins.py` を実行して SHA を進めます。**浮動の manifest を
`main` にマージしないこと。**

## レイヤー構成

| index | ノード | Studio 表示名 | 用途 |
|---|---|---|---|
| 0 | `default_layer` | Base | ベース（Windows） |
| 1 | `iPad` | iPad | ベース（iPad / iOS）。`default-layer` が OS 判定で有効化 |
| 2 | `NUM,SYM1` | NUM/SYM1 | 数字・記号 |
| 3 | `SYM2` | SYM2 | 記号2。`&msc` の 5倍速スクロール対象 |
| 4 | `MOUSE` | Mouse | マウス |
| 5 | `NUM-SCROLL` | NUM/Scroll | スクロール（慣性・スナップの対象） |
| 6 | `gest_arrow` | Gest: Arrow | ジェスチャー（矢印）＋ F キー |
| 7 | `bluetooth` | Bluetooth | BLE プロファイル操作 |
| 8〜12 | `gest_*` | Gest: … | ジェスチャー（タブ / 仮想デスクトップ / ナビ / ウィンドウ / PowerToys） |

iPad レイヤーは Windows との**差分だけ**を書いてあり、残りは `&trans` でレイヤー0に落ちます。

| | Windows（レイヤー0） | iPad（レイヤー1） |
|---|---|---|
| 修飾キー | Ctrl / Win | **入れ替え**（コピペ等を Cmd 側へ） |
| IME 切替 | 無変換 / 変換 | **英数（LANG2）/ かな（LANG1）** |

**iPad レイヤーの置き場所は index 1 でなければいけません。** ZMK は有効なレイヤーを
高い index から順に見るため、末尾に置くと機能レイヤーをすべて上書きしてしまいます。

**レイヤー番号は複数のファイルに散らばっています。** 順序を変えるときは以下すべてを追従させてください。

* `config/mona2.keymap` の `&lt` / `&mo` / `&to`、および `&msc_input_listener` の `layers`
* `config/gestures.dtsi` の `GESTURE_ROUTE(N, ...)`
* `boards/shields/mona2/mona2.dtsi` の `active-layers = <BIT(N)>`
* `boards/shields/mona2/mona2_r.overlay` の慣性スクロールの `layer`
* `config/mona2_r.conf` の `CONFIG_ZMK_DEFAULT_LAYER_MIN/MAX_INDEX`

---

## トラックボールの向きを変えたいとき

**devicetree ではなく Kconfig で指定します。** ZMK v0.4 で
`cormoran/zmk-driver-pmw3610-with-custom-studio-rpc` に移行したためです
（Zephyr 4.1 が純正の `pixart,pmw3610` を取り込んだので、旧 compatible はそのまま使えません）。

`config/mona2_r.conf`:

```conf
CONFIG_PMW3610_INVERT_X=y
CONFIG_PMW3610_INVERT_Y=y
CONFIG_PMW3610_SWAP_XY=y
```

**COROPIT 版を使う場合は `_INVERT_X` と `_INVERT_Y` の両方を `y` にしてください。**
以前の README にあった `mona2_r.overlay` の `invert-x;` / `invert-y;` は、
このドライバでは効きません（プロパティ自体が無くなっています）。

再ビルドせずに DYA Studio の「トラックボールセンサー」タブから実行時に変更することもできます。

---

## エンコーダーの設定を DYA Studio で変えられなかった理由（解決済み）

かつて「キーマップタブ下のロータリーエンコーダー設定で保存すると、その層の
エンコーダが無反応になる」という不具合がありました。`config/mona2_r.conf` の

```conf
CONFIG_ZMK_BEHAVIOR_LOCAL_IDS_IN_BINDINGS=y
```

で直っています。**今は DYA Studio から自由に設定して構いません。**

### 何が起きていたのか

`zmk-behavior-runtime-sensor-rotate` は、Studio が保存した binding を
「behavior local id」という数値で持ち、実行時に behavior 名へ引き直します。
その引き直しが `#if` で囲まれていました
（`src/behaviors/behavior_runtime_sensor_rotate.c`）。

```c
const char *behavior_name = NULL;

if (triggered_binding_data.behavior_local_id == 0) {
    /* Studio で未設定 → DT に書いた既定を使う。ここは #if の外 */
    behavior_name = config->default_cw_binding_name;
} else {
    /* Studio で設定済み → local_id から behavior 名を引く */
#if IS_ENABLED(CONFIG_ZMK_BEHAVIOR_LOCAL_IDS_IN_BINDINGS)
    behavior_name =
        zmk_behavior_find_behavior_name_from_local_id(triggered_binding_data.behavior_local_id);
#endif
    if (!behavior_name) {
        LOG_ERR("Failed to find behavior for local_id %d", ...);
        return ZMK_BEHAVIOR_TRANSPARENT;   /* ← 何も出さずに終わる */
    }
}
```

このシンボルが `n` だと `#if` の中身がプリプロセッサごと消え、`behavior_name`
は `NULL` のまま `ZMK_BEHAVIOR_TRANSPARENT` が返ります。

`CONFIG_ZMK_BEHAVIOR_LOCAL_ID_TYPE_SETTINGS_TABLE` 方式は Kconfig で
`select ZMK_BEHAVIOR_LOCAL_IDS_IN_BINDINGS` しているため自動で `y` になりますが、
この構成が使っている `CONFIG_ZMK_BEHAVIOR_LOCAL_ID_TYPE_CRC16` は
`select CRC` しかしていないので、明示しないと `n` のままでした。

症状が全部これで説明できます。

| 症状 | 理由 |
|---|---|
| 設定する前は普通に効く | `local_id` が 0 なので上の分岐に入り、DT の既定が使われる |
| 設定した瞬間に死ぬ | `local_id != 0` になり `else` 側へ。引き直しが消えているので `NULL` |
| **そのレイヤーだけ**死ぬ | 保存が `global_data.bindings[sensor][layer]` とレイヤーごとだから |
| 焼き直しても直らなかった | Flash の `local_id` が残り続けるため。`mona2_reset` で 0 に戻ると復活していた |

モジュール作者自身のキーボードでは手当てされています
（`cormoran/zmk-keyboard-dya-dash` の `dya_dash_right.conf` と
`dya_dash_v3_right.conf`、どちらも `CRC16=y` の直後の行）。
モジュールの `Kconfig` に `select` が無いので、モジュールだけ持ってきた構成が踏みます。

`config/mona2_r.conf`（central 側）に書くこと。`mona2_l.conf` に書いても
無警告で無視されます。Flash の保存形式はこのシンボルの有無で変わらないので、
**`mona2_reset` は不要**です。すでに死んでいる割当も焼き直すだけで復活します。

---

## エンコーダーのスクロール量の決め方

`config/mona2.keymap` の `ZMK_POINTING_DEFAULT_SCRL_VAL` で決まります。
**これはカウント数ではなく「速さ」**なので、そのままの数字は出てきません。

```
1クリックで出るカウント = floor(SCRL_VAL × trigger_period_ms / 1000)
                        = floor(SCRL_VAL × 0.016)      ← trigger_period_ms の既定は 16
```

`&msc`（`zmk,behavior-input-two-axis`）が、押している間 16ms ごとに刻んで
出力するためです（`behavior_input_two_axis.c` の `update_movement_1d`）。

さらに `CONFIG_ZMK_POINTING_SMOOTH_SCROLLING=y` にしていると、HID の
Resolution Multiplier（`hid.h`: logical 0〜15 / physical 1〜16）をホストが
最大 **16** に設定するので、ホスト側で **1カウント = 1/16 ノッチ**になります。

ZMK 側で補正するはずの `input_listener.c` の `apply_resolution_scaling` は、
`scaled` を計算したあと `evt->value = val` としていて**割った結果を捨てています**
（本家 ZMK main も同じコード）。つまりファームは生値を送り、ホストだけが 16 で割ります。

| `SCRL_VAL` | 出るカウント | 体感 |
|---|---|---|
| 100 | 1 | 1/16 ノッチ。16クリックで1行 |
| 200 | 3 | 3/16 ノッチ。5〜6クリックで1行 |
| **1000**（現在） | **16** | **1クリック = 1ノッチ** |

`1000 × 0.016 = 16.0` ちょうどなので、切り捨ての端数も出ません。
スムーススクロールを `n` に戻す場合は、`SCRL_VAL` も 100 に戻さないと 16倍速になります。

### `tap-ms` は 16 にしないこと

`scroll_up_down` / `scroll_right_left` の `tap-ms` は **24** にしてあります。
`runtime-sensor-rotate` は `tap_ms` だけ押して離しますが、`&msc` は押下時には
何も出さず `trigger_period_ms`（16ms）後の最初のティックで初めて出力します。
`tap-ms = 16` だと「ティック」と「離してキャンセル」が同時刻になり、
**クリックが丸ごと無視されることがあります**。24 なら必ず1ティックだけ出ます
（次のティックは 32ms なので2回出ることもない）。

`rsr_vol`（音量、`tap-ms = 5`）は `&kp` 直結で `&msc` を通らないので対象外です。

### どの設定がどちらに効くか

エンコーダとトラックボールは経路が別です。

| 変えるもの | エンコーダ | トラックボール |
|---|---|---|
| `ZMK_POINTING_DEFAULT_SCRL_VAL` | 効く | 効かない |
| `&msc` の `time-to-max-speed-ms` など | 効く | 効かない |
| `scroll_up_down` の `tap-ms` | 効く | 効かない |
| `zip_wheel_scaler` の `scale-multiplier` / `scale-divisor` | 効かない | 効く |
| `CONFIG_ZMK_POINTING_SMOOTH_SCROLLING` | 効く | 効く |

`ZMK_POINTING_DEFAULT_SCRL_VAL` は `SCRL_UP` / `SCRL_DOWN` / `SCRL_LEFT` /
`SCRL_RIGHT` というマクロの中身でしかなく、それを使うのは `&msc` の引数だけです。
トラックボールはセンサーから直接 REL イベントが出て `zip_wheel_scaler` を通るので、
`&msc` も `SCRL_*` も経由しません。

SYM2 レイヤーには `&msc_input_listener` 側で `&zip_wheel_scaler 5 1` が掛かっており、
そこだけエンコーダが 5倍（5ノッチ/クリック）になります。掛け算なので
`SCRL_VAL` を変えても比率は保たれます。

`CONFIG_ZMK_POINTING_SMOOTH_SCROLLING` は `input_listener.c` の
`input_handler` の中で `INPUT_REL_WHEEL` / `INPUT_REL_HWHEEL` に一律で掛かるため、
どのリスナー経由でも効きます。**この設定だけは両方に影響します。**
HID のレポートマップが変わるので、切り替えるとペアリング済みホストとの
再ペアリングが必要になることがあります（ホスト側からデバイスを削除して繋ぎ直す。
`mona2_reset` は不要です）。

---

## 左右で揃えないといけない設定

`config/mona2_r.conf` と `config/mona2_l.conf` の両方に必要です。片側だけだとビルドが落ちます。

```conf
CONFIG_ZMK_SPLIT_RELAY_EVENT_DATA_LEN=256
CONFIG_ZMK_KSCAN_DIAGNOSTICS=y
CONFIG_ZMK_KSCAN_DIAGNOSTICS_MAX_POSITIONS=48
```

kscan 診断の peripheral 応答（`struct ksd_relay_reply`）が ZMK 既定の 128 バイトに収まらず、
`BUILD_ASSERT` で落ちます。モジュール側も Kconfig の `default` で 256 にしていますが、
ZMK 本体の `default` に parse 順で負けることがあるため明示する必要があります。

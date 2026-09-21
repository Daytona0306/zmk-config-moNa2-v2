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

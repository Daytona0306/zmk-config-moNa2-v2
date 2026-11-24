# Ctrl / Cmd + ボールのズームを1ノッチ単位にまとめる入力プロセッサ。
# torabo-tsuki LP (Daytona0306/zmk-keyboard-torabo-tsuki-lp) の src/input_processor_zoom_step.c と同じもの。
target_sources_ifdef(CONFIG_ZMK_INPUT_PROCESSOR_ZOOM_STEP app PRIVATE src/input_processor_zoom_step.c)

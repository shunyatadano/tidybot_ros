# 環境構築ガイド (Native Ubuntu + NVIDIA GPU)

このドキュメントはネイティブ Ubuntu (24.04) + NVIDIA GPU 環境における tidybot_ros のセットアップ手順をまとめたもの。
WSL2 環境は [`setup_wsl2.md`](./setup_wsl2.md) を参照。

**WSL2 との主な違い:**

| 項目 | Native Ubuntu | WSL2 |
|---|---|---|
| NVIDIA ドライバ | Ubuntu 側でインストール必要 | Windows 側のドライバを自動引き継ぎ |
| GUI (Gazebo/RViz) | デスクトップ環境がそのまま使える | WSLg が必要 (Win11 推奨) |
| Vulkan ICD | `/usr/share/vulkan/icd.d` が存在する | 存在しないことがある (run.sh がスキップ) |
| canivore-usb-kernel | コンテナ内でインストール可能 | カーネル不一致でインストール失敗 |

---

## Step 0: NVIDIA ドライバのインストール

WSL2 では不要だが、Native Ubuntu では GPU ドライバを先にインストールする。

```bash
# 推奨ドライバを自動選択してインストール
sudo ubuntu-drivers autoinstall
sudo reboot
```

再起動後に確認:

```bash
nvidia-smi
```

ドライババージョンを指定したい場合:

```bash
sudo apt install nvidia-driver-570  # バージョンは適宜変更
```

---

## Step 1〜4: upstream 追加 / Docker / udev / イメージビルド

WSL2 ガイドと手順は共通。[`setup_wsl2.md`](./setup_wsl2.md) の Step 1〜4 を参照。

---

## Step 5: コンテナ起動 (Native Ubuntu 固有の注意)

```bash
./docker/tidybot/run.sh restart
```

Native Ubuntu では `DISPLAY` 変数と X 認証がデスクトップセッションから自動で引き継がれるため、
Gazebo/RViz は特別な設定なしに起動する。

もし Gazebo が開かない場合のみ:

```bash
xhost +SI:localuser:$(whoami)
```

Wayland セッションを使っている場合は XWayland 経由で動作する。通常は `DISPLAY=:1` などが自動設定されている。

---

## Step 5 続き: canivore-usb のインストール

Native Ubuntu では **コンテナ内で正常にインストールできる** (WSL2 では失敗するが、こちらは問題なし)。

コンテナ起動後、コンテナ内で:

```bash
sudo apt update && sudo apt install canivore-usb -y
```

カーネルモジュールのビルドに Linux ヘッダーが必要な場合:

```bash
# ホスト側で事前にインストールしておく
sudo apt install linux-headers-$(uname -r)
```

実機を使う場合は CAN バスの設定も必要:

```bash
# ホスト側で実行
sudo bash ./scripts/setup_docker_can.sh
```

---

## Step 5 続き: Python 仮想環境 / OpenCV / Kortex API

[`setup_wsl2.md` の Step 5](./setup_wsl2.md#step-5-コンテナ内環境セットアップ) と手順は共通。
同じコマンドをコンテナ内で実行する。

---

## 動作確認チェックリスト

- [ ] `nvidia-smi` でドライバが認識されている
- [ ] `git remote -v` に `upstream` が表示される
- [ ] `docker info` の `Runtimes` に `nvidia` が含まれる
- [ ] `docker images` に `tidybot_platform:latest` が存在する
- [ ] コンテナ内で `ros2 topic list` が通る
- [ ] Gazebo + RViz が起動する

日常的な使い方は [`usage.md`](./usage.md) を参照。

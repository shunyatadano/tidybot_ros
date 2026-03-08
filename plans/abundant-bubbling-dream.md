# 環境構築プラン: tidybot_ros (WSL2 + NVIDIA GPU + フォーク運用)

## Context

- ユーザーはリポジトリをフォーク済み (`origin` = 自分のフォーク)
- Quick Start の `git clone` 完了後の状態からスタート
- 主目的: シミュレーション (Gazebo / Isaac Sim) での検証。実機はほかチームが担当
- 環境: WSL2 (Ubuntu) + NVIDIA GPU
- フォーク元 upstream を追跡できる状態にしておきたい

---

## Step 1: upstream リモートを追加

```bash
# フォーク元を upstream として登録
git remote add upstream https://github.com/roahmlab/tidybot_platform.git

# 確認
git remote -v
```

これにより、本家の変更を `git fetch upstream && git merge upstream/main` で取り込める。

---

## Step 2: WSL2 上の Docker + NVIDIA 環境を確認

```bash
# Docker が動作しているか確認
docker info

# NVIDIA Container Toolkit が入っているか確認
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.0-base-ubuntu24.04 nvidia-smi
```

**WSL2 固有の注意点:**
- `nvidia-container-toolkit` が必要 (`sudo apt install nvidia-container-toolkit`)
- Docker Desktop ではなく Docker Engine (CE) を推奨 (GPU パススルーの安定性)
- `/usr/share/vulkan/icd.d` がホスト側に存在しない場合、run.sh の Vulkan マウントが警告を出すが無視してよい (Gazebo 動作には影響小)

---

## Step 3: Docker イメージのビルド

```bash
cd /home/shunya/tidybot_ros
./docker/tidybot/build.sh
```

- ベースイメージ: Ubuntu 24.04
- ROS 2 Jazzy + MoveIt2 + Gazebo Harmonic が含まれる
- Orbbec SDK を vcstool で取得してビルド (ネット接続が必要)
- ビルドには 20〜40分程度かかる見込み

---

## Step 4: udev ルールのインストール (将来の実機対応向けにホスト側で実施)

```bash
# ホスト側 (WSL2 側) で実行
sudo bash ./scripts/install_udev_rules.sh
sudo udevadm control --reload-rules && sudo udevadm trigger
```

シミュレーションのみなら今は不要だが、将来の実機対応チームのためにホスト側に入れておく。

---

## Step 5: Docker コンテナを起動

```bash
./docker/tidybot/run.sh restart
```

**WSL2 での GUI (RViz / Gazebo) 表示:**
- WSLg が使えるなら DISPLAY は自動設定される (Windows 11 推奨)
- Windows 10 の場合は VcXsrv 等の X サーバーが別途必要

コンテナ内に入ったことを確認後、以降はコンテナ内で実行。

---

## Step 6: コンテナ内環境セットアップ

```bash
# CANivore USBツールのインストール (将来の実機用; シミュのみでも入れておく)
sudo apt update && sudo apt install canivore-usb -y

# Python venv 作成 (colcon ビルドと共存させるため --system-site-packages)
python3 -m venv --system-site-packages venv
source venv/bin/activate

# Python 依存ライブラリをインストール
pip install -r requirements.txt

# OpenCV (Dropbox からのカスタムビルド wheel) をインストール
wget -O opencv_python-4.9.0.80-cp312-cp312-linux_x86_64.whl "<README記載のDropbox URL>"
pip install --force-reinstall --no-deps opencv_python-4.9.0.80-cp312-cp312-linux_x86_64.whl
rm opencv_python-4.9.0.80-cp312-cp312-linux_x86_64.whl

# Kortex API (Kinova アーム用) のインストール
wget https://artifactory.kinovaapps.com:443/artifactory/generic-public/kortex/API/2.6.0/kortex_api-2.6.0.post3-py3-none-any.whl
pip install ./kortex_api-2.6.0.post3-py3-none-any.whl
pip install protobuf==3.20.0
rm ./kortex_api-2.6.0.post3-py3-none-any.whl

# venv を colcon に無視させる
touch venv/COLCON_IGNORE
```

---

## Step 7: ROS 2 ワークスペースのビルド

```bash
# ワークスペースルートで実行
colcon build
source install/setup.bash

# Python パスを通す (venv の site-packages を優先)
export PYTHONPATH=$PWD/venv/lib/python3.12/site-packages:$PYTHONPATH
```

永続設定 (コンテナ内 ~/.bashrc):
```bash
echo 'source /home/<USER>/tidybot_platform/install/setup.bash' >> ~/.bashrc
echo 'export PYTHONPATH=$HOME/tidybot_platform/venv/lib/python3.12/site-packages:$PYTHONPATH' >> ~/.bashrc
```

---

## Step 8: シミュレーション動作確認

### Gazebo Harmonic

```bash
# 速度制御モードで起動
ros2 launch tidybot_description launch_sim_robot.launch.py base_mode:=velocity
```

### Isaac Sim (将来対応)

```bash
# TidyBot コンテナとは別に Isaac Sim コンテナを起動
./docker/isaac-sim-ros2/run.sh
# コンテナ内: ros2 launch tidybot_description launch_isaac_sim.launch.py use_velocity_control:=true
```

---

## 重要ファイル一覧

| ファイル | 用途 |
|---|---|
| `docker/tidybot/build.sh` | イメージビルド |
| `docker/tidybot/run.sh` | コンテナ起動・アタッチ |
| `docker/tidybot/Dockerfile` | Ubuntu 24.04 + ROS 2 Jazzy + MoveIt2 + Gazebo |
| `docker/fastdds.xml` | コンテナ間 DDS 通信設定 |
| `requirements.txt` | Python 依存ライブラリ |
| `scripts/install_udev_rules.sh` | 実機用 udev 設定 |
| `src/tidybot_description/launch/launch_sim_robot.launch.py` | Gazebo 起動 |

---

## Verification

1. `git remote -v` で `upstream` が追加されているか確認
2. `docker images` で `tidybot_platform:latest` が存在するか確認
3. `colcon build` が全パッケージ成功するか確認
4. `ros2 launch tidybot_description launch_sim_robot.launch.py base_mode:=velocity` で Gazebo + RViz が起動するか確認
5. RViz でロボットモデルが表示され、joint state が流れているか確認

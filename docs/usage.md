# 日常的な使い方

## コンテナの起動・アタッチ

### 初回 or 再起動

```bash
cd ~/tidybot_ros
./docker/tidybot/run.sh restart
```

### 既存コンテナに入る (2つ目のターミナルを開く場合など)

```bash
./docker/tidybot/run.sh
```

コンテナが動いていればアタッチ、止まっていれば再開してアタッチ。

---

## シミュレーション (Gazebo)

```bash
# コンテナ内で実行
ros2 launch tidybot_description launch_sim_robot.launch.py base_mode:=velocity
```

WSLg が有効であれば Gazebo と RViz のウィンドウが自動で開く。

### Gazebo が起動しない場合

ホスト側で以下を実行してから `run.sh restart`:

```bash
xhost +SI:localuser:$(whoami)
xhost +SI:localuser:root
export DISPLAY=${DISPLAY:-:0}   # Wayland 環境の場合
```

### 動作確認 (別ターミナルのコンテナ内)

```bash
ros2 topic list
ros2 topic echo /joint_states --once
```

---

## シミュレーション (Isaac Sim)

TidyBot コンテナとは**別に** Isaac Sim コンテナを起動する。

```bash
./docker/isaac-sim-ros2/run.sh
# コンテナ内:
ros2 launch tidybot_description launch_isaac_sim.launch.py use_velocity_control:=true
```

DDS 通信は `docker/fastdds.xml` の設定で自動的に通る (`--net=host`)。

---

## コントロールモード

### ゲームパッド (Xbox Series X)

```bash
ros2 launch tidybot_policy launch_gamepad_policy.launch.py sim_mode:=gazebo
# sim_mode は gazebo / isaac / hardware から選択
```

### スマホ WebXR テレオペ

```bash
ros2 launch tidybot_policy launch_phone_policy.launch.py sim_mode:=gazebo
```

ポート 5000 でサーバーが立ち上がる。同一ネットワークのスマホからアクセス。

---

## デモ収集 & データ変換

### 収録付きテレオペ起動

```bash
ros2 launch tidybot_policy launch_phone_policy.launch.py sim_mode:=gazebo record:=true
```

`episode_bag/` に ROS bag が保存される。

### HDF5 変換

```bash
ros2 run tidybot_episode rosbag_to_hdf5
```

`data.hdf5` が生成される。

---

## コンテナのパス構成

| パス (コンテナ内) | 内容 |
|---|---|
| `~/tidybot_platform/` | ビルド済みワークスペース (イメージに焼き込まれている) |
| `~/Documents/tidybot_platform/` | ホストのリポジトリのマウント (ライブ編集可) |
| `~/tidybot_platform/venv/` | Python 仮想環境 |
| `~/tidybot_platform/install/` | colcon ビルド成果物 |

> ソースを編集する場合は `~/Documents/tidybot_platform/src/` 以下を変更し、
> `~/tidybot_platform/` で `colcon build` を実行する。

---

## よくあるトラブル

### `permission denied` (Docker socket)

```bash
newgrp docker
# または再ログイン
```

### `apt-get update` 失敗 (ビルド時)

リポジトリの一時的な不整合。そのままリビルドすればキャッシュが効いて通ることが多い。

### `canivore-usb-kernel` インストール失敗

WSL2 環境ではカーネルモジュールが使えないため既知の問題。シミュ用途では無視してよい。

### ログ確認

```bash
~/.ros/log/                  # ROS ログ
docker logs tidybot_platform # コンテナログ
```

---

## 設定ファイル早見表

| ファイル | 用途 |
|---|---|
| `docker/tidybot/build.sh` | イメージビルド |
| `docker/tidybot/run.sh` | コンテナ起動・アタッチ |
| `docker/tidybot/Dockerfile` | イメージ定義 |
| `docker/fastdds.xml` | コンテナ間 DDS 設定 |
| `src/tidybot_description/config/tidybot_controllers.yaml` | コントローラ設定 |
| `src/tidybot_moveit_config/config/joint_limits.yaml` | 関節リミット |

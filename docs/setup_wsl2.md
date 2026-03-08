# 環境構築ガイド (WSL2 + NVIDIA GPU)

このドキュメントは WSL2 (Ubuntu 24.04) + NVIDIA GPU 環境における tidybot_ros の初回セットアップ手順をまとめたもの。
実機担当チームへの引き継ぎ時にも参照してほしい。

## 前提条件

- Windows 10/11 上の WSL2 (Ubuntu 24.04 noble)
- NVIDIA GPU (動作確認済み: RTX 5060, Driver 572.97, CUDA 12.8)
- Windows 11 + WSLg 推奨 (Gazebo/RViz の GUI 表示に必要)
- リポジトリをフォーク済みで `git clone` 完了済みの状態からスタート

---

## Step 1: upstream リモートを追加

フォーク元 (本家) を追跡できるようにする。

```bash
git remote add upstream https://github.com/roahmlab/tidybot_platform.git
git remote -v
```

本家の変更を取り込む際は:

```bash
git fetch upstream && git merge upstream/main
```

---

## Step 2: Docker Engine + NVIDIA Container Toolkit のインストール

### Docker Engine (CE)

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

### NVIDIA Container Toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### グループ設定の反映

```bash
newgrp docker
# または一度ログアウト & 再ログイン
```

### 動作確認

```bash
docker info  # Runtimes に "nvidia" が含まれていれば OK
nvidia-smi
```

> **注意:** `docker run --rm --gpus all nvidia/cuda:12.0-base-ubuntu24.04 nvidia-smi` は該当タグが存在しないためエラーになるが、
> `docker info` で nvidia runtime が確認できていれば問題なし。

---

## Step 3: udev ルールのインストール (ホスト側)

実機対応チームのためにホスト側 (WSL2 側) に入れておく。

```bash
sudo bash ./scripts/install_udev_rules.sh
sudo udevadm control --reload-rules && sudo udevadm trigger
```

シミュレーションのみの場合は後回しでも可。

---

## Step 4: Docker イメージのビルド

> 20〜40 分かかる。`docker` グループが有効なターミナルで実行すること。

```bash
./docker/tidybot/build.sh 2>&1 | tee /tmp/docker_build.log
```

ビルドが途中で `apt-get update` 失敗した場合はリトライすれば解消することが多い (キャッシュが効くので途中から再開される)。

完了確認:

```bash
docker images | grep tidybot_platform
```

---

## Step 5: コンテナ内環境セットアップ

### コンテナを起動

```bash
./docker/tidybot/run.sh restart
```

以降はコンテナ内で実行する。

### canivore-usb (実機用・シミュはスキップ可)

```bash
sudo apt update && sudo apt install canivore-usb -y
```

> **WSL2 注意:** `canivore-usb-kernel` はカーネルモジュールのため WSL2 コンテナ内では
> インストールに失敗する。シミュレーション用途では無視してよい。実機使用時は別途検討が必要。

### Python 仮想環境の作成

uv を使う場合 (推奨):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

uv venv --system-site-packages venv
source venv/bin/activate
uv pip install -r requirements.txt
```

標準 venv を使う場合:

```bash
python3 -m venv --system-site-packages venv
source venv/bin/activate
pip install -r requirements.txt
```

### OpenCV (カスタムビルド wheel)

**wget と pip は別コマンドで実行すること (改行で分割されるとエラーになる)。**

```bash
wget -O opencv_python-4.9.0.80-cp312-cp312-linux_x86_64.whl "https://www.dropbox.com/scl/fi/mzfz0i7qljmwzm5bm22td/opencv_python-4.9.0.80-cp312-cp312-linux_x86_64.whl?rlkey=rbero1ycahdk1d6jeixiu5p5l&st=k5dzfv6a&dl=0"
```

```bash
uv pip install --reinstall --no-deps opencv_python-4.9.0.80-cp312-cp312-linux_x86_64.whl && rm opencv_python-4.9.0.80-cp312-cp312-linux_x86_64.whl
```

### Kortex API (Kinova アーム用)

```bash
wget https://artifactory.kinovaapps.com:443/artifactory/generic-public/kortex/API/2.6.0/kortex_api-2.6.0.post3-py3-none-any.whl
```

```bash
uv pip install ./kortex_api-2.6.0.post3-py3-none-any.whl && uv pip install protobuf==3.20.0 && rm ./kortex_api-2.6.0.post3-py3-none-any.whl
```

> wget が `.whl.1` として保存した場合は `mv` でリネームしてから実行:
> ```bash
> mv kortex_api-2.6.0.post3-py3-none-any.whl.1 kortex_api-2.6.0.post3-py3-none-any.whl
> ```

### 仕上げ

```bash
touch venv/COLCON_IGNORE

echo 'source /home/shunya/tidybot_platform/install/setup.bash' >> ~/.bashrc
echo 'export PYTHONPATH=$HOME/tidybot_platform/venv/lib/python3.12/site-packages:$PYTHONPATH' >> ~/.bashrc
source ~/.bashrc
```

> `source ~/.bashrc` 後に venv の `(venv)` プレフィックスが消えるのは正常。
> 必要なら `source venv/bin/activate` で再度有効化する。

---

## 動作確認チェックリスト

- [ ] `git remote -v` に `upstream` が表示される
- [ ] `docker info` の `Runtimes` に `nvidia` が含まれる
- [ ] `docker images` に `tidybot_platform:latest` が存在する
- [ ] コンテナ内で `ros2 topic list` が通る
- [ ] Gazebo + RViz が起動する (次ドキュメント参照)

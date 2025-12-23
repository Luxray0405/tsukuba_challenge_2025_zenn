---
title: "SLAMをする-つくばチャレンジ備忘録04"
emoji: "🦾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["つくばチャレンジ", "機械", "ソフトウェア"]
published: false # false: 下書き / true: 公開
---

# SLAMとは

# 3D SLAMのOSS
FAST-LIO
GLIM
DLIO

# GLIM

## 導入方法
基本的な導入方法は[公式ドキュメント](https://koide3.github.io/glim/installation.html)に従ってください．また，[Abudoriさんの記事](https://www.abudorilab.com/entry/2024/08/10/192221)も非常に丁寧に書かれており参考になります．一応，自分がインストールした手順も書いておきます．（公式ドキュメントとほぼ同じ）
### 依存関係のインストール
```md:依存関係
sudo apt install libomp-dev libboost-all-dev libmetis-dev \
                 libfmt-dev libspdlog-dev \
                 libglm-dev libglfw3-dev libpng-dev libjpeg-dev
```

```md:GTSAMのインストール
git clone https://github.com/borglab/gtsam
cd gtsam && git checkout 4.3a0
mkdir build && cd build
cmake .. -DGTSAM_BUILD_EXAMPLES_ALWAYS=OFF \
         -DGTSAM_BUILD_TESTS=OFF \
         -DGTSAM_WITH_TBB=OFF \
         -DGTSAM_USE_SYSTEM_EIGEN=ON \
         -DGTSAM_BUILD_WITH_MARCH_NATIVE=OFF
make -j$(nproc)
sudo make install
```

```md:Iridescenceのインストール
mkdir iridescence/build && cd iridescence/build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
sudo make install
```

```md:gtsam_pointsのインストール
git clone https://github.com/koide3/gtsam_points
mkdir gtsam_points/build && cd gtsam_points/build
cmake .. -DBUILD_WITH_CUDA=ON
make -j$(nproc)
sudo make install

sudo ldconfig
```

### ROS2パッケージのインストール
```
cd ~/ros2_ws/src
git clone https://github.com/koide3/glim
git clone https://github.com/koide3/glim_ros2

cd ~/ros2_ws
colcon build --symlink-install
```
`--symlink-install`オプションをつけると，ビルド時に可能な限りシンボリックリンクが作成されるようになります．パラメータファイルなどを変更した際にいちいちビルドしなおさなくてよくなるため毎回つけるのを推奨します．
## 使用方法
こちらも基本的には[公式ドキュメント](https://koide3.github.io/glim/quickstart.html)に準拠しています．
### 準備・設定
1. センサデータの準備
   まずは，3D-LiDARとIMUのメッセージを記録したrosbagを用意します．LiDARやIMUの設定方法については[こちら]()を参照ください．
2. パラメータの設定
glim/config/下の設定ファイルを編集します．
必ず設定が必要なのが，`config.json`，`config_sensors.json`，`config_ros.json`の3つです．
config.jsonは，"どのパラメータファイルを使用するか"の設定を行います．GPU有/無，IMUデータ有/無によって設定が変わります．自分の環境ではGPUなし・IMUありで行っています．
:::details GPUなし・IMUあり
```json:config.json
 "config_odometry": "config_odometry_cpu.json",
 "config_sub_mapping": "config_sub_mapping_passthrough.json",
 "config_global_mapping": "config_global_mapping_pose_graph.json",
```
:::

:::details GPUあり・IMUあり
```json:config.json
 "config_odometry": "config_odometry_gpu.json",
 "config_sub_mapping": "config_sub_mapping_gpu.json",
 "config_global_mapping": "config_global_mapping_gpu.json",
```
:::

config_ros.jsonでは，LiDAR・IMUメッセージに関する設定を行います．
変更すべき箇所は"frame_idの設定"と"topic名の設定"です．
まず，frame_idです．config_ros.jsonの中の，`imu_frame_id`，`lidar_frame_id`を使用しているデータに合わせてください．
```json:config_ros.json
    "imu_frame_id": "imu",
    "lidar_frame_id": "velodyne",
```
frame_idがわからない場合，該当のセンサデータが流れている状態で以下を実行すると確認できます．
```
ros2 topic echo /velodyne_points --field header.frame_id
# velodyne_points は確認したいトピック名に置き換える
```
topic名の設定についても行います．
```json:config_ros.json
"imu_topic": "/imu",
"points_topic": "/velodyne_points",
```

### 実行
いよいよGLIMで地図作成を行います．
実行方法は2つあり，1つは`glim_rosnode`，もう一つは`glim_rosbag`です．
`glim_rosnode`では，現在流れているセンサデータをもとにSLAMを行います．最初に`glim_rosnode`を実行します．
```
ros2 run glim_ros glim_rosnode
```
別のターミナルで，rosbagを再生します．/your_bag_pathを自身のrosbagファイルのパスに書き換えてください．
```
ros2 bag play /your_bag_path
```
再生を始めると，GLIMのGUI上でmap作成が始まります．ここで何も表示されない場合，トピック名などの設定が間違っている可能性が高いです．

rosbagの再生が終了したら，glim_rosnodeを実行しているターミナルでCtrl+Cを押して中断してください．ターミナルに`saved`と表示されたら保存がされています．それ以降の手順は`glim_rosbag`の解説後にまとめて説明します．

次に，`glim_rosbag`の使用方法を解説します．こちらは，センサデータが入っているrosbagを起動時に指定します．
```
ros2 run glim_ros glim_rosbag /your_bag_path
```
これだけで実行可能な上，rosbagの再生速度を適切に早めてくれます．GPUなしの自分の環境でも3~4倍速で再生してくれました．
再生が終了したら，同様にCtrl+Cを押して中断してください．

`glim_rosnode`，もしくは`glim_rosbag`の実行が終了すると，`/tmp/dump`にそのデータが保存されます．これは毎回上書きされてしまうので，ホームディレクトリ下などに避難させておきます．
```
cp -r /tmp/dump ~/map_data/tsukuba_map
```

次に，このdumpデータを用いてマップを出力します．まず，以下を実行して`offline_viewer`を開きます．

```
ros2 run glim_ros offline_viewer
```
GUIが立ち上がります．左上の`File`
## 


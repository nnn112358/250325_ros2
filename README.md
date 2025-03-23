# 250325_ros2

ROS2（Robot Operating System 2）のインストール手順をご案内いたします。以下に、Ubuntu 22.04 LHSをベースにしたROS2のインストール方法を示します。

# ROS2のインストール手順

## 1. ロケールの設定

```bash
sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
```

## 2. ROS2リポジトリの追加

```bash
sudo apt install software-properties-common
sudo add-apt-repository universe
```

## 3. GPGキーの追加

```bash
sudo apt update && sudo apt install curl
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
```

## 4. リポジトリをソースリストに追加

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

## 5. パッケージリストの更新

```bash
sudo apt update
```

## 6. ROS2パッケージのインストール

デスクトップインストール（推奨：RVizやその他のツールを含む）:
```bash
sudo apt install ros-humble-desktop
```

ベースインストール（GUIツールなし）:
```bash
sudo apt install ros-humble-ros-base
```

## 7. 依存関係のインストール

```bash
sudo apt install python3-rosdep python3-colcon-common-extensions
sudo rosdep init
rosdep update
```

## 8. 環境設定

ROS2の環境をセットアップするには、新しいターミナルを開くたびに以下のコマンドを実行するか、`.bashrc`に追加します:

```bash
source /opt/ros/humble/setup.bash
```

`.bashrc`に追加する場合:
```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

## 9. 動作確認

インストールを確認するには、新しいターミナルを開いて以下のコマンドを実行してください:

```bash
ros2 topic list
```

これでROS2のインストールは完了です。

注意: 上記の手順はROS2 Humbleディストリビューション向けです。別のディストリビューション（例：Foxy、Iron）をインストールする場合は、`humble`を該当するディストリビューション名に置き換えてください。​​​​​​​​​​​​​​​​


ROS2でTF（Transform）をPythonから操作する方法をご説明します。TFは、ROS2において座標変換を扱うライブラリで、ロボットの各パーツや周囲の環境の座標系を管理するのに使用されます。

## 1. 必要なパッケージのインストール

まず、TF2関連のパッケージをインストールします：

```bash
sudo apt install ros-humble-tf2-ros ros-humble-tf2-tools ros-humble-tf-transformations
```

Pythonでの使用のために、以下のパッケージも必要です：

```bash
pip install transforms3d numpy
```

## 2. 基本的なTF2ブロードキャスター（静的変換を送信）

以下は、静的な座標変換（Static Transform）を発行するシンプルなPythonスクリプトの例です：

```python
#!/usr/bin/env python3

import rclpy
from rclpy.node import Node
from geometry_msgs.msg import TransformStamped
import tf2_ros
import tf_transformations
from math import radians

class StaticFramePublisher(Node):
    def __init__(self):
        super().__init__('static_tf2_broadcaster')
        
        # TFブロードキャスターの作成
        self.tf_broadcaster = tf2_ros.StaticTransformBroadcaster(self)
        
        # 変換メッセージの作成
        static_transformStamped = TransformStamped()
        
        # ヘッダー情報の設定
        static_transformStamped.header.stamp = self.get_clock().now().to_msg()
        static_transformStamped.header.frame_id = 'world'
        static_transformStamped.child_frame_id = 'robot_base'
        
        # 移動量の設定（x, y, z）
        static_transformStamped.transform.translation.x = 1.0
        static_transformStamped.transform.translation.y = 0.0
        static_transformStamped.transform.translation.z = 0.0
        
        # 回転量の設定（クォータニオン）
        quat = tf_transformations.quaternion_from_euler(0.0, 0.0, radians(45.0))
        static_transformStamped.transform.rotation.x = quat[0]
        static_transformStamped.transform.rotation.y = quat[1]
        static_transformStamped.transform.rotation.z = quat[2]
        static_transformStamped.transform.rotation.w = quat[3]
        
        # TF発行
        self.tf_broadcaster.sendTransform(static_transformStamped)
        self.get_logger().info('Static transform published from world to robot_base')

def main():
    rclpy.init()
    node = StaticFramePublisher()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## 3. 動的なTF2ブロードキャスター（時間とともに変化する変換）

こちらは、時間とともに変化する動的な座標変換を発行する例です：

```python
#!/usr/bin/env python3

import rclpy
from rclpy.node import Node
from geometry_msgs.msg import TransformStamped
import tf2_ros
import tf_transformations
import math

class DynamicFramePublisher(Node):
    def __init__(self):
        super().__init__('dynamic_tf2_broadcaster')
        
        # TFブロードキャスターの作成
        self.tf_broadcaster = tf2_ros.TransformBroadcaster(self)
        
        # タイマーによる定期的なTF発行（10Hz）
        self.timer = self.create_timer(0.1, self.broadcast_timer_callback)
        
        # 初期角度
        self.theta = 0.0

    def broadcast_timer_callback(self):
        # 時間とともに変化する変換を作成
        t = TransformStamped()
        
        # 現在時刻の取得
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'world'
        t.child_frame_id = 'moving_frame'
        
        # 円運動する位置の計算
        t.transform.translation.x = 2.0 * math.cos(self.theta)
        t.transform.translation.y = 2.0 * math.sin(self.theta)
        t.transform.translation.z = 0.0
        
        # 回転量の設定
        q = tf_transformations.quaternion_from_euler(0, 0, self.theta)
        t.transform.rotation.x = q[0]
        t.transform.rotation.y = q[1]
        t.transform.rotation.z = q[2]
        t.transform.rotation.w = q[3]
        
        # TF発行
        self.tf_broadcaster.sendTransform(t)
        
        # 角度の更新
        self.theta += 0.1

def main():
    rclpy.init()
    node = DynamicFramePublisher()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## 4. TF2リスナー（変換を受信して使用）

以下は、TFデータを受信して利用する例です：

```python
#!/usr/bin/env python3

import rclpy
from rclpy.node import Node
from tf2_ros import TransformException
from tf2_ros.buffer import Buffer
from tf2_ros.transform_listener import TransformListener
from geometry_msgs.msg import PointStamped

class FrameListener(Node):
    def __init__(self):
        super().__init__('tf2_listener')
        
        # TFバッファとリスナーの作成
        self.tf_buffer = Buffer()
        self.tf_listener = TransformListener(self.tf_buffer, self)
        
        # タイマーによる定期的な変換の検索
        self.timer = self.create_timer(1.0, self.on_timer)
        
        # 変換元と変換先のフレームID
        self.from_frame = 'world'
        self.to_frame = 'moving_frame'

    def on_timer(self):
        # PointStampedメッセージの作成（座標変換のテスト用）
        p = PointStamped()
        p.header.frame_id = self.from_frame
        p.header.stamp = self.get_clock().now().to_msg()
        p.point.x = 1.0
        p.point.y = 0.0
        p.point.z = 0.0
        
        try:
            # 座標変換を取得
            trans = self.tf_buffer.lookup_transform(
                self.to_frame,
                self.from_frame,
                rclpy.time.Time())
            
            self.get_logger().info(
                f'Transform from {self.from_frame} to {self.to_frame}:\n'
                f'Translation: [{trans.transform.translation.x}, '
                f'{trans.transform.translation.y}, {trans.transform.translation.z}]\n'
                f'Rotation: [{trans.transform.rotation.x}, {trans.transform.rotation.y}, '
                f'{trans.transform.rotation.z}, {trans.transform.rotation.w}]')
                
        except TransformException as ex:
            self.get_logger().info(f'Could not transform: {ex}')

def main():
    rclpy.init()
    node = FrameListener()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## 5. ROS2パッケージとして実行する場合

これらのスクリプトをROS2パッケージとして実行するには：

1. パッケージを作成：

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python my_tf2_examples --dependencies rclpy tf2_ros geometry_msgs
```

2. Pythonスクリプトを配置：
   - 上記スクリプトを`~/ros2_ws/src/my_tf2_examples/my_tf2_examples/`ディレクトリに保存
   - スクリプトに実行権限を付与：`chmod +x スクリプト名.py`

3. `setup.py`にエントリポイントを追加：

```python
entry_points={
    'console_scripts': [
        'static_broadcaster = my_tf2_examples.static_broadcaster:main',
        'dynamic_broadcaster = my_tf2_examples.dynamic_broadcaster:main',
        'tf_listener = my_tf2_examples.tf_listener:main',
    ],
},
```

4. ビルドと実行：

```bash
cd ~/ros2_ws
colcon build --packages-select my_tf2_examples
source install/setup.bash

# 静的TFの実行
ros2 run my_tf2_examples static_broadcaster

# 動的TFの実行
ros2 run my_tf2_examples dynamic_broadcaster

# TFリスナーの実行（別ターミナルで）
ros2 run my_tf2_examples tf_listener
```

## 6. TFを可視化する

TF2の変換を視覚的に確認するには、RViz2を使用します：

```bash
ros2 run rviz2 rviz2
```

RViz2を起動したら、「Add」ボタンから「TF」を選択して追加すると、座標系の関係が可視化されます。

これらの例を参考にして、ROS2でのTF2の基本的な使い方をマスターしていただければと思います。​​​​​​​​​​​​​​​​
　　

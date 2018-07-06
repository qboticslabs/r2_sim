## What's R2?

The r2 robot, which is an acronym for *Robonaut 2*, is a dexterous humanoid robot created by NASA, with the goal of it aiding humans in space (Robonaut 2 is just an upgrade from its predecessor, the Robonaut). The robot was created to ressemble as close as a human (with a head, hands, fingers, etc), it will be used to expand our ability for construction and discovery.

See `NASA-R2-poster.pdf` for details.

## Preview

See `r2_sim/video/`  for details.

## Installation


安装r2 gazebo model和r2 controler
```
mkdir -p ~/r2_sim/src
git clone -b indigo https://bitbucket.org/nasa_ros_pkg/deprecated_nasa_r2_simulator.git
git clone -b indigo https://bitbucket.org/nasa_ros_pkg/deprecated_nasa_r2_common.git
cd ~/r2_sim
catkin_make
```

## start

在simulator中载入r2模型，并启动

```
cd ~/r2_sim
. devel/setup.bash
roslaunch r2_gazebo r2_gazebo.launch
```

此时，在终端输入`rqt_graph ` 显示如下：

![img1](img/r2.svg)

这里，节点robot_state_publisher的作用是进行前向运动学计算，也就是说，它使用R2的几何描述和它的关节状态向量持续地计算并更新机器人上的所有坐标系。这种操作到标准ros实现是不依赖于机器人的，直接启动它，它就能为R2做正确到事情。

接下来，启动moveit：

```
cd ~/r2_sim
. devel/setup.bash
roslaunch r2_moveit_config move_group.launch
```

此时，在终端输入`rqt_graph ` 显示如下：

![img1](img/r2_move_group.svg)




```
cd ~/r2_sim
mkdir my_r2
. devel/setup.bash
roslaunch r2_moveit_config move_group.launch
```

## Simulation of NASA Robonaut 2

The following codes are available in `my_r2` package.

### 让R2随机挥动手臂

运行`r2_mime.py`，利用moveit运动规划系统让R2随机挥动手臂

```
r2_mime.py

#!/usr/bin/env python
import sys, rospy, tf, moveit_commander, random
from geometry_msgs.msg import Pose, Point, Quaternion

orient = [Quaternion(*tf.transformations.quaternion_from_euler(3.14, -1.5, -1.57)),
          Quaternion(*tf.transformations.quaternion_from_euler(3.14, -1.5, -1.57))] # <1>
pose = [Pose(Point( 0.5, -0.5, 1.3), orient[0]),
        Pose(Point(-0.5, -0.5, 1.3), orient[1])] # <2>
moveit_commander.roscpp_initialize(sys.argv) # <3>
rospy.init_node('r2_wave_arm',anonymous=True)
group = [moveit_commander.MoveGroupCommander("left_arm"),
         moveit_commander.MoveGroupCommander("right_arm")]
# now, wave arms around randomly 
while not rospy.is_shutdown():
  pose[0].position.x =  0.5 + random.uniform(-0.1, 0.1)
  pose[1].position.x = -0.5 + random.uniform(-0.1, 0.1)
  for side in [0,1]:
    pose[side].position.z =  1.5 + random.uniform(-0.1, 0.1)
    group[side].set_pose_target(pose[side])
    group[side].go(True)

moveit_commander.roscpp_shutdown()
```

### 在棋盘上方移动R2的手臂

```
#!/usr/bin/env python
import sys, rospy, tf, moveit_commander, random
from geometry_msgs.msg import Pose, Point, Quaternion

class R2ChessboardWrapper:
  def __init__(self):
    self.left_arm = moveit_commander.MoveGroupCommander("left_arm")

  def setPose(self, x, y, z, phi, theta, psi):
    orient = \
      Quaternion(*tf.transformations.quaternion_from_euler(phi, theta, psi))
    pose = Pose(Point(x, y, z), orient)
    self.left_arm.set_pose_target(pose)
    self.left_arm.go(True)

  def setSquare(self, square, height_above_board):
    if len(square) != 2 or not square[1].isdigit():
      raise ValueError(
        "expected a chess rank and file like 'b3' but found %s instead" %
        square)
    rank_y = -0.3 - 0.05 * (ord(square[0]) - ord('a'))
    file_x =  0.5 - 0.05 * int(square[1])
    z = float(height_above_board) + 1.0
    self.setPose(file_x, rank_y, z, 3.14, 0.3, -1.57)

if __name__ == '__main__':
  moveit_commander.roscpp_initialize(sys.argv)
  rospy.init_node('r2_chessboard_cli')
  argv = rospy.myargv(argv=sys.argv) # filter out any arguments used by ROS
  if len(argv) != 3:
    print "usage: r2_chessboard.py square height"
    sys.exit(1)
  r2w = R2ChessboardWrapper()
  r2w.setSquare(*argv[1:])
  moveit_commander.roscpp_shutdown()
```

运行 r2_chessboard_cli.py：
```
$ ./r2_chessboard_cli.py a2 0.04
```


### 为R2定义下棋动作

定义手臂的三个状态：打开，预抓取，抓取：

```
#!/usr/bin/env python
import sys, rospy, tf, moveit_commander, random
from geometry_msgs.msg import Pose, Point, Quaternion

class R2Hand:
  def __init__(self):
    self.left_hand = moveit_commander.MoveGroupCommander("left_hand")

  def setGrasp(self, state):
    if state == "pre-pinch":
      vec = [ 0.3, 0, 1.57, 0,  # index
              -0.1, 0, 1.57, 0, # middle
              0, 0, 0,          # ring
              0, 0, 0,          # pinkie
              0, 1.1, 0, 0]       # thumb
    elif state == "pinch":
      vec = [ -0.1, 0, 1.57, 0,
              0, 0, 1.57, 0,
              0, 0, 0,
              0, 0, 0,
              0, 1.1, 0, 0]
    elif state == "open":
      vec = [0] * 18
    else:
      raise ValueError("unknown hand state: %s" % state)
    self.left_hand.set_joint_value_target(vec)
    self.left_hand.go(True)

if __name__ == '__main__':
  moveit_commander.roscpp_initialize(sys.argv)
  rospy.init_node('r2_hand')
  argv = rospy.myargv(argv=sys.argv) # filter out any arguments used by ROS
  if len(argv) != 2:
    print "usage: r2_hand.py STATE"
    sys.exit(1)
  r2w = R2Hand()
  r2w.setGrasp(argv[1])
```

运行r2_hand.py：
```
$ ./r2_hand.py pre-pinch
$ ./r2_hand.py pinch
```

### 让R2在棋盘上下棋

载入/重置棋盘
```
$ ./spawn_chessboard
```

历史上的对局被记录成pgn格式。
安装第三方的pgn 文件解释器。
```
sudo pip install pgnparser
```

运行spawn_chessboard.py，开始对局：
```
$ ./spawn_chessboard.py
```
# Raspi_Capture
以上文件位于树莓派的`~/ros2_ws`文件夹，主要代码位于：`src/two_camera/src/appsink1~3.cpp`以及`src/sub_synch/src/time_synch.cpp`中，ReadMe文档将从代码逻辑、执行方式以及如何在新的树莓派上部署代码展开。另外，服务端代码在最后给出并解释。
## 实现功能及代码逻辑
图采系统包括采集、同步、传输三部分，其中树莓派采集三个摄像头图像并做ros2时间戳同步，服务端发出指令：`capture N/save/shutdown`，三条指令分别对应：树莓派采集并保存N张同步且拼接后的图像到树莓派本地/将保存到本地的图片传输到服务器中/结束连接。
### appsink1~3
  当树莓派接收到capture N（N为正整数）指令时，在代码中会自动执行命令：ros2 run two_camera appsink1/2/3，这三个cpp文件分别对应三个摄像头，作用是发布三个摄像头的ros2节点信息，并带有时间戳方便同步。
### time_synch
  树莓派主要代码，作用是查询服务器有没有发出指令，以及在接收到指令之后执行对应操作。当图采系统处于准备状态时，场地上的所有树莓派一直在执行这段代码，并且执行shutdown断开与服务器的连接后也不会结束进程。
  - **接收到“capture N”后：**
  
    启动订阅节点订阅三个摄像头的图像，并做同步。每当成功同步一次就执行触发函数`void syncCallback()`，2*2拼接三张图像（右下角为纯黑图像），保存拼接后的图像并以“端口号_三张图象的平均ros2时间戳.jpg”命名。此时的树莓派日志会看到有`"[1]ROS 2 Time: %ld.%09ld\n", seconds, nanoseconds`的日志滚动，表明摄像头打开并发布消息。如果不想要日志滚动，在`appsink1~3.cpp`中注释掉对应的代码即可。
  - **接收到“save”后：**

    按照之前保存的路径，遍历其中所有的图片并依次发送到服务器，先发送文件名长度和文件名，再编码并发送图片。
  - **接收到“shutdown”后：**

    断开与服务器的连接，并删除保存在树莓派本地的图片，并关闭摄像头。关闭摄像头的实现是通过命令`fuser -k /dev/video0`查询并杀死对应摄像头的进程。此时可以看到之前滚动的日志结束了，表明摄像头关闭，这样可以避免摄像头持续打开，避免摄像头因为发热折寿[doge]。
## 部署方法
- **前期准备：** 有一个已经烧录`ubuntu-22.04-preinstalled-desktop-arm64+raspi.img`的SD卡+树莓派4b，安装ros2、时间设置、SSH开启等，参考这一篇博客（我写的）：https://blog.csdn.net/weixin_61967846/article/details/139688989?spm=1001.2014.3001.5501
- **代码部署：**
1. 新建文件夹，并安装pip3
  ```
    mkdir -p ~/ros2_ws/src
    cd ~/ros2_ws/src
    sudo apt install -y python3-pip
    sudo pip3 install rosdepc
    sudo rosdepc init
    rosdepc update
  ```
2. 安装/更新依赖，安装colcon命令并编译
  ```
    cd ~/ros2_ws
    rosdepc install -i --from-path src --rosdistro humble -y
    sudo apt update
    sudo apt upgrade
    sudo apt install python3-colcon-ros
    cd ~/ros2_ws
    colcon build
  ```
3. 安装two_camera和sub_synch工作包，创建launch文件夹（否则会编译报错）
  ```
    cd ~/ros2_ws/src
    ros2 pkg create two_camera --build-type ament_cmake --dependencies rclcpp --license Apache-2.0
    ros2 pkg create sub_synch --build-type ament_cmake --dependencies rclcpp --license Apache-2.0
    cd ~/ros2_ws/src/two_camera
    mkdir launch
    cd ~/ros2_ws/src/sub_synch
    mkdir launch
    cd ~/ros2_ws
    colcon build
  ```
4. 将`appsink1~3.cpp`和`time_synch.cpp`代码迁移过来
- 在two_camera/src中执行`nano appsink1.cpp``nano appsink2.cpp``nano appsink3.cpp`等命令创建cpp文件，将已有代码复制到其中，**注意修改：** 节点名称以及发布话题的名称，否则会与之前的树莓派重名导致程序卡死
  ```
    // 33行左右，在定义节点和话题的时候，将10003改为对应的端口。端口号建议与用户名后缀数字相同
    auto node = rclcpp::Node::make_shared("camera1_10003_publisher");
    auto publisher = node->create_publisher<sensor_msgs::msg::Image>("camera1_10003_image", 10);
    // 125行左右也要修改
    msg->header.frame_id = "camera1_10003_frame";
  ```
- 在sub_synch/src中执行`nano time_synch.cpp`命令创建cpp文件，将已有代码复制到其中，**注意修改：**
  ```
    // 17行左右的树莓派通信端口
    #define server_port 10003           // 本树莓派与服务器通信的端口
    // 32行左右，修改订阅话题名称
    this->declare_parameter("camera1_topic", "camera1_10003_image");
    this->declare_parameter("camera2_topic", "camera2_10003_image");
    this->declare_parameter("camera3_topic", "camera3_10003_image");
    this->declare_parameter("stitched_image_topic", "stitched_10003_image");
    // 36行左右，将保存路径中的用户名改为对应用户名，不是lamps-xe3
    this->declare_parameter("save_directory", "/home/lamps-xe3/ros2_ws/image");
  ```








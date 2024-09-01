# Raspi_Capture
以上文件位于树莓派的`~/ros2_ws`文件夹，主要代码位于：`src/two_camera/src/appsink1~3.cpp`以及`src/sub_synch/src/time_synch.cpp`中，ReadMe文档将从代码逻辑、执行方式以及如何在新的树莓派上部署代码展开。另外，服务端代码在最后给出并解释。
## 实现功能及代码逻辑
图采系统包括采集、同步、传输三部分，其中树莓派采集三个摄像头图像并做ros2时间戳同步，服务端发出指令：`capture N/save/shutdown`，三条指令分别对应：树莓派采集并保存N张同步且拼接后的图像到树莓派本地/将保存到本地的图片传输到服务器中/结束连接。
### appsink1~3
  当树莓派接收到capture N（N为正整数）指令时，在代码中会自动执行命令：ros2 run two_camera appsink1/2/3，这三个cpp文件分别对应三个摄像头，作用是发布三个摄像头的ros2节点信息，并带有时间戳方便同步。
### time_synch
  树莓派主要代码，作用是查询服务器有没有发出指令，以及在接收到指令之后执行对应操作。
  - 接收到“capture N”后：
  
    启动订阅节点订阅三个摄像头的图像，并做同步。每当成功同步一次就执行触发函数`void syncCallback()`，2*2拼接三张图像（右下角为纯黑图像），保存拼接后的图像并以“端口号_三张图象的平均ros2时间戳.jpg”命名。
    ```
// 设置消息同步器，并将同步后的回调函数注册
        sync_ = std::make_shared<message_filters::Synchronizer<ApproximateTime<sensor_msgs::msg::Image, sensor_msgs::msg::Image, sensor_msgs::msg::Image>>>(
            ApproximateTime<sensor_msgs::msg::Image, sensor_msgs::msg::Image, sensor_msgs::msg::Image>(20),
            sub1_, sub2_, sub3_);
        sync_->registerCallback(&CameraSyncNode::syncCallback, this);
    ```
  - 接收到“save”后：

    
  - 接收到“shutdown”后：

    
  当图采系统处于准备状态时，树莓派一直在执行这段代码，并且执行shutdown断开与服务器的连接后也不会结束进程，也就是说树莓派一直

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
- **前期准备：** 有一个已经烧录`ubuntu-22.04-preinstalled-desktop-arm64+raspi.img`的SD卡+树莓派4b，安装ros2、时间设置、SSH开启等，参考我写的这篇博客（建议收藏_嘿嘿）：https://blog.csdn.net/weixin_61967846/article/details/139688989?spm=1001.2014.3001.5501
- **代码部署：**

**1. 新建文件夹，并安装pip3**
  ```
    mkdir -p ~/ros2_ws/src
    cd ~/ros2_ws/src
    sudo apt install -y python3-pip
    sudo pip3 install rosdepc
    sudo rosdepc init
    rosdepc update
  ```
**2. 安装/更新依赖，安装colcon命令并编译**
  ```
    cd ~/ros2_ws
    rosdepc install -i --from-path src --rosdistro humble -y
    sudo apt update
    sudo apt upgrade
    sudo apt install python3-colcon-ros
    cd ~/ros2_ws
    colcon build
  ```
**3. 安装two_camera和sub_synch工作包，创建launch文件夹（否则会编译报错）**
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
**4. 将`appsink1~3.cpp`和`time_synch.cpp`代码迁移过来**
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
- **报错解决：** 在编译two_camera包的时候会提示缺少部分库，可执行如下命令安装：
  ```
    sudo apt-get install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
    sudo apt install libgtk-3-dev
  ```
- **环境变量：** 首次执行时需要把环境变量写入bash文件
```
  echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
  source ~/.bashrc
```

## 服务端程序
- 这是一个python程序，复制代码即可运行，迁移时**注意修改**：112行左右的端口号，把场地上所有的树莓派端口都写在里面
- 当需要执行服务端程序的时候，如果出现无法访问摄像头的提示，则可能是没有进入系统，此时需要拿着显示器和键盘一个个解锁树莓派（我承认这是一个缺点）
- 在运行本代码之前，需要保证所有树莓派都在执行`time_synch.cpp`代码，运行本代码即开始连接树莓派，连接到所有树莓派后需要发出命令：`capture N/save/shutdown`中的一条，例如N=10时，每个树莓派会保存10张图片到本地，最后传输也会传10张。一般流程是：
①  “capture 20”：每个树莓派保存20张同步且拼接好的图片
②  “save”：发送保存路径下的所有图片
③  “shutdown”：断开连接并删除树莓派保存路径下的所有图片
  ```
  import socket
  import os
  import numpy as np
  import cv2
  import multiprocessing
  import queue
  import time
  
  # 创建全局 command_queues 字典
  command_queues = {}
  
  def handle_client(client_socket, address, port):
      print(f"Connection from {address}:{port} has been established!")
      
      try:
          while True:
              # 从对应端口的队列中获取命令
              if port in command_queues and not command_queues[port].empty():
                  command = command_queues[port].get()
                  if command.startswith("capture"):
                      # 提取 image_count
                      _, count = command.split()
                      image_count = int(count)
                      client_socket.sendall(command.encode())
                      print(f"Capture command sent to client {address}:{port}. Image count: {image_count}")
                  elif command == "save":
                      client_socket.sendall(command.encode())
                      print(f"Save command sent to client {address}:{port}.")
                      # 使用默认值或通过其他方式获取 image_count
                      receive_images(client_socket, image_count)          
                  elif command == "shutdown":
                      client_socket.sendall(command.encode())
                      print(f"Shutting down connection to {address}:{port}.")
                      client_socket.shutdown(socket.SHUT_RDWR)
                      client_socket.close()
                      break
                  else:
                      print("Invalid command. Please enter 'capture', 'save', or 'shutdown'.")
              else:
                  time.sleep(0.1)
      except Exception as e:
          print(f"An error occurred with client {address}:{port}: {e}")
      finally:
          client_socket.close()
          print(f"Connection to {address}:{port} closed.")
  
  def receive_images(client_socket, image_count):
      save_directory = "./received_images"
      received_images = 0
      
      if not os.path.exists(save_directory):
          os.makedirs(save_directory)
  
      for _ in range(image_count):
          try:
              filename_length_buffer = client_socket.recv(4).decode().strip()
              if not filename_length_buffer:
                  break
              filename_length = int(filename_length_buffer)
  
              file_name = client_socket.recv(filename_length).decode('utf-8')
              print(f"Received file name: {file_name}")
  
              length_buffer = client_socket.recv(16).decode().strip()
              if not length_buffer:
                  break
              img_size = int(length_buffer)
  
              img_data = b''
              while len(img_data) < img_size:
                  packet = client_socket.recv(1024)
                  if not packet:
                      break
                  img_data += packet
              if len(img_data) != img_size:
                  continue
  
              img_array = np.frombuffer(img_data, dtype=np.uint8)
              img = cv2.imdecode(img_array, cv2.IMREAD_COLOR)
  
              if img is None:
                  continue
  
              save_path = os.path.join(save_directory, file_name)
              cv2.imwrite(save_path, img)
              print(f"Saved image: {file_name}")
              received_images += 1
  
              client_socket.sendall(b'ok')
          
          except Exception as e:
              print(f"Error: {e}")
              client_socket.sendall(b'error')
              break
  
      print(f"Successfully received {received_images}/{image_count} images.")
  
  def start_server_for_port(port):
      host = "0.0.0.0"
      server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
      server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
      server_socket.bind((host, port))
      server_socket.listen(5)
      print(f"Server started, listening on {host}:{port}")
  
      while True:
          client_socket, addr = server_socket.accept()
          process = multiprocessing.Process(target=handle_client, args=(client_socket, addr[0], port))
          process.start()
  
  def start_server():
      ports = [10001,10002,10003,10009]
      processes = []
      
      # 初始化每个端口的命令队列
      for port in ports:
          command_queues[port] = multiprocessing.Queue()
          process = multiprocessing.Process(target=start_server_for_port, args=(port,))
          process.start()
          processes.append(process)
  
      while True:
          command = input("Enter global command (e.g., 'capture 10' or 'save' or 'shutdown'): ")
          if command.startswith("capture"):
              for port in ports:
                  command_queues[port].put(command)
          elif command == "save":
              for port in ports:
                  command_queues[port].put(command)
                  # 等待当前端口的图片接收完成
                  while not command_queues[port].empty():
                      time.sleep(0.1)
          elif command == "shutdown":
              for port in ports:
                  command_queues[port].put(command)
              break
          else:
              print("Invalid command.")
  
  if __name__ == "__main__":
      start_server()
  ```







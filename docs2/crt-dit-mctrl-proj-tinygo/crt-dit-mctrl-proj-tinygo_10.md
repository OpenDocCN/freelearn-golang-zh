# 第九章：评估

# *第一章*

1.  `tinygo info` 命令。

1.  `tinygo flash` 命令。

1.  Arduino UNO 的时钟速度为 16 MHz。以 16 MHz 闪烁非常快，我们无法看到它。这就是为什么我们设置 LED 以毫秒为单位开启和关闭。

1.  您可以在代码仓库中找到解决方案：[`github.com/PacktPublishing/Creative-DIY-Microcontroller-Projects-with-TinyGo-and-WebAssembly/blob/master/Chapter01/blink-sos`](https://github.com/PacktPublishing/Creative-DIY-Microcontroller-Projects-with-TinyGo-and-WebAssembly/blob/master/Chapter01/blink-sos)

# *第二章*

1.  我们这样做是为了防止 LED 损坏。Arduino 入门套件或类似套件中的大多数 LED 都使用低于 5V 的电压。用 5V 供电可能会永久损坏它们。

1.  可以通过使用外部上拉（或下拉）电阻，或者使用内置电阻来实现。

1.  为了给调度器运行 goroutine 的时间，它需要进入休眠状态。

1.  您可以在 GitHub 仓库中的`traffic-lights-blink`文件夹下的`Chapter02`目录中找到这个问题的解决方案。

# *第三章*

1.  键 3 位于第 0 行第 2 列，因此坐标是 0,2。

1.  您可以在以下链接中找到解决方案：[`github.com/PacktPublishing/Creative-DIY-Microcontroller-Projects-with-TinyGo-and-WebAssembly/blob/master/Chapter03/safety-lock-keypad-check-key/main.go`](https://github.com/PacktPublishing/Creative-DIY-Microcontroller-Projects-with-TinyGo-and-WebAssembly/blob/master/Chapter03/safety-lock-keypad-check-key/main.go)

# *第四章*

1.  关闭这些传感器可以节省能量并延长水位传感器的使用寿命，因为它减缓了腐蚀。

1.  当信号（输入）端口信号为高时，电路闭合。

# *第五章*

1.  默认情况下，5V 引脚是不激活的。要激活它，需要焊接。或者，当 Arduino 通过 USB 端口供电时，可以使用 VIN 引脚。

1.  `pulseLength` 存储发送脉冲直到返回的时间。因此脉冲走了两倍的距离。这就是为什么我们必须将 `pulseLength` 除以 `2`。

1.  在这里找到解决方案：[`github.com/PacktPublishing/Creative-DIY-Microcontroller-Projects-with-TinyGo-and-WebAssembly/blob/master/Chapter05/touchless-handwash-timer-120seconds/main.go`](https://github.com/PacktPublishing/Creative-DIY-Microcontroller-Projects-with-TinyGo-and-WebAssembly/blob/master/Chapter05/touchless-handwash-timer-120seconds/main.go)

# *第六章*

1.  I2C 消息包含消息针对的设备的地址。

1.  CS 引脚被用来表示消息是针对特定设备的，因为 CS 引脚直接连接到设备。

# *第七章*

1.  为了确保消息能够送达，我们需要使用 QOS 级别 1 或级别 2，因为级别 0 是一种“发射后不管”的方法。

1.  是的，没有、一个或多个客户端可以订阅一个主题。

# *第八章*

1.  在 Wasm 代码内部验证凭证是不安全的，因为 Wasm 二进制文件正在发送给客户端。然后 Wasm 二进制文件可以被反编译，凭证可以被提取。

1.  我们始终可以使用任何类型的授权服务。一般来说，凭证不应该在客户端逻辑内部进行验证，而应该在任何其他服务上进行验证。

---
title: DIY Raspberry Pi 倒车雷达（一）程序设计
date: 2018-01-06 15:31:01
tags: [raspberry pi, 树莓派, C++, 面向对象, 设计模式]
categories: [DIY]
---
{% youtube 5cGRnKQr8CA %}

最近气温每天零下十几度，在家闲得无聊，突发奇想用Raspberry PI做了一个简易的倒车雷达。至于有没有实际用途，再说嘛，或许过几年就有钱买车了呢...

选择用Raspberry PI做的原因是... 嗯，家里正好有一个闲置的树莓派3。并且raspbian上可以直接用gcc编译c++程序。

第一部分内容主要针对程序设计。涉及wiringPi的使用和设计模式。


Github：<a href="https://github.com/ChrisXJH/parking_sensor">https://github.com/ChrisXJH/parking_sensor</a>

<!--more-->
<br>
## wiringPi
{% blockquote @CSDN http://blog.csdn.net/xukai871105/article/details/17737005 %}
WiringPi是应用于树莓派平台的GPIO控制库函数，WiringPi遵守GUN Lv3。wiringPi使用C或者C++开发并且可以被其他语言包转，例如python、ruby或者PHP等。WiringPi中的函数类似于Arduino的wiring系统，这使得熟悉arduino的用户使用wringPi更为方便。
树莓派具有26个普通输入和输出引脚。在这26个引脚中具有8个普通输入和输出管脚，这8个引脚既可以作为输入管脚也可以作为输出管脚。除此之外，树莓派还有一个2线形式的I2C、一个4线形式的SPI和一个UART接口。树莓派上的I2C和SPI接口也可以作为普通端口使用。如果串口控制台被关闭便可以使用树莓派上的UART功能。如果不使用I2C，SPI和UART等复用接口，那么树莓派总共具有8+2+5+2 =17个普通IO。wiringPi包括一套gpio控制命令，使用gpio命令可以控制树莓派GPIO管脚。用户可以利用gpio命令通过shell脚本控制或查询GPIO管脚。wiringPi是可以扩展的，可以利用wiringPi的内部模块扩展模拟量输入芯片，可以使用MCP23x17/MCP23x08（I2C 或者SPI）扩展GPIO接口。另外可通过树莓派上的串口和Atmega（例如arduino等）扩展更多的GPIO功能。另外，用户可以自己编写扩展模块并把自定义的扩展模块集成到wiringPi中。WiringPi支持模拟量的读取和设置功能，不过在树莓派上并没有模拟量设备。但是使用WiringPi中的软件模块却可以轻松地应用AD或DA芯片。
{% endblockquote %}
<br>

wiringPi官网：<a href="http://wiringpi.com/">http://wiringpi.com/</a>

下面是一个自己写的interface，用来简化程序开发时的gpio操作。

{% codeblock gpio_interface.h lang:c %}
#include <time.h>
#include "wiringPi.h"
#include "gpio_interface.h"

#define BUZZER 5
#define LED_SUCCESS 19
#define DISTANCE_TRIG 16
#define DISTANCE_ECHO 20


void buzz(const bool on) {
    if (on) digitalWrite(BUZZER, HIGH);
    else digitalWrite(BUZZER, LOW);
}

void blink_green(const bool on) {
    if (on) digitalWrite(LED_SUCCESS, HIGH);
    else digitalWrite(LED_SUCCESS, LOW);
}

double measure_distance() {
    clock_t start, end;
    digitalWrite(DISTANCE_TRIG, HIGH);
    delay(0.00001);
    digitalWrite(DISTANCE_TRIG, LOW);

    // Making sure the signal is sent out
    while (!digitalRead(DISTANCE_ECHO)) continue;
    // Mark down the time at which the signal is sent out
    start = clock();
    // Keep listening for echo signal
    while (digitalRead(DISTANCE_ECHO)) continue;
    // Mark down the time when receiving the feedback signal
    end = clock();

    double clocks_per_mill = CLOCKS_PER_SEC / 1000;
    return ((end - start) / clocks_per_mill) * 34.3;
}

void init_gpio() {
    wiringPiSetupGpio();
    pinMode(BUZZER, OUTPUT);
    pinMode(LED_SUCCESS, OUTPUT);
    pinMode(DISTANCE_TRIG, OUTPUT);
    pinMode(DISTANCE_ECHO, INPUT);
}

void close_gpio() {
    digitalWrite(BUZZER, LOW);
    digitalWrite(LED_SUCCESS, LOW);
    digitalWrite(DISTANCE_TRIG, LOW);
}

{% endcodeblock %}
<br>
## 观察者设计模式
本项目采用观察者设计模式，主要有两个基类：Observer（观察者）和 Subject（主体）。每个观察者可以订阅一个或多个主体，当主体的状态发生变化时，主体会立即通知观察者。

{% img [class names] http://design-patterns.readthedocs.io/zh_CN/latest/_images/Obeserver.jpg 观察者模式图解 %}


{% codeblock observer.h lang:c %}
template <typename T> class Subject;
template <typename T>
class Observer {

public:
    Observer() {}
    virtual void notify(Subject<T> &) = 0;
    virtual ~Observer() {}

};

{% endcodeblock %}

{% codeblock subject.h lang:c %}
#include <vector>
#include "observer.h"

template <typename T>
class Subject {
    std::vector<Observer<T>* > observers;

public:
    Subject() {}
    virtual ~Subject() {}

    virtual T getInfo() = 0;
    void attachObserver(Observer<T> * ob) {
        observers.emplace_back(ob);
    }

    void notifyObservers() {
        for (auto &ob : observers) {
            ob->notify(* this);
        }
    }
};

{% endcodeblock %}
<br>
因为当感应器检测到障碍物时，系统需要通知报警器进行报警，所以感应器需继承主体（Subject）。当检测到障碍物时，感应器便会通知主体并且告诉主体关于障碍物的信息。为了提升可扩展性（感应器可以有很多种），距离传感器（distanceSensor）应继承感应器基类（Sensor），而不应该直接继承Subject。

{% codeblock distanceSensor.h lang:c %}
class DistanceSensor : public Sensor {
    double distance;
    double interval;

    double detectDistance();
    double simulateDistance();
    void setInterval(const double &inv);

public:
    DistanceSensor();
    ~DistanceSensor();

    SensorInfo getInfo() override;
    void start();
};

{% endcodeblock %}
<br>

distanceSensor会定期测量汽车尾部与障碍物的距离。每当得到新距离时，distanceSensor会通过Controller::notify(Subject<SensorInfo> &whoFrom)来通知Controller。这时， Controller可以调用distanceSensor::getInfo()来获取到汽车相对于障碍物的距离。当然了，前提是Controller继承Observer。

Controller同时继承Subject。当Controller接收到distanceSensor传来的距离信息时，会通知Monitor（Observer），并把距离信息传给Monitor来判断需不需要报警。

{% codeblock controller.h lang:c %}
#include "monitor.h"
#include "observer.h"
#include "subject.h"
#include <memory>
class Sensor;
class Monitor;
class SensorInfo;
class ControllerInfo;

struct ControllerTemp {
    double distance;
};

class Controller : public Observer<SensorInfo>, public Subject<ControllerInfo> {
    std::unique_ptr<ControllerTemp> temp;

public:
    Controller();
    ~Controller();

    void addSensor(Sensor * );
    void addMonitor(Monitor * );
    void init();
    void notify(Subject<SensorInfo> &) override;
    ControllerInfo getInfo() override;
};

{% endcodeblock %}

{% codeblock monitor.h lang:c %}
#include "observer.h"
class ControllerInfo;
class Monitor : public Observer<ControllerInfo> {

public:
    Monitor();
    virtual ~Monitor();

    virtual void notify(Subject<ControllerInfo> &) = 0;

};

{% endcodeblock %}
<br>

## 参考网站
- <a href="http://wiringpi.com/">http://wiringpi.com/</a>
- <a href="http://design-patterns.readthedocs.io/zh_CN/latest/behavioral_patterns/observer.html">http://design-patterns.readthedocs.io/zh_CN/latest/behavioral_patterns/observer.html</a>
- <a href="http://blog.csdn.net/xukai871105/article/details/17737005">http://blog.csdn.net/xukai871105/article/details/17737005</a>

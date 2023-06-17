## BYD汽车的CAN总线DBC文件

最近尝试研究改装汽车的辅助驾驶或自动驾驶，发现很多改装大都是基于开源项目openpilot做的，而openpilot控制汽车的方法主要是基于汽车上的CAN总线控制方向盘油门刹车等，但是CAN总线的通信协议基本上汽车厂家都是不公开的需要自己去研究解析。有一群人做了这个opendbc项目用于整理很多车辆的CAN总线协议，但目前还没有比亚迪车的dbc文件。我对比亚迪的CAN总线做了一些初步研究，在此分享我的解析到的信息。  

研究CAN协议需要硬件获取CAN总线的信号，目前最便宜的可能是几十元的开源硬件CANable USB转CAN适配器。作为初步研究我是从OBD2接口的4, 6 和14针引出CAN总线（接线参考图：https://openlightlabs.com/collections/frontpage/products/obd2-16-pin-pigtail-cable ）。电脑端软件可以使用开源的SocketCAN, Cangaroo或者SavvyCAN用于记录分析数据。  

这堆dbc文件中的`byd-tang-phev-2015.dbc`是我研究一款2016款唐新能源插电混动车型做的，型号BYD6480STHEV，VIN开头LC0CD4C3×××××××××。我发现开关门之类的信号比较容易研究，容易发现对应的位置。而速度方向盘角度之类的变量较难研究解析。另外我发现用`candump`记录的文件，用`canplay`发送车辆没有反应可能是OBD2接口的CAN总线有过滤。  

其实CAN总线解析有一些商家也在做，比如有些HUD抬头显示器也是依据CAN协议获取的数据。另外比亚迪原厂的诊断仪VDS2100或者第三方诊断仪应该也是通过CAN协议获取数据。如果能接触到相关硬件，可以考虑从这些硬件中尝试提取研究。  

## DBC file basics

A DBC file encodes, in a humanly readable way, the information needed to understand a vehicle's CAN bus traffic. A vehicle might have multiple CAN buses and every CAN bus is represented by its own dbc file.
Wondering what's the DBC file format? [Here](http://www.socialledge.com/sjsu/index.php?title=DBC_Format) and [Here](https://github.com/stefanhoelzl/CANpy/blob/master/docs/DBC_Specification.md) a couple of good overviews.

## How to start reverse engineering cars

[opendbc](https://github.com/commaai/opendbc) is integrated with [cabana](https://github.com/commaai/openpilot/tree/master/tools/cabana).

Use [panda](https://github.com/commaai/panda) to connect your car to a computer.

## How to use reverse engineered DBC
To create custom CAN simulations or send reverse engineered signals back to the car you can use [CANdevStudio](https://github.com/GENIVI/CANdevStudio) project.

## DBC file preprocessor

DBC files for different models of the same brand have a lot of overlap. Therefore, we wrote a preprocessor to create DBC files from a brand DBC file and a model specific DBC file. The source DBC files can be found in the generator folder. After changing one of the files run the generator.py script to regenerate the output files. These output files will be placed in the root of the opendbc repository and are suffixed by _generated.

## Good practices for contributing to opendbc

- Comments: the best way to store comments is to add them directly to the DBC files. For example:
    ```
    CM_ SG_ 490 LONG_ACCEL "wheel speed derivative, noisy and zero snapping";
    ```
    is a comment that refers to signal `LONG_ACCEL` in message `490`. Using comments is highly recommended, especially for doubts and uncertainties. [cabana](https://community.comma.ai/cabana/) can easily display/add/edit comments to signals and messages.

- Units: when applicable, it's recommended to convert signals into physical units, by using a proper signal factor. Using a SI unit is preferred, unless a non-SI unit rounds the signal factor much better.
For example:
    ```
    SG_ VEHICLE_SPEED : 7|15@0+ (0.00278,0) [0|70] "m/s" PCM
    ```
    is better than:
    ```
    SG_ VEHICLE_SPEED : 7|15@0+ (0.00620,0) [0|115] "mph" PCM
    ```
    However, the cleanest option is really:
    ```
    SG_ VEHICLE_SPEED : 7|15@0+ (0.01,0) [0|250] "kph" PCM
    ```

- Signal size: always use the smallest amount of bits possible. For example, let's say I'm reverse engineering the gas pedal position and I've determined that it's in a 3 bytes message. For 0% pedal position I read a message value of `0x00 0x00 0x00`, while for 100% of pedal position I read `0x64 0x00 0x00`: clearly, the gas pedal position is within the first byte of the message and I might be tempted to define the signal `GAS_POS` as:
    ```
    SG_ GAS_POS : 7|8@0+ (1,0) [0|100] "%" PCM
    ```
    However, I can't be sure that the very first bit of the message is referred to the pedal position: I haven't seen it changing! Therefore, a safer way of defining the signal is:
    ```
    SG_ GAS_POS : 6|7@0+ (1,0) [0|100] "%" PCM
    ```
    which leaves the first bit unallocated. This prevents from very erroneous reading of the gas pedal position, in case the first bit is indeed used for something else.

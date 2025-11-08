# 用Vscode 开发 STM32
原文@Jayant 图片来源：知乎 @Jayant

## 前期准备
路径都最好不要有中文。
原理：
要用到的工具/环境：
- STM32CubeMX：https://www.st.com.cn/zh/development-tools/stm32cubemx.html
- JRE（因为STM32CubeMX依赖JAVA）:https://www.oracle.com/java/technologies/downloads/
- OpenOCD ：https://gnutoolchains.com/arm-eabi/openocd/
- Zadig （Linux系统无视这条就行；不用JLink也无视这条。）
- Git：https://git-scm.com/（可以不装但是建议装，不然也是要装mingw的）
- VSCode
- GNU Arm Embedded Toolchain（这里要注意你的装载环境和靶机环境，例如我的win11 x86 ->stm32f103x 就选择arm-gnu-toolchain-14.3.rel1-mingw-w64-x86_64-arm-none-eabi.exe）：https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads

要加的环境变量
将工具链中的bin
OpenOCD中的bin
Git中的mingw64/bin

环境变量修改完毕后，需要重启电脑才能生效。然后打开Power Shell，输入：

arm-none-eabi-gcc --version
和

openocd --version
可以看到版本信息，说明安装成功。

这两个命令也可以在git bash中运行

# 1 用STM32CubeMX创建工程
STM32CubeMX生成的是用HAL库开发的项目，具体怎么配置这里就不介绍了，只介绍重要部分。
## 1.1. 安装支持包
主页右边可以安装不同芯片的支持包，例如F1系列，F4系列
![alt text](img/image.png)
## 1.2. Debug接口配置
左上角File..可以新建项目。
新建项目以后，SYS里选择debug接口，这里选的是SWD，也可以选JTAG（这里的框默认好像是叠起来的，要自己拉出来。）
![alt text](img/image-1.png)
## 1.3. 时钟配置
先在RCC里选择高速外部时钟（HSE）和低速外部时钟源（LSE），这里选的都是晶振（因为板子上有这两个晶振）。
![alt text](img/image-2.png)
然后选择“时钟配置”，先在左边填好外部晶振的频率，然后在右边填上自己想要的主频，Cube会自动帮你配置锁相环。我是直接写最大值了
![alt text](img/image-3.png)
## 1.4. 项目配置
Toolchain/IDE选择生成makefile即可。前面的项目结构我选的basic，你也可以选Advanced，后面目录结构就不一样了，VSCode的配置要稍微改一下。
![alt text](img/image-4.png)
## 1.5. 代码生成器配置
我选了这几个选项，看名字就知道意思了，推荐勾选
![alt text](img/image-5.png)
## 1.6. 生成代码
点击右上角的GENERATE CODE生成代码。

# 2. 配置VS Code
用VS Code打开工程文件夹，你将会看到这样的目录结构:
![alt text](img/image-6.png)
.ioc文件和.mxproject文件是STM32Cube的工程文件，Driver里是STM32和ARM CMSIS的库，最好不要修改。Inc和Src是供用户修改的源码。

## 2.1. 安装VS Code 插件
在插件商店搜索即可，需要这几样：

Chinese (Simplified)：VS Code的语言支持是以插件形式存在的，需要装个中文插件;


C/C++：提供代码补全、智能感知和debug功能；
（注意，VSCode 可能会推荐你安装C/C++ Intellisense插件来做智能感知 ，但它依赖于GNU Global工具，我们的arm工具链里没有这个，所以不用装）；

C/C++ Snippets：好用的代码模板小工具。比如说，装好以后，敲个for，就可以自动补全整个for循环代码；
ARM：提供ARM汇编语言的代码高亮；（2025注：好像没找到了，不装不影响使用）
Cortex-Debug：本教程的核心，有了它，才能把ARM工具链和OpenOCD等命令行工具组织到VSCode中，并进行图形化操作。

## 2.2. 配置VS Code内置终端
（2025注：不配置没有影响，新建终端后点终端框框上面的加号加一个git bash就可以了）
## 2.3 配置智能感知
其实这个时候我们敲make已经可以编译成功了。但是VS Code的编辑窗口里会给我们亮一堆红点，代码里给我们一堆红色波浪线。这是因为VS Code本身的一个待改进的地方，下面具体解决。

还记得我们使用Keil开发时，Project Options里的全局宏定义吗？
![alt text](img/image-7.png)
在通过Makefile组织的项目中，这两个宏是通过gcc的-D参数在编译时添加的：
![alt text](img/image-8.png)
但是，VS Code只是一个编辑器，它检查代码的时候并不会去读makefile，而是只看.h和.c文件，于是STM32F1xx.h中就检测不到那个宏，表现为灰色（认为这个宏没有被定义）：
![alt text](img/image-9.png)
于是你就可以看到一大串”xxxx is undefined”之类的报错，其实都是这个原因导致的。但是直接去make的话是没有问题的。

因此我们需要在当前目录的.vscode文件夹下创建c_cpp_properties.json配置文件，用来告诉VS Code我们定义了这些宏。

随便找到一处红色波浪线，点击并把光标移到那一行，左上角会出现一个黄色小灯泡。点击黄色小灯泡并选择“编辑‘includePath设置’”。
![alt text](img/image-10.png)
直接用c_cpp_properties.json来配置:
![alt text](img/image-11.png)
VS Code自动在当前目录下的.vscode文件夹下生成一个c_cpp_properties.json文件，我的配置给出如下：
```json
{
    "configurations": [
        {
            "name": "Win32",
            "includePath": [
                "C:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update/lib/gcc/arm-none-eabi/9.3.1/include",
                "${workspaceFolder}/Inc",
                "${workspaceFolder}/Drivers/STM32F1xx_HAL_Driver/Inc",
                "${workspaceFolder}/Drivers/STM32F1xx_HAL_Driver/Inc/Legacy",
                "${workspaceFolder}/Drivers/CMSIS/Device/ST/STM32F1xx/Include",
                "${workspaceFolder}/Drivers/CMSIS/Include"
            ],
            "defines": [
                "USE_HAL_DRIVER",
                "STM32F103xx"  
            ],
            "compilerPath": "C:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update/bin/arm-none-eabi-gcc.exe",
            "intelliSenseMode": "gcc-x64",
            "browse": {
                "limitSymbolsToIncludedHeaders": true,
                "databaseFilename": "",
                "path": [
                    "${workspaceFolder}"
                ]
            }
        }
    ],
    "version": 4
}
```
解释几个重要部分：

- “name”：这是用于标记使用的平台的标签。除了win32还可以选Linux或Mac。也就是说，这个json里
- “configuration“下可以写三组配置，只要每组配置前面写上不同的平台，即可在不同的操作系统上使用就会自动适配不同的配置，非常方便
- "includePath"：告诉VS Code该去哪里查找头文件。第一个目录是C语言标准库的目录， 剩下的几个目录直接从Makefile里复制然后稍微修改下即可。"${workspaceFolder}"表示项目文件夹；
- ”defines“：全局宏定义，告诉VS Code这些宏都被定义了，只是没写在源码中而已。上述多加的两个宏是makefile里的。
- "compilerPath"：指定编译器的路径。因为有一些宏是编译器自带的，连makefile里都没有，例如__GNUC__。有些教程里会让你在defines里面加上__GNUC__，但是这是没必要的。只要你指定了编译器路径，所有的编译器自带的宏就都导入了VS Code。
- "intelliSenseMode"：因为我们用的是gcc所以选gcc-x64
- "browse.path"：源文件搜索路径。据说是用来做代码补全和查找定义的，但是我测试后发现删去也不影响使用，不过还是留着吧。这个路径和includePath不同，browse.path是自动递归所有子目录的。而include.path默认只看本目录。

Ctrl+S保存c_cpp_properties.json文件，发现左边目录里一个红点都没有了，强迫症舒服了！
![alt text](img/image-12.png)

## 2.31让git集成make
到 https://sourceforge.net/projects/ezwinports/files/ 去下载 make-4.1-2-without-guile-w32-bin.zip这个文件。 把该文件进行解压 把解压出来的文件全部拷贝的git的安装目录下： . \Program Files\Git\mingw64\ ,把文件夹进行合并，如果跳出来需要替换的文件要选择不替换 这样在git bash窗口下就可以执行make了

## 2.4. 配置build任务
直接在终端里敲一个make，就会根据makefile的内容，在当前目录下创建一个build文件夹，在里面生成每个源文件生成的.o文件，以及最终链接得到的elf文件（用于调试），以及用于直接下载用的十六进制文件(.hex)和二进制文件(.bin)。编译成功的话看起来就像这样：
![alt text](img/image-13.png)

为了方便后面的操作，我们在.vscode目录下创建tasks.json文件(文件名里别少了s！)，内容如下：
```cpp
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
        "version": "2.0.0",
        "tasks": [
            {
                "label": "build",
                "type": "shell",
                "command": "make",
                "args": [
                    "-j4"
                ] 
            },
            {
                "label": "clean",
                "type": "shell",
                "command": "make",
                "args": [
                    "clean"
                ] 
            }
        ]
}
```
这个文件创建了两个任务，分别叫做build和clean，build任务就是在bash里执行了make -j4，clean任务就是在bash里执行了make clean。VS Code是可以给任务绑定快捷键的，具体就不介绍了，有兴趣可以自己搜索。

不使用快捷键而执行task的方法：按Ctrl + P，然后输入”task[空格]“，就会出现可用的任务列表。

至此，编译的部分已经完成

1. openocd配置
直接在项目文件夹下新建一个openocd.cfg文件，内容如下
```
# 选择调试器为jlink
source [find interface/stlink.cfg]
#source [find interface/cmsis-dap.cfg]

# 选择接口为SWD
transport select swd

# 选择目标芯片
source [find target/stm32f1x.cfg]

reset_config none
```
openocd启动时，会自动在当前目录下寻找名为openocd.cfg的文件作为配置文件。

本配置文件中引用到的其他配置文件，都在openocd安装目录下的share/openocd/scripts目录下。其中interface目录下都是接口相关配置文件、target目录下都是芯片相关的配置文件。

2. 下载svd文件
点[我爱拉屎](https://github.com/modm-io/cmsis-svd-stm32)寻找STM32F1的svd文件。CMSIS-SVD是CMSIS的一个组件，它包含完整微控制器系统（包括外设）的程序员视图的系统视图描述 XML 文件。简单来说，VS Code可以通过它来知道外设寄存器的地址分布，从而把寄存器内容展示到窗口中。
下载好的STM32F1.svd文件放在项目文件夹根目录即可。

3. 配置VS Code的调试功能
【openocd版】在.vscode文件夹中新建一个launch.json，内容如下：
```JSON
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        
        {
            "name": "Cortex Debug",
            "cwd": "${workspaceRoot}",
            "executable": "${workspaceRoot}/build/${workspaceFolderBasename}.elf",
            "request": "launch",
            "type": "cortex-debug",
            
            "device":"STM32F407VE",        //使用J-link GDB Server时必须；其他GBD Server时可选（有可能帮助自动选择SVD文件）。支持的设备见 https://www.segger.com/downloads/supported-devices.php
            "svdFile": "./STM32F103.svd",  //svd文件，有这个文件才能查看寄存器的值，每个单片机都不同。可以在以下地址找到 https://github.com/posborne/cmsis-svd
            "servertype": "openocd",       //使用的GDB Server
            "configFiles": [                  
                "${workspaceRoot}/openocd.cfg"
            ],
            "preLaunchTask": "build",
            "armToolchainPath": "Z:\\GUN_ARM\\14.3 rel1\\bin",
            "runToMain": true
        }
    ]
}
```
解释几个重要选项：

- "executable"：编译出的二进制文件，也就是最终烧录到单片机中的，这里是elf文件。根据芯片的不同，可能产生不同的名称和后缀（例如TI的TM4C123芯片编译出来的名称是"main.axf"）
- "request"：可以选launch或attach。launch是指启动调试时同时开始执行程序；attcah是指程序已经在运行了，然后开始调试。我没测试过attach。
- "type"：调试的类型，选cortex-debug，这是我们装的插件。其实也可以填cppdbg之类的，但是那样我们就得自己配置gdb了，配置起来将会非常麻烦。
- "device"：目标芯片。如果你使用J-LINK GDB Server时必须要设置这个选项。然而我们的GDB Server是openocd，J-Link只用来连接芯片。
- "svdFile"：svd文件的路径。
- "servertype"：要选择的gdb server。我们用openocd。
- "configFiles"：gdb server的配置文件路径。其实openocd会自动读当前目录下的openocd.cfg文件，这个选项不填也行。但是如果你想把openocd.cfg放在别处，就可以用这个选项指定配置文件的路径。
- "preLaunchTask"：在启动调试前，预先执行的任务。在这里我们设置为前一篇文章里配置的build任务。这样每次调试前都会先自动编译好
- "armToolchainPath"：工具链的路径。配置了全局环境变量的情况下好像不设置也行。

4. 测试使用
保存以上所有文件后，目录结构应该是这样：
![alt text](img/image-14.png)

直接按F5 

可以看到变量窗口、调用堆栈、断点、外设寄存器、CPU寄存器。

完整流程：
```
[1] STM32CubeMX 生成项目
    ↓
[2] VSCode 打开项目
    ↓
[3] 编写逻辑代码 (main.c)
    ↓
[4] make 编译 (arm-none-eabi-gcc)
    ↓
[5] OpenOCD 烧录/调试 (.bin)
    ↓
[6] F5 调试 (Cortex-Debug)
```
文件目录：
```
Z:.
│  .mxproject //STM32CubeMX 项目的配置文件。
│  Makefile
│  openocd.cfg //定义了调试器接口、目标芯片类型等信息。
│  readme.md
│  startup_stm32f103xb.s //负责 STM32 微控制器的启动过程，在项目开始时，程序从这个文件启动。
│  STM32F103.svd //描述了目标芯片的外设寄存器地址和结构。它用于调试工具（如 VS Code 中的 Cortex-Debug 插件）来正确显示外设寄存器的值。
│  STM32F103XX_FLASH.ld //链接脚本文件，定义程序如何在STM32存储器中布局。指定程序的起始地址、各个段的地址范围等内容，确保程序能够正确加载到目标设备的内存中。
│  TEST1.ioc //通过 STM32CubeMX 生成，包含了硬件配置和外设设置。
│
├─.vscode
│      .cortex-debug.peripherals.state.json
│      .cortex-debug.registers.state.json
│      c_cpp_properties.json
│      launch.json
│      settings.json
│      tasks.json
│
├─build
│      gpio.d
│      gpio.lst
│      gpio.o
│      main.d
│      main.lst
│      main.o
│      ......
│
├─Drivers
│  ├─CMSIS
│  │  │  LICENSE.txt
│  │  │
│  │  ├─Device
│  │  │  └─ST
│  │  │      └─STM32F1xx
│  │  │          │  LICENSE.txt
│  │  │          │
│  │  │          ├─Include
│  │  │          │      stm32f103xb.h
│  │  │          │      stm32f1xx.h
│  │  │          │      system_stm32f1xx.h
│  │  │          │
│  │  │          └─Source
│  │  │              └─Templates
│  │  └─Include
│  │          cmsis_armcc.h
│  │          cmsis_armclang.h
│  │          cmsis_compiler.h
│  │          ......
│  │
│  └─STM32F1xx_HAL_Driver
│      │  LICENSE.txt
│      │
│      ├─Inc
│      │  │  stm32f1xx_hal.h
│      │  │  stm32f1xx_hal_cortex.h
│      │  │  stm32f1xx_hal_def.h
│      │  │  ......
│      │  │  
│      │  │
│      │  └─Legacy
│      │          stm32_hal_legacy.h
│      │
│      └─Src
│              stm32f1xx_hal.c
│              stm32f1xx_hal_cortex.c
│              stm32f1xx_hal_dma.c
│              ......
│
├─img
│      image-1.png
│      image-10.png
│      image-11.png
│      ......
│
├─Inc
│      gpio.h
│      main.h
│      stm32f1xx_hal_conf.h
│      stm32f1xx_it.h
│
└─Src
        gpio.c
        main.c
        stm32f1xx_hal_msp.c
        stm32f1xx_it.c
        syscalls.c
        sysmem.c
        system_stm32f1xx.c
```

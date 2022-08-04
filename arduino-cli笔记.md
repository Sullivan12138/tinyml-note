注意，要编译的文件.ino必须放在一个与它同名的文件夹下。如果有其他要调用的.h或者.c文件，也必须放在同一个文件夹下。否则就要在compile的时候加--library或者--libraries选项。
arduino-cli必须在这个文件夹外运行。
首先运行`arduino-cli.exe core update-index`
然后运行`arduino-cli core install arduino:mbed_nano`安装arduino nano 33 ble sense对应的内核
然后运行
```
arduino-cli compile --fqbn arduino:mbed_nano:nano33ble hello_world --outout-dir xxx 
```
来编译。
其中xxx为你想要把编译后的文件夹放在的位置
--fqbn是针对你这个板子的一个型号，我们的板子是arduino:mbed_nano:nano33ble.如果不知道自己板子对应的fqbn可以插上板子以后运行
`arduino-cli board list`查看。但是编译的过程是不需要板子的。
编译好了以后可以去那个文件夹底下看看，应该能看到bin文件。
最后运行
```
arduino-cli upload -p xxx --fqbn arduino:mbed_nano:nano33ble --input-dir yyy
```
其中xxx是板子连接的端口，yyy是你之前编译以后生成的文件夹放的位置。
如果不知道板子连接的端口也可以运行`arduino-cli board list`查看。
添加自定义库的方法：--library xxx，其中xxx为你的库的路径，注意必须为绝对路径。或者你可以把你的多个库文件夹放在一个总的文件夹yyy下，然后--libraries yyy。
另外，arduino里面库文件夹一般的结构是
-- library-name
&nbsp;&nbsp;&nbsp;&nbsp;|-- examples
&nbsp;&nbsp;&nbsp;&nbsp;|-- src
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|-- library-name.h
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|-- your library folder...
&nbsp;&nbsp;&nbsp;&nbsp;|-- library.properties
你当然也可以把你要引用的库文件直接放在library-name文件夹底下，但是放src下的话更加的整齐划一美观。
引用的时候不必写#include "src/.."，也就是说不必把src文件夹加上。为什么呢？
在library.properties中写上includes=library-name.h，然后在src文件夹下放一个library-name.h，这样在你的.ino里面引用这个库的时候，就可以不必在include的路径里面加src了。library-name.h里面可以什么都不用写，它的作用是编译器加载了它以后，就会顺便把和它同文件夹下的你的库文件夹一起加载。

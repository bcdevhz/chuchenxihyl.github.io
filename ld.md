PHP扩展配置文件checklib导致ld链接错误。

PHP扩展编译参数配置文件config.m4,用GNU autoXXX工具的一个checklib宏来检查某一个外部库(扩展里使用了该库)。

由于本地库环境的问题导致ld链接不到正确的库，这是写作本文的缘由。

# 1.PHP配置
PHP为简化扩展的实现，引入phpize、php-config脚本，用来补充autoconf、automake生成Makefile文件。

phpize.m4主要配置configure.in，操作GNU autoXXX的众多工具，典型如autoconf/automake。大体经过头文件、宏的配置后，生成config.h.in与configure.in。

经config后分别生成config.h与configure文件。automake则是基于configure将makefile.in最终转换为makefile。

php-config则主要用来配置PHP安装信息(版本、路径、编译链接器选项、链接外部库名及路径等)。 扩展config时可通过参数传给configure.in生成makefile文件。

扩展完成config后，可在config.log里查看整个配置过程。

生成makefile的流程参考下图：

![Image](/Users/huyanlinyouzan.com/Downloads/blog1-makefile.gif)

从上可看出configure.in的用处，基本的配置项PHP内核已帮您配置好。而在PHP扩展中则提供了一个config.m4会被自动include进内核配置。编写扩展时只需通过修改这个文件来配置该扩展的编译配置项。

# 2.PHP扩展配置文件内容
config.m4文件主要包含添加编译参数、链接库名及相应路径、编译本扩展的源目录及文件列表、config参数检查与配置。

autoXXX工具主要通过宏来配置，常见的宏有：

`
AC_ARG_ENABLE(arg_name,check message,help info)  //--enable-arg_name开配置项

AC_ARG_WITH(arg_name,check message,help info)   //同上，默认开，不需要arg_name

AC_DEFINE(variable, value, [description])  //自定义宏，配在config.h

ADD_INCLUDE(dir)  // 添加include路径，即类似于链接器ld的-Idir,用于在指定的路径下找符号表信息

AC_CHECK_FUNC(function, [action-if-found], [action-if-not-found])  //check函数是否存在

AC_CHECK_LIB(library, function, [action-if-found], [action-if-not-found], [other-libraries]) //check库里的函数是否存在
`
除了以上还有未列出的如AC_CHECK_FILE／AC_ADD_LIBRARY_WITH_PATH等。

这里特别需要提出的是AC_CHECK_LIB,它主要用来：

- 将要链接的library库是否存在(第一个参数library)

- 如果库存在，库里是否包含function这个函数的符号表信息(第二个参数function) //这个可通过nm工具查看某库是否存在该符号表的引用、定义

- 如果上述两步check下来结果为真，所要做的操作(第三个参数action-if-found)

- 如果上述两步check下来结果为假，所要做的操作(第四个参数action-if-not-found)

-如果第二步check下来的符号表信息不充分，则尝试链接其它的库以匹配符号表(第五个参数other-libraries) //相当于-lX

# 3. 链接器信息与lib check

链接器的引入主要为解决跨文件、跨模块间的符号引用。由于引用的符号地址不能确定，链接器的工作即是把所有的目标文件连接到一起进而来确定最终的符号地址。

如果A模块想引入B模块中定义的符号(典型的函数名符号信息)，需要透过链接器这个中介。

那么如果在正常编译生成尚未链接的目标文件后，链接阶段，想要链接第三方库文件，如果提示无法找到库文件，当且仅当原因有三：

1. 没有库文件

2. 路径配置不正确

3. 库使用方式不正确

为解决本文开头引入的问题，当然也按上述步骤：

1. 重新安装了库文件

2. 库路径设置称为绝对路径(不设情况下，默认库顺序为)

3. 库名严格对应checklib宏中lXXX静态链接的libXXX.so（mac下则是libXXX.dylib)  

(注意libXXX.so.xxx动态库一般含大小中三个版本，静态链接时认为libXXX.so.xxx与libXXX.so是两个不同的库，一般情况下通过软链接指过去)

但是问题依然没有解决，结合config.log想到，不能按程序链接阶段去思考。这里在config阶段检查libXXX.so库是否存在，为什么一定要在编译阶段检查

一个动态库的某一个函数是否存在呢，原因是目标扩展里使用这个库里的该函数，就需要静态链接该动态库。

这里需要注意为什么会有静态链接与动态链接的区别？

众所周知，链接器的主要工作即是填写符号表地址的过程，即重定位。填写这个地址依据地址的类型与阶段，来区分链接器的功效。

1. 模块内链接：解决程序内部跨文件引用

2. 装载重定位：引用外部库文件装载时重定位

3. 延迟绑定：引用外部库文件时加快加载速度

于是查看了config.log里链接出错的地方。一般出错会用下面log输出：

```
 /* Override any GCC internal prototype to avoid an error.
    Use char because int might match the return type of a GCC
    builtin and then its argument prototype would still apply.  */
 #ifdef __cplusplus
 extern "C"
 #endif
 char XXX ();  //XXX为上述libcheck中的函数，即第二个参数名
 int
 main ()
 {
 return XXX ();
   ;
   return 0;
 }
```

用gcc -lXXX命令行链接(本地环境库环境混乱，通过-I指定绝对路径)，链接成功，nm工具可查看完整符号表信息。

这才把焦点放回到扩展的配置上来，说明传递给config的参数包含库路径、名称信息没有传递到configure.in。

因此重新执行了phpize并重新config上库路径及名称,这次终于check成功，能成功链接，生成扩展符号表信息当然也正确。

# 4.总结
1. AC_CHECK_LIB的机制，在编译时而非链接时，致使链接器部分链接时选项失效。

2. 即使.so文件被拿来静态链接，链接时仍须遵守链接时命名规则。

这也是众多自动安装工具(如yum/brew)安装完一个.so后，会自动软链一个不含版本号到原因。



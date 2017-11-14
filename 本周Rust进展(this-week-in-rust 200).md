本周Rust进展(this-week-in-rust 200)
# 1.新闻与博客

- 宣布进入rust "实施互动期" ("impl period")

  完整链接[https://blog.rust-lang.org/2017/09/18/impl-future-for-rust.html]
  
  一直以2017路线图(2017 roadmap)努力工作，但在本年度最后一个季度，有一个更加崇高的目标——希望你能加入我们!(we want you to join us!)
  
  本年底雄心勃勃的目标有：
  
 - Rust应该有较低的学习曲线
 - Rust应该是个编辑-编译-调试过程
 - Rust应该提供稳定基础的IDE体验
 - Rust应该提供易使用的高质量crates
 - Rust应该能完备写出稳定高性能服务器
 - Rust应该有1.0级的crates完成基本功能
 - Rust应该易于集成到大型构建系统中
 - Rust社区应该在各种级别提供指导
 
  为达成目标，本年度剩下时间专注于"实施"工作——不仅仅是代码。2017 已merge了近90个RFC，将会放慢RFC节奏。
  
  每个Rust团队已经组织了几个专注于特定子区域的工作组。每个工作组负责知识管理，并配有一个聊天渠道来互动。
  
  所以如果你一直想为Rust贡献，但不知道如何做贡献，这是你的绝佳机会。不要害羞——我们想要并需要你的帮助，按照我们的路线图，我们的目标是在各个层次的经验上进行指导。
  
  在线联系渠道：
  
  专门的gitter社区   [https://gitter.im/rust-impl-period/]
  
  全新的开放式网站    [https://www.rustaceans.org/findwork]
  
  两个现场活动，配合即将举办的Rust大会:
  
  October 2-3 at RustFest.          [http://blog.rustfest.eu/this-week-in-rustfest-9-impl-days]
  October 24-25 at Rust Belt Rust.  [https://docs.google.com/forms/d/e/1FAIpQLSfmC_2XHsZ_jUKKR3eyQJygN2FsikjVhNwPypLScw5X3GXATQ/viewform]
  
  工作组阵容
  
  编译团队
  - WG-编译器错误	       使Rust的错误信息更加友善。	           	
  - WG-编译器前端         用解析和语法糖滋润你。	        
  - WG-编译器中端         实现关于类型检查的功能。
  - WG-编译器特性	       关联类型如何泛型？手把手教你怎么做。
  - WG-编译器增量	       完成增量编译; 收获满满的爱。	
  - WG-编译器NLL	        深入理解借用：NLL！	
  - WG-编译器常量	       常量泛型。够说的了。	
  
  库团队
  - WG-库-blitz	        在所有问题消失之前解决blitz!
  - WG-库-cookbook	      小例子带你玩转Rust。	
  - WG-库-guidelines     blitz中获取营养并传递下去。    	
  - WG-库-SIMD	          用Rust提供硬件并行！	
  - WG-库-OpenSSL	      想要更好的openssl文档么？我们提供。	
  - WG-库-rand           封装一个稳定核心的随机crate           	

  文档团队
  - WG-文档-rustdoc	     文档变得更优美!
  - WG-文档-rustdoc2	   自下而上的改造rustdoc文档!
  - WG-docs的-rbe	      在浏览器中教其他人Rust。	


  开发工具团队  
  - WG-DEV-工具-RLS	     第一节课带你体验Rust的IDE。
  - WG-DEV-工具-vscode	 用VSCode改善对Rust IDE的体验。
  - WG-DEV-工具-clients	 实现新的RLS客户端：Atom,Sublime,Visual Studio等
  - WG-DEV-工具-IntelliJ 继续完善已经很丰富的Rust IDE体验。
  - WG-DEV-工具-rustfmt	 让你的Rust代码最漂亮！
  - WG-DEV-工具-rustup	 让你对Rust的第一印象更好！
  - WG-DEV-工具-clippy	 貌似你正在尝试写一个检查小工具。需要帮助么？
  - WG-DEV-工具-bindgen  使C和C++变得简单、自动、强大！	

  Cargo团队
  - WG-cargo-native     不再为本地代码的依赖关系而发愁。
  - WG-cargo-registries 不用crate.io支持自定义注册。
  - WG-cargo-pub-deps   Cargo教导您的依赖关系影响您的用户。
  - WG-cargo-integration 使用Cargo构建你的系统有多容易？

  框架团队
  - WG-infra-crates.io	手把手教你如何开发Rust Web应用程序
  - WG-infra-host	      保持Rust机器运行的服务管理。
  - WG-infra-rustbuild	简化编译器构建过程。

  核心团队
  - WG-core-site	      该网站正在大修; 欢迎塑造新内容！！
  - WG-infra-crates.io	让我们一起验证Rust变得更快。
  - WG-infra-crater	    归一化测试编译器与Rust生态系统。
  - WG-infra-secure	    实施Rust框架最佳实践！

- 用Rust进行嵌入式系统开发的书籍 《Discover the world of microcontrollers through Rust》
- Rust是最节约能耗的语言：SLE会议论文用工具及数据证明。
- 播客show_notes::cysk::rayon，安全、多线程、并行的Rust代码。

# 2.本周crate
rug 使用GMP、MPFR和MPC提供任意精度整数、有理数及浮点数。

# 3.呼吁参与
   通过Rust项目参与开放式问题 (FindWork).
   
   libz/blitz及其衍生crate semver.
   
# 4.rust核心更新
   - 合并了160个PR
   - unicode转义允许使用下划线，如`\u{10_FFFF}`
   - 惰性计算定长数组长度表达式`[T; T::A]`(为支持数组长度多态及常量泛型)
   - 删除`Box <ZeroSizeType>`激进优化
   - 修复中间表示MIR借用生命周期EndRegion与StorageDead先后顺序
   - 修复引用static右值提升
   - 修复新宏`trace_macros`在宏展开发生错误时不起作用
   - Rust编译器`queries`删除`HirId`、`Session::dep_graph`
   - `let` 绑定可以加`allow(unused_mut)`
   - 修复tab键(社区一般用4空格)缩进的错误提示
   - 每日自动构建系统添加运行miri测试套(是将来的类型系统更好用)
   - 自动检查使用分配器crate的类型(如allocator_system, allocator_jemalloc)
   - `DepGraphQuery`时预分配(节约内存)
   - 删除`rustc_bitflags`,使用crate `bitflags` 
   - 定制`<FlatMap as Iterator>::fold`
   - 简单构造`Ipv4Addr`及`Ipv6Addr`
   - 几个入口API(目前主要是BTreeMap、HashMap)添加方法`or_default`
   - `std::mem::ManuallyDrop`加特性
   - 实现`<Rc<Any>>::downcast`
   - 实现`?Sized`的`Arc`/`Rc`原始指针转换
   - 实现`{&mut Hasher, Box<Hasher>}`的hash特性
   - 删除`SliceExt::binary_search`借用域
   - 实现不安全的指针方法
   - 个性化定制const函数调用的特征
   - 删除废弃的语言项
   - 夯实  `iterator_for_each`、`drop_types_in_const`、`tcpstream_connect_timeout`、`compiler_fences`、`ord_max_min`
   
   
# 5.活动日程
   9.21——10.05期间，Rust的发布、社区团队会议、Rust巴黎峰会、RustFest大会、Rust开发团队与开发者线上线下互动等。
   
   (活动较为密集，基本每天都有)
  

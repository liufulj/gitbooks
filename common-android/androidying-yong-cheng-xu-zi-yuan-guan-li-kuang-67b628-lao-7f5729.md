## **概述**

Android应用程序主要由代码和资源组成。资源主要就是指那些与UI相关的东西，例如UI布局、字符串和图片等。代码和资源分开可以使得应用程序在运行时根据实际需要来组织UI。这样就可使得应用程序只需要编译一次，就可以支持不同的UI布局。这种特性使得应用程序在运行时可以适应不同的屏幕大小和密度，以及不同的国家和语言等。资源在Android应用程序编译的过程中，也会被编译成二进制格式。这是为了压缩资源存储空间，以及加快运行时的资源解析速度。Android应用程序在运行的时候，资源管理器AssetManager和Resources会根据当前的机器设置，即屏幕大小、密度、方向，以及国家、地区语言的信息，查找正确的资源，并且进行解析，最后将它们渲染在UI上。

主要讲Android应用程序资源的编译、打包，以及它们在运行时的查找、解析过程。了解Android应用程序资源管理框架，有助于我们更好地开发出能够适配多种机型的应用程序。

* Android资源框架概述
* Android资源编译过程
* Android资源查找过程

## **Android资源框架概述**

**背景**

* Android设备屏幕大小和密度各不相同
* Android设备运行在不同国家、地区和语言上
 
  **问题**
* 同一个应用程序需要支持不同的语言和UI布局
 
  **方案**
* 代码和资源分离

**流行的资源管理框架**

* Web开发：CSS文件
* MFC开发：RC文件
* WPF开发：XAML文件
* QT开发：QML文件
* COCOA开发：XIB文件
 
  **Android设备的多样性导致其资源组织和管理更复杂**

**资源分类**

* assets
* res
  * animator
  * anim
  * color
  * drawable
  * layout
  * menu
  * raw
  * values
  * xml

**资源目录组织**

* &lt;
  resources\_name
  &gt;
  -
  &lt;
  config\_qualifier
  &gt;
 
  ![](https://box.kancloud.cn/79f9879e75779316cae054cc1e6cc2be_571x536.jpg)

## **Android资源编译过程**

**Android资源编译框架**  
![](https://box.kancloud.cn/20700984aff294834247bd27bf2db80c_582x365.jpg)

**编译过程主要是将XML文件从文本格式编译为二进制格式**

* 二进制格式的XML文件占用空间更小。这是由于所有XML元素的标签、属性名称、属性值和内容所涉及到的字符串都会被统一收集到一个字符串资源池中去，并且会去重。有了这个字符串资源池，原来使用字符串的地方就会被替换成一个索引到字符串资源池的整数值，从而可以减少文件的大小。
* 二进制格式的XML文件解析速度更快。这是由于二进制格式的XML元素里面不再包含有字符串值，因此就避免了进行字符串解析，从而提高速度。

**为了支持运行时快速定位最匹配资源，编译过程还会有为资源生成ID，以及生成资源索引表**

* 赋予每一个非assets资源一个ID值，这些ID值以常量的形式定义在一个R.java文件中
* 生成一个resources.arsc文件，用来描述那些具有ID值的资源的配置信息，它的内容就相当于是一个资源索引表
* 资源ID是一个4字节的无符号整数，其中，最高字节表示Package ID，次高字节表示Type ID，最低两字节表示Entry ID
* Package ID相当于是一个命名空间，限定资源的来源。Android系统当前定义了两个资源命令空间，其中一个系统资源命令空间，它的Package ID等于0x01，另外一个是应用程序资源命令空间，它的Package ID等于0x7f。所有位于\[0x01, 0x7f\]之间的Package ID都是合法的，而在这个范围之外的都是非法的Package ID
* Type ID是指资源的类型ID。资源的类型有animator、anim、color、drawable、layout、menu、raw、string和xml等等若干种，每一种都会被赋予一个ID
* Entry ID是指每一个资源在其所属的资源类型中所出现的次序。注意，不同类型的资源的Entry ID有可能是相同的，但是由于它们的类型不同，我们仍然可以通过其资源ID来区别开来

**例子**  
![](https://box.kancloud.cn/69d42672979ebfb66efb9278eb92cea1_692x576.png)

### **Step 1: 解析AndroidManifest.xml**

* 为了获得要编译资源的应用程序的包名称。
* 在AndroidManifest.xml文件中，manifest标签的package属性的值描述的就是应用程序的包名称。
* 有了这个包名称之后，就可以创建一个资源表\(Resource Table\)

### **Step 2:添加被引用资源包**

* android:orientation=“vertical”中的vertical实际上是由系统定义的一个资源。
* 在AOSP中，系统资源经过编译后，位于out/target/common/obj/APPS/framework-res\_intermediates/package-export.apk文件中。
* 因此，在AOSP中编译的应用程序资源，都会引用到系统资源包package-export.apk

### **Step 3:  收集资源文件**

* 在我们的例子中，包含三种类型的资源drawable、layout和values
* 名称相同的资源归为一组。例如，名称为icon.png的资源项res/drawable-ldpi/icon.png、res/drawable-mdpi/icon.png和res/drawable-hdpi/icon.png归为一组。它们通过屏幕密度\(ldpi、mdpi和hdpi\)来区分

### **Step 4:将收集到的非values资源增加到资源表**

![](https://box.kancloud.cn/bbb9fbb0df6d8727157c09e5541dfd5e_644x158.jpg)

### **Step 5: 编译values类资源**

* strings.xml文件的内容
 
  ![](https://box.kancloud.cn/cbd1ade022c970b975d0dfbd1320eef4_597x148.png)
* strings.xml文件编译后得到的资源项
 
  ![](https://box.kancloud.cn/45f32485df5033535350c6530d2b3e16_641x155.jpg)

### **Step 6: 给自定义资源分配资源ID**

* 假设在res/values目录下有一个attrs.xml文件
 
  ![](https://box.kancloud.cn/7617a73f9dd440a227aae9cd689bf5f9_434x132.png)
* attrs.xml文件被解析后
 
  ![](https://box.kancloud.cn/05828932250d420a448b380dcff1bb5e_643x105.jpg)

### **Step 7: 编译Xml资源文件**

* 以res/layout/main.xml为例
 
  ![](https://box.kancloud.cn/aa038a3a3cc90ba4a382c401dd7a44ee_522x376.png)

#### **Step 7.1: 解析xml文件**

* 解析Xml文件是为了可以在内存中用一系列树形结构的XMLNode来表示

#### **Step 7.2:赋予属性名称资源ID**

* 对于main.xml的根节点LinearLayout来说，就是要将它的属性名称android:orientation、android:layout\_width、android:layout\_height和android:gravity转换为资源ID

### **Step 7.3: 解析属性值**

* 例如，对于main.xml的属性android:orientation来说，它的合法取值为horizontal或者vertical，这里需要进行验证，并且将它们转换为ID值
* 有些属性值是属于引用类型的，例如main.xml文件的两个Button节点的android:id属性值”@+id/button\_start\_in\_process”和”@+id/button\_start\_in\_new\_process” ，将会生成新的资源项
 
  ![](https://box.kancloud.cn/1434213d6f6500cc1c136f38acab2b2e_643x87.jpg)

> **注意**，一个资源项一旦创建之后，要获得它的资源ID是很容易的，因为它的Package ID、Type ID和Entry ID都是已知的。

### **Step 7.4：将xml文件从文本格式转换为二进制格式**

![](https://box.kancloud.cn/b530dd269ca0545ac42d7d0fdd6ec551_338x503.jpg)

#### **Step 7.4.1: 收集有资源ID的属性的名称字符串**

* 将收集到的字符串及其资源ID分别保存一个字符串池以及资源数组中
* 对于main.xml文件来说，具有资源ID的Xml元素属性的名称字符串有”orientation”、”layout\_width”、”layout\_height”、”gravity”、”id”和”text”，假设它们的资源ID分别为0x010100c4、0x010100f4、0x010100f5、0x010100af、0x010100d0和0x0101014f:
 
  ![](https://box.kancloud.cn/15b79bb8e71b47ef0f2640197129b144_668x136.jpg)

#### **Step 7.4.2: 收集其它字符串**

* 这一步收集的字符串是不具有资源ID的
* 对于main.xml文件来说，这一步收集到的字符串如下所示：
 
  ![](https://box.kancloud.cn/0cd2f3b140843f1a2be2c425a1a35a18_663x93.jpg)

#### **Step 7.4.3:写入Xml文件头**

![](https://box.kancloud.cn/1cd8a6e2bd873b91a01b1d6aa297e236_561x330.png)  
![](https://box.kancloud.cn/d2d5c4fe31e0386d2894c28f9dbc39c8_489x72.png)

**对于ResXMLTree\_header头部来说，内嵌在它里面的ResChunk\_header的成员变量的值如下所示**：

* **type**
  ：等于RES\_XML\_TYPE，描述这是一个Xml文件头部。
* **headerSize**
  ：等于sizeof\(ResXMLTree\_header\)，表示头部的大小。
* **siz**
  e：等于整个二进制Xml文件的大小，包括头部headerSize的大小。

#### **Step 7.4.4:写入字符串资源池**

* 对于main.xml来说，依次写入的字符串为”orientation”、”layout\_width”、”layout\_height”、”gravity”、”id”、"text"、"android"、”
  [http://schemas.android.com/apk/res/android”](http://schemas.android.com/apk/res/android%E2%80%9D)
  、”LinearLayout”和”Button”
* 写入的字符串池同样有一个头部
 
  ![](https://box.kancloud.cn/04492d40a64ab5a3139827ea51e80506_579x503.png)

**内嵌在ResStringPool\_header里面的ResChunk\_header的成员变量的值如下所示**：  
**type**：等于RES\_STRING\_POOL\_TYPE，描述这是一个字符串资源池。  
**headerSize**：等于sizeof\(ResStringPool\_header\)，表示头部的大小。  
**size**：整个字符串chunk的大小，包括头部headerSize的大小。

**ResStringPool\_header的其余成员变量的值如下所示**：  
**stringCount**：等于字符串的数量。  
**styleCount**：等于字符串的样式的数量。  
**flags**：等于0、SORTED\_FLAG、UTF8\_FLAG或者它们的组合值，用来描述字符串资源串的属性，例如，SORTED\_FLAG位等于1表示字符串是经过排序的，而UTF8\_FLAG位等于1表示字符串是使用UTF8编码的，否则就是UTF16编码的。  
**stringsStart**：等于字符串内容块相对于其头部的距离。  
**stylesStart**：等于字符串样式块相对于其头部的距离。

#### **Step 7.4.4:字符串池的结构**

![](https://box.kancloud.cn/ad741ae940436e53914b61ba62d75887_573x389.jpg)

#### **Step 7.4.4：样式字符串**

* 假设有一个字符串”
  `<`
  `b`
  `>`
  `man`
  `<`
  `/b`
  `>`
  `<`
  `i`
  `>`
  `go`
  `<`
  `/i`
  `>`
  ”，实际上包含三个字符串”mango”、”b”和”I”，其中”mango”来有两个sytle，第一个style表示第1到第3个字符是粗体的，第二个style表示第4到第5个字符是斜体的
* 样式字符串由ResStringPool\_span和ResStringPool\_ref两个结构描述

以字符串“mango”的第一个样式描述为例，对应的ResStringPool\_span的各个成员变量的取值为：

* **name**
  ：等于字符串“b”在字符串资源池中的位置。
* **firstChar**
  ：等于0，即指向字符“m”。
* **lastChar**
  ：等于2，即指向字符"n"。

综合起来就是表示字符串“man”是粗体的。

再以字符串“mango”的第二个样式描述为例，对应的ResStringPool\_span的各个成员变量的取值为：

* **name**
  ：等于字符串“i”在字符串资源池中的位置。
* **firstChar**
  ：等于3，即指向字符“g”。
* **lastChar**
  ：等于4，即指向字符“o”。

综合起来就是表示字符串“go”是斜体的。

另外有一个地方需要注意的是，字符串样式内容的最后会有8个字节，每4个字节都被填充为ResStringPool\_span::END，用来表达字符串样式内容结束符。

#### **Step 7.4.5:写入资源ID**

* 资源ID数据块位于字符串池后面，它的头部使用ResChunk\_header来描述
* 以main.xml为例，字符串资源池的第一个字符串为”orientation”，而在资源ID数据块记录的第一个数据为0x010100c4，那么就表示属性名称字符串”orientation”对应的资源ID为0x010100c4

资源ID数据块ResChunk\_header各个成员变量的含义  
**type**：等于RES\_XML\_RESOURCE\_MAP\_TYPE，表示这是一个从字符串资源池到资源ID的映射头部。  
**headerSize**：等于sizeof\(ResChunk\_header\)，表示头部大小。  
**size**：等于headerSize的大小再加上sizeof\(uint32\_t\) \* count，其中，count为收集到的资源ID的个数。

#### **Step 7.4.6:压平Xml文件**

* 压平Xml文件其实就是指将里面的各个Xml元素中的字符串都替换掉。这些字符串要么是被替换成到字符串资源池的一个索引，要么是替换成一个具有类型的其它值。

**Step 7.4.6.1:首先被压平的是一个表示命名空间的Xml Node**

* 这个Xml Node用两个ResXMLTree\_node和两个ResXMLTree\_namespaceExt来表示：
 
  ![](https://box.kancloud.cn/7e67125ff4b5e9f35fdb9ac2528e3aad_321x287.jpg)
 
  **Step 7.4.6.1： ResXMLTree\_node和ResXMLTree\_namespaceExt的定义**
 
  ![](https://box.kancloud.cn/4fe97c344943b12d87882c8a8c978005_553x415.png)

对于main.xml文件来说，在它的命名空间chunk中，内嵌在第一个ResXMLTree\_node里面的ResChunk\_header的各个成员变量的取值如下所示：

* **type**
  ：等于RES\_XML\_START\_NAMESPACE\_TYPE，表示命名空间开始标签的头部。
* **headerSize**
  ：等于sizeof\(ResXMLTree\_node\)，表示头部的大小。
* **size**
  ：等于sizeof\(ResXMLTree\_node\) + sizeof\(ResXMLTree\_namespaceExt\)。

第一个ResXMLTree\_node的其余成员变量的取值如下所示：

* **lineNumber**
  ：等于命名空间开始标签在原来文本格式的Xml文件出现的行号。
* **comment**
  ：等于命名空间的注释在字符池资源池的索引。
* 内嵌在第二个ResXMLTree\_node里面的ResChunk\_header的各个成员变量的取值如下所示：
* **type**
  ：等于RES\_XML\_END\_NAMESPACE\_TYPE，表示命名空间结束标签的头部。
* **headerSize**
  ：等于sizeof\(ResXMLTree\_node\)，表示头部的大小。
* **size**
  ：等于sizeof\(ResXMLTree\_node\) + sizeof\(ResXMLTree\_namespaceExt\)。

第二个ResXMLTree\_node的其余成员变量的取值如下所示：

* **lineNumber**
  ：等于命名空间结束标签在原来文本格式的Xml文件出现的行号。
* **comment**
  ：等于0xffffffff，即-1。

两个ResXMLTree\_namespaceExt的内容都是一样的，它们的成员变量的取值如下所示：

* **prefix**
  ：等于字符串“android”在字符串资源池中的索引。
* **uri**
  ：等于字符串“
  [http://schemas.android.com/apk/res/android”](http://schemas.android.com/apk/res/android%E2%80%9D)
  在字符串资源池中的索引。

**Step 7.4.6.2:接下来被压平的是标签为LinearLayout的Xml Node**

* 这个Xml Node由两个ResXMLTree\_node、一个ResXMLTree\_attrExt、一个ResXMLTree\_endElementExt和四个ResXMLTree\_attribute来表示
 
  ![](https://box.kancloud.cn/eadb917624ad2309efceb712bda55eea_330x587.jpg)

第一个ResXMLTree\_node表示一个Node的开始，最后面的ResXMLTree\_node及其后面的ResXMLTree\_endElementExt表示一个Node的结束

**Step 7.4.6.2: ResXMLTree\_attrExt的定义**  
![](https://box.kancloud.cn/e6d99e08624733eaa4fcbfd8f4a5b51c_569x532.png)

ResXMLTree\_attrExt的各个成员变量的取值如下所示：

* **ns**
  ：等于LinearLayout元素的命令空间在字符池资源池的索引，没有指定则等于-1。
* **name**
  ：等于字符串“LinearLayout”在字符池资源池的索引。
* **attributeStart**
  ：等于sizeof\(ResXMLTree\_attrExt\)，表示LinearLayout的属性chunk相对type值为RES\_XML\_START\_ELEMENT\_TYPE的ResXMLTree\_node头部的位置。
* **attributeSize**
  ：等于sizeof\(ResXMLTree\_attribute\)，表示每一个属性占据的chunk大小。
* **attributeCount**
  ：等于4，表示有4个属性chunk。
* **idIndex**
  ：如果LinearLayout元素有一个名称为“id”的属性，那么就将它出现在属性列表中的位置再加上1的值记录在idIndex中，否则的话，idIndex的值就等于0。
* **classIndex**
  ：如果LinearLayout元素有一个名称为“class”的属性，那么就将它出现在属性列表中的位置再加上1的值记录在classIndex中，否则的话，classIndex的值就等于0。
* **styleIndex**
  ：如果LinearLayout元素有一个名称为“style”的属性，那么就将它出现在属性列表中的位置再加上1的值记录在styleIndex中，否则的话，styleIndex的值就等于0。

**Step 7.4.6.2: ResXMLTree\_attrExt的定义**  
![](https://box.kancloud.cn/baa8c722f0a26a8cc9c815574aa587fa_425x258.png)

LinearLayout元素有四个属性，每一个属性都对应一个ResXMLTree\_attribute，接下来我们就以名称为“orientation”的属性为例，来说明它的各个成员变量的取值，如下所示：

* **ns**
  ：等于属性orientation的命令空间在字符池资源池的索引，没有指定则等于-1。
* **name**
  ：等于属性名称字符串“orientation”在字符池资源池的索引。
* **rawValue**
  ：等于属性orientation的原始值“vertical”在字符池资源池的索引，这是可选的，如果不用保留，它的值就等于-1。

**Step 7.4.6.2: Res\_value的定义**  
![](https://box.kancloud.cn/e761e87f5083dc2253b481ced9ccd3fb_688x496.png)  
**Step 7.4.6.2: ResXMLTree\_endElementExt的定义**  
![](https://box.kancloud.cn/94aabd64747dddd22833591b51f2b8bc_488x233.png)  
**ns**：等于LinearLayout元素的命令空间在字符池资源池的索引，没有指定则等于-1。  
**name**：等于字符串“LinearLayout”在字符池资源池的索引。

**Step 7.4.6.3**:对于一个Xml文件来说，它除了有命名空间和普通标签类型的Node之外，还有一些称为CDATA类型的Node

* 假设一个Xml文件，它的一个Item标签的内容如下所示:  
  ![](https://box.kancloud.cn/5ce9581faa4ee84494a52915776303cf_312x92.png)

* 字符串“This is a normal text”就称为一个CDATA，它在二进制Xml文件中用一个ResXMLTree\_node和一个ResXMLTree\_cdataExt来描述  
  ![](https://box.kancloud.cn/64a470f94fd4c7e1aac398805000ada6_305x157.jpg)  
  **Step 7.4.6.3**: ResXMLTree\_cdataExt的定义  
  ![](https://box.kancloud.cn/ccaba908ac8eded921ef059c78647b15_501x215.png)

内嵌在上面的ResXMLTree\_node的ResChunk\_header的各个成员变量的取值如下所示：

* **type**
  ：等于RES\_XML\_CDATA\_TYPE，表示CDATA头部。
* **headerSize**
  ：等于sizeof\(ResXMLTree\_node\)，表示头部的大小。
* **size**
  ：等于sizeof\(ResXMLTree\_node\) + sizeof\(ResXMLTree\_cdataExt\) 。

上面的ResXMLTree\_node的其余成员变量的取值如下所示：

* **lineNumber**
  ：等于字符串“This is a normal text”在原来文本格式的Xml文件出现的行号。

下面的ResXMLTree\_cdataExt的成员变量data等于字符串“This is a normal text”在字符串资源池的索引，另外一个成员变量typedData所指向的一个Res\_value的各个成员变量的值如下所示：

* **size**
  ：等于sizeof\(Res\_value\)。
* **res0**
  ：等于0，保留给以后用。
* **dataType**
  ：等于TYPE\_NULL，表示没有包含数据，数据已经包含在ResXMLTree\_cdataExt的成员变量data中。
* **data**
  ：等于0，由于dataType等于TYPE\_NULL，这个值是没有意义的。

### **Step 8:生成资源符号**

* 这里生成资源符号为后面生成R.java文件做好准备的
* 目前所有收集到的资源项都按照类型来保存在一个资源表中
* 只要依次遍历资源表中的每一个Package的每一个Type的每一个Entry，就可以生成一系列的资源符号及其ID
* 例如对于strings.xml文件中名称为”start\_in\_process”的Entry来说，它是一个类型为string的资源项，假设它出现的次序为第3，那么它的资源符号就等于R.string.start\_in\_process，对应的资源ID就为0x7f050002，其中，高字节0x7f表示Package ID，次高字节0x05表示string的Type ID，低两字节0x02就表示”start\_in\_process”是第三个出现的字符串

### **Step 9:生成资源索引表**

经过前面的八个操作，所获得的资源列表如下所示：  
![](https://box.kancloud.cn/5efa8cf02a036be0b420ae3ed7aeb3b6_622x296.jpg)

### **Step 9: 资源索引表生成过程**

![](https://box.kancloud.cn/a8000ef60efbdd30f56aa61e2fa58097_400x578.jpg)

#### **Step 9.1:收集类型字符串**

* 在我们的例子中，一共有4种类型的资源，分别是drawable、layout、string和id，于是对应的类型字符串就为“drawable”、“layout”、“string”和“id”
* 注意，这些字符串是按Package来收集的，也就是说，当前被编译的应用程序资源有几个Package，就有几组对应的类型字符串，每一个组类型字符串都保存在其所属的Package中

#### **Step 9.2:收集资源项名称字符串**

* 在我们的例子中，收集到的资源项名称字符串就为”icon”、”main”、”sub”、”app\_name”、”sub\_activity”、”start\_in\_process”、”start\_in\_new\_process”、”finish”、”button\_start\_in\_process”和”button\_start\_in\_new\_process”。
* 注意，这些字符串同样是按Package来收集的，也就是说，当前被编译的应用程序资源有几个Package，就有几组对应的资源项名称字符串，每一个组资源项名称字符串都保存在其所属的Package中。

#### **Step 9.3:收集资源项值字符串**

* 在我们的例子中，一共有12个资源项，但是只有10项是具有值字符串的，它们分别是”res/drawable-ldpi/icon.png”、”res/drawable-mdpi/icon.png”、”res/drawable-hdpi/icon.png”、”res/layout/main.xml”、”res/layout/sub.xml”、”Activity”、”Sub Activity”、”Start sub-activity in process”、”Start sub-activity in new process”和”Finish activity”。
* 注意，这些字符串不是按Package来收集的，也就是说，当前所有参与编译的Package的资源项值字符串都会被统一收集在一起。

#### **Step 9.4:生成Package数据块**

* 参与编译的每一个Package的资源项元信息都写在一块独立的数据上，这个数据块使用一个类型为ResTable\_package的头部来描述
 
  ![](https://box.kancloud.cn/bfb3d9bf0a441e927bbbff03820f26ba_358x406.jpg)

**Step 9.4.1**:写入Package资源项元信息数据块头部，即一个ResTable\_package结构体  
![](https://box.kancloud.cn/846192f8048d392a99772ad5cfc1f41f_484x503.png)  
嵌入在ResTable\_package内部的ResChunk\_header的各个成员变量的取值如下所示：

* **type**
  ：等于RES\_TABLE\_PACKAGE\_TYPE，表示这是一个Package资源项元信息数据块头部。
* **headerSize**
  ：等于sizeof\(ResTable\_package\)，表示头部大小。

* **size**
  ：等于sizeof\(ResTable\_package\) + 类型字符串资源池大小 + 资源项名称字符串资源池大小 + 类型规范数据块大小 + 数据项信息数据块大小。

ResTable\_package的其它成员变量的取值如下所示：

* **id**
  ：等于Package ID。
* **name**
  ：等于Package Name。
* **typeStrings**
  ：等于类型字符串资源池相对头部的偏移位置。
* **lastPublicType**
  ：等于最后一个导出的Public类型字符串在类型字符串资源池中的索引，目前这个值设置为类型字符串资源池的大小。
* **keyStrings**
  ：等于资源项名称字符串相对头部的偏移位置。
* **lastPublicKey**
  ：等于最后一个导出的Public资源项名称字符串在资源项名称字符串资源池中的索引，目前这个值设置为资源项名称字符串资源池的大小。

**Step 9.4.1: ResTable\_package结构体图示**  
![](https://box.kancloud.cn/fef55f1e51686f2c64636d7ccb78bdb3_323x423.png)  
**Step 9.4.1: public资源**

* 定义在res/values/public.xml文件，形式如下所示：
 
  ![](https://box.kancloud.cn/bddba85d79a4b9eb403e745cd4425fb3_426x72.jpg)
* 用来告诉Android资源编译工具将类型为string的资源string3的ID固定为0x7f040001，以便第三方应用程序可以固定地通过0x7f040001来访问字符串”string3”

**Step 9.4.2:写入类型字符串资源池**

* 前面已经将每一个Package用到的类型字符串收集起来了，因此，这里就可以直接将它们写入到Package资源项元信息数据块头部后面的那个数据块去。

**Step 9.4.3:写入资源项名称字符串资源池**

* 前面已经将每一个Package用到的资源项名称字符串收集起来了，这里就可以直接将它们写入到类型字符串资源池后面的那个数据块去。

**Step 9.4.4:写入类型规范数据块**

* 类型规范数据块用来描述资源项的配置差异性。通过这个差异性描述，我们就可以知道每一个资源项的配置状况。
* 知道了一个资源项的配置状况之后，Android资源管理框架在检测到设备的配置信息发生变化之后，就可以知道是否需要重新加载该资源项。
* 类型规范数据块是按照类型来组织的，也就是说，每一种类型都对应有一个类型规范数据块。

**Step 9.4.4:类型规范数据块的头部用一个ResTable\_typeSpec来定义**  
![](https://box.kancloud.cn/ef74b11d710e078feb891bebad14d4b0_479x395.png)  
嵌入在ResTable\_typeSpec里面的ResChunk\_header的各个成员变量的取值如下所示：

* **type**
  ：等于RES\_TABLE\_TYPE\_SPEC\_TYPE，用来描述一个类型规范头部。
* **headerSize**
  ：等于sizeof\(ResTable\_typeSpec\)，表示头部的大小。
* **size**
  ：等于sizeof\(ResTable\_typeSpec\) + sizeof\(uint32\_t\) \* entryCount，其中，entryCount表示本类型的资源项个数。

ResTable\_typeSpec的其它成员变量的取值如下所示：

* **id**
  ：表示资源的Type ID。
* **res0**
  ：等于0，保留以后使用。
* **res1**
  ：等于0，保留以后使用。
* **entryCount**
  ：等于本类型的资源项个数，注意，这里是指名称相同的资源项的个数。

**Step 9.4.4**: ResTable\_typeSpec后面紧跟着的是一个大小为entryCount的uint32\_t数组，

* 每一个数组元数，即每一个uint32\_t，都是用来描述一个资源项的配置差异性的
* Android资源管理框架根据这个差异性信息，就可以知道当设备配置发生变化时，是否需要重新加载资源

**Step 9.4.4: 在我们的例子中，类型为drawable的规范数据块内容：**  
![](https://box.kancloud.cn/70c6472f7acdab17d5840885a57a9dc7_592x112.jpg)

* 由此可知，类型为drawable的资源项icon在设备的屏幕密度发生变化之后，Android资源管理框架需要重新对它进行加载，以便获得更合适的资源项

**Step 9.4.4: 在我们的例子中，类型为layout的规范数据块内容：**  
![](https://box.kancloud.cn/0c1847ff39ebac934eb8b9f46fc060fc_591x141.jpg)

* 由此可知，类型为layout的资源项在设备配置变化之后，Android资源管理框架无需重新对它们进行加载，因为只有一种资源

**Step 9.4.4: 在我们的例子中，类型为string的规范数据块内容**：  
![](https://box.kancloud.cn/e0e461392e012ef123ea442464461e16_692x223.jpg)

* 由此可知，类型为string的资源项在设备配置变化之后，Android资源管理框架无需重新对它们进行加载，因为只有一种资源

**Step 9.4.5:写入类型资源项数据块**

* 类型资源项数据块用来描述资源项的具体信息， 这样我们就可以知道每一个资源项名称、值和配置等信息。类型资源项数据同样是按照类型和配置来组织的，也就是说，一个具有N个配置的类型一共对应有N个类型资源项数据块。

**Step 9.4.5:类型资源项数据块的头部用一个ResTable\_type来定义**  
![](https://box.kancloud.cn/b494b6615c241e8fb57e01886c00dcff_480x488.png)  
嵌入在ResTable\_type里面的ResChunk\_header的各个成员变量的取值如下所示：

* **type**
  ：等于RES\_TABLE\_TYPE\_TYPE，用来描述一个类型资源项头部。
* **headerSize**
  ：等于sizeof\(ResTable\_type\)，表示头部的大小。
* **size**
  ：等于sizeof\(ResTable\_type\) + sizeof\(uint32\_t\) \* entryCount，其中，entryCount表示本类型的资源项个数。

ResTable\_type的其它成员变量的取值如下所示：

* **id**
  ：表示资源的Type ID。
* **res0**
  ：等于0，保留以后使用。
* **res1**
  ：等于0，保留以后使用。
* **entryCount**
  ：等于本类型的资源项个数，注意，这里是指名称相同的资源项的个数。
* **entriesStart**
  ：等于资源项数据块相对头部的偏移值。
* **config**
  ：指向一个ResTable\_config，用来描述配置信息，它的定义可以参考图2的类图。

**Step 9.4.5**: ResTable\_type紧跟着的是一个大小为entryCount的uint32\_t数组，每一个数组元数，即每一个uint32\_t，都是用来描述一个资源项数据块的偏移位置。紧跟在这个uint32\_t数组后面的是一个大小为entryCount的ResTable\_entry数组，每一个数组元素，即每一个ResTable\_entry，都是用来描述一个资源项的具体信息。在我们的例子中，一共有4种不同类型的资源项，其中，类型为drawable的资源有1个资源项以及3种不同的配置，类型为layout的资源有2个资源项以及1种配置，类型为string的资源有5个资源项以及1种配置，类型为id的资源有2个资源项以及1种配置。这样一共就对应有3 + 1 + 1 + 1个类型资源项数据块。

**Step 9.4.5:类型为drawable和配置为ldpi的资源项数据块**  
![](https://box.kancloud.cn/7ae8c019633ffaf34c70a9e03b541fb3_468x185.jpg)  
**Step 9.4.5:类型为drawable和配置为mdpi的资源项数据块**  
![](https://box.kancloud.cn/e2f62f9292bd915a59d4dd4b1abb7345_473x180.jpg)  
**Step 9.4.5:类型为drawable和配置为hdpi的资源项数据块**  
![](https://box.kancloud.cn/e8a22cb42e106b669b17cb3ed22abc6e_471x182.jpg)  
**Step 9.4.5:类型为layout和配置为default的资源项数据块**  
![](https://box.kancloud.cn/27d60eb65f0449ea5c02c5973f6b7aa5_467x275.jpg)  
**Step 9.4.5:类型为string和配置为default的资源项数据块**  
![](https://box.kancloud.cn/68db9c7fe034c107ef3fa49cebc626d7_533x561.jpg)  
**Step 9.4.5:类型为id和配置为default的资源项数据块**  
![](https://box.kancloud.cn/e1cded49279be57b4da0f29e04e0f02a_477x278.jpg)  
**Step 9.4.5: 每一个资源项数据都是通过一个ResTable\_entry来定义的**  
![](https://box.kancloud.cn/9e80d5547361f195a89c1527d944f0d3_551x324.png)  
ResTable\_entry的各个成员变量的取值如下所示：

* **size**
  ：等于sizeof\(ResTable\_entry\)，表示资源项头部大小。
* **flags**
  ：资源项标志位。如果是一个Bag资源项，那么FLAG\_COMPLEX位就等于1，并且在ResTable\_entry后面跟有一个ResTable\_map数组，否则的话，在ResTable\_entry后面跟的是一个Res\_value。如果是一个可以被引用的资源项，那么FLAG\_PUBLIC位就等于1。
* **key**
  ：资源项名称在资源项名称字符串资源池的索引。

**Step 9.4.5: 普通资源项数据都是通过一个Res\_value来定义的**  
![](https://box.kancloud.cn/7c7ad7d164e7b47629ceb1061213c79b_216x141.jpg)  
**Step 9.4.5: 自定义资源项数据都是通过一个ResTable\_map\_entry以及若干个ResTable\_map来定义的**  
![](https://box.kancloud.cn/52d3f5fdd698849c60c15b2e648ee796_243x324.jpg)  
**Step 9.4.5: 以前面的自定义资源custom\_orientation为例**：  
![](https://box.kancloud.cn/2ca28b2a11e5e73807bd1c19180daa23_434x133.png)

* 它有三个bag，分别是^type、custom\_vertical和custom\_horizontal，其中，custom\_vertical和custom\_horizontal是两个自定义的bag，它们的值分别等于0x0和0x1，而^type是一个系统内部定义的bag，它的值固定为0x10000。 注意，^type、custom\_vertical和custom\_horizontal均是类型为id的资源，假设它们分配的资源ID分别为0x1000000、0x7f040000和7f040001。

**Step 9.4.5: ResTable\_map\_entry的定义如下所示**：  
![](https://box.kancloud.cn/4fc8b698c3d2355474bdf82d9de47da8_545x289.png)

ResTable\_map\_entry是从ResTable\_entry继承下来的，我们首先看ResTable\_entry的各个成员变量的取值：

* **size**
  ：等于sizeof\(ResTable\_map\_entry\)。
* **flags**
  ：由于在紧跟在ResTable\_map\_entry前面的ResTable\_entry的成员变量flags已经描述过资源项的标志位了，因此，这里的flags就不用再设置了，它的值等于0。
* **key**
  ：由于在紧跟在ResTable\_map\_entry前面的ResTable\_entry的成员变量key已经描述过资源项的名称了，因此，这里的key就不用再设置了，它的值等于0。

ResTable\_map\_entry的各个成员变量的取值如下所示：

* **parent**
  ：指向父ResTable\_map\_entry的资源ID，如果没有父ResTable\_map\_entry，则等于0。
* **count**
  ：等于bag项的个数。

**Step 9.4.5: ResTable\_map的定义如下所示:**  
![](https://box.kancloud.cn/8385fbe547d2dfaee22f45d1be012014_541x233.png)  
ResTable\_map只有两个成员变量，其中：

* **name**
  ：等于bag的资源项ID。
* **value**
  ：等于bag的资源项值。

例如，对于custom\_vertical来说，用来描述它的ResTable\_map的成员变量name的值就等于0x7f040000，而成员变量value所指向的一个Res\_value的各个成员变量的值如下所示：

* **size**
  ：等于sizeof\(Res\_value\)。
* **res0**
  ：等于0，保留以后使用。
* **dataType**
  ：等于TYPE\_INT\_DEC，表示data是一个十进制的整数。
* **data**
  ：等于0。

#### **Step 9.5:写入资源索引表头部**

资源索引表头部使用一个ResTable\_header来表示  
![](https://box.kancloud.cn/076bf756eea43fb4ed0e4acb65e5c4ba_512x295.png)  
嵌入在ResTable\_header内部的ResChunk\_header的各个成员变量的取值如下所示：

* **type**
  ：等于RES\_TABLE\_TYPE，表示这是一个资源索引表头部。
* **headerSize**
  ：等于sizeof\(ResTable\_header\)，表示头部的大小。
* **size**
  ：等于整个resources.arsc文件的大小。

ResTable\_header的其它成员变量的取值如下所示：

* packageCount：等于被编译的资源包的个数。

#### **Step 9.6:写入资源项的值字符串资源池**

* 前面已经将所有的资源项的值字符串都收集起来了，因此，这里直接它们写入到资源索引表去就可以了。
* 注意，这个字符串资源池包含了在所有的资源包里面所定义的资源项的值字符串，并且是紧跟在资源索引表头部的后面。

嵌入在ResTable\_header内部的ResChunk\_header的各个成员变量的取值如下所示：

* **type**
  ：等于RES\_TABLE\_TYPE，表示这是一个资源索引表头部。
* **headerSize**
  ：等于sizeof\(ResTable\_header\)，表示头部的大小。
* **size**
  ：等于整个resources.arsc文件的大小。

ResTable\_header的其它成员变量的取值如下所示：

* **packageCount**
  ：等于被编译的资源包的个数。

#### **Step 9.7:写入Package数据块**

* 前面已经所有的Package数据块都收集起来了，因此，这里直接将它们写入到资源索引表去就可以了。
* 这些Package数据块是依次写入到资源索引表去的，并且是紧跟在资源项的值字符串资源池的后面。

### **Step 10:编译AndroidManifest.xml文件**

* 经过前面的操作，应用程序的所有资源项就编译完成了，这时候就开始将应用程序的配置文件AndroidManifest.xml也编译成二进制格式的Xml文件。
* 之所以要在应用程序的所有资源项都编译完成之后，再编译应用程序的配置文件，是因为后者可能会引用到前者。

### **Step 11:生成R.java文件**

* 前面已经将所有的资源项及其所对应的资源ID都收集起来了，因此，这里只要将直接将它们写入到指定的R.java文件去就可以了。
* 例如，假设分配给类型为layout的资源项main和sub的ID为0x7f030000和0x7f030001，那么在R.java文件，就会分别有两个以main和sub为名称的常量，如下所示：
 
  ![](https://box.kancloud.cn/6c2ea7b9e5f7b3c429e7fdcd1223217f_385x180.png)

### **Step 12:打包资源文件到APK包**

* assets目录
* res目录，但是不包括values子目录的文件，这些文件的资源项已经包含在resources.arsc文件中
* resources.arsc
* AndroidManifest.xml

### **Android资源查找过程**

#### **Android资源查找框架**

![](https://box.kancloud.cn/c8733c36a23523247b9da8de26018bee_484x441.jpg)

#### **Android资源查找框架相关实现类**

![](https://box.kancloud.cn/32b31a529c8931d09203d23e11225521_448x441.jpg)  
每一个App Package都有一个Resources对象和一个AssetManager对象，用来查找本Package的资源，每一个App Process又都包含有一个Resources对象和一个AssetManager对象用来查找系统资源

#### **Android资源查找框架的初始化**

**设置资源路径**

* AssetManager::addAssetPath
  * /system/framework/framework-res.apk
  * /vendor/overlay/framework/framework-res.apk\(optional\)
  * Self Apk File
 
    **设置配置信息**
* AssetManager::setConfiguration

对于System Resources来说，资源路径只包含/system/framework/framework-res.apk和/vendor/overlay/framework/framework-res.apk\(optional\)

#### **根据资源ID找到资源Value**

* Step 1: 根据资源ID的Package ID在resources.arsc中找到对应的Package数据块\(ResTable\_package\)

* Step 2: 根据资源ID的Type ID在Package数据块找到对应的类型规范数据块\(ResTable\_typeSpec\)和类型资源项数据块\(ResTable\_type\)

* Step 3: 根据资源ID的Entry ID在对应的类型资源项数据块中找到与当前设备配置最匹配的资源项

* Step 4: 返回最匹配的资源项值及其配置差异性

* 找到的资源Value是一个文件路径，则通过AssetManager打开该文件，并且对它进行解析以及使用，否则直接使用

ResTable\_typeSpec和ResTable\_type均有一个id域，该id域描述的就是Type ID，因此根据资源ID的Type ID在Package数据块找到对应的类型规范数据块和类型资源项数据块  
每一个类型资源项数据块都包含有一个资源项数组，以Entry ID为索引，即可在资源项数组的资源项

对于一个包含多种配置的资源项来说，它就会对应多个ResTable\_type，因此需要根据当前设备配置最匹配的资源项找到最匹配的资源项

同时返回资源项的类型规范数据块\(描述配置差异性\)是为了让调用者知道它正在查找的资源有哪些配置，这样当设备配置发生变化时，就可以知道有没有必要更新资源项的值，即重新查找最匹配的资源项

#### **资源匹配算法**

![](https://box.kancloud.cn/77301361634282b79b22fc4f0cdf2fc3_361x463.jpg)

#### **资源匹配算法示例**

* 资源配置清单
 
  ![](https://box.kancloud.cn/a7e3c0508557c01d52a5fa31387ee1d1_236x128.jpg)
* 设备配置信息
 
  ![](https://box.kancloud.cn/92588de27206ff1992f9facfc54f6274_256x91.jpg)
* Step 1: 消除与设备配置冲突的drawable目录，即drawable-fr-rCA目录，因为设备设置的语言是en-GB：
 
  ![](https://box.kancloud.cn/82078a7edce38a9aea2f718deea9ddee_257x109.jpg)
* Step 2:从MMC开始，选择一个资源组织维度来过渡从Step 1筛选后剩下来的目录。
* Step 3:检查Step 2选择的维度是否有对应的资源目录。如果没有，就返回到Step 2继续处理。如果有，那么就继续往下执行Step 4。在我们示例中，要一直重复执行Step 2，直到检查到language这个维度时。
* Step 4: 消除那些不包含有Step 2所选择的资源维度的目录。在我们的示例中，就是要消除那些不包含有en这个language的目录：
 
  ![](https://box.kancloud.cn/ea3832ebed2d2f1aaaf5799d6da5a0c1_254x54.png)
* Step 5:继续执行Step 2、Step 3和Step 4，直到找到一个最匹配的资源目录为止，即剩下最后一个目录为止。在我们的示例中，下一个要检查的维度是screen orienation。由于设备的screen orienation为port。因此，所有不包含有port资源维度的目录将被消除
 
  ![](https://box.kancloud.cn/5df44014e52d9a15ba3375f4616cdac0_181x19.png)




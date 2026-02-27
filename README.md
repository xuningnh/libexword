# 个人经验总结

卡西欧的电子词典十分好用，但是价格昂贵，很多人难以一步到位直接买旗舰机，目前能廉价提升手头电子词典价值的方法主要分两种：
-   刷机: 使用别人已经导出好的高阶词典的系统镜像，在 Test Menu 刷机，可以将丐版的电子词典升级为旗舰机，操作最简单，但是前提是已经有人分享了镜像文件，并且需要是同一系列的电子词典才能刷机成功
    -   注意: 如果操作不慎刷机失败可能导致变砖
-   附加词典:不改变现有系统的情况下，附加新词典，风险较低
    -   注意：操作难度高，需要有IT开发相关经验才能上手

以上方法都可以选择上网找人代操作，但是价格也相对高，且为了保证升级文件不外泄，代刷机都需要邮寄

本人使用的是日版的DATAPLUS 4系列词典，该文档为自己参考论坛摸索出的附加词典的经验，用于帮助其他有需要同学

> **注意**: 刷机有风险，请务必备份您的数据，具体适用的词典型号请自行尝试。更详细的刷机步骤、Test Menu 功能说明和技术细节，请参阅 [**`docs/nand-and-nor.md`**](./docs/nand-and-nor.md)。

## 前期准备
-   **TF卡（可选）**: 准备一张2Ｇ左右的TF卡，较新机型可能支持的容量更大，我目前的词典支持上限仅2G但已经绰绰有余（机身自带容量较小只能安装少量词典）
- 词典文件：从GitHub仓库 [dictdump](https://github.com/dictdump/dictdump) 下载附加词典文件，词典文件在`addons-on-dictionary`目录下
    - **addons-on-dictionary**: 存放附加词典
        -   **addons-for-cn**: 仅用于国行词典
        -   `addons-for-ja`: 仅用于日版词典
-  虚拟*: 我使用的系统是*Ubuntu 24.04.4 LTS*，建议都用Ubuntu 24.04的版本，其他版本过程可能略有差异
-  libexword工具：[libexword](https://github.com/xuningnh/libexword): 这是我fork的很早期的开源工具，原工具在现在的Ubuntu 24.04上无法良好运行，适当做了一点修改，如果你的词典型号在**models.txt**里有记录，说明已经验证过可以直接使用，如果没有，可以尝试使用较新一点的 [libexword-re](https://github.com/caesarw/libexword-re)


## 新增附加词典步骤
以附加 **SG014**(日中辞典 第2版)词典为例:
1.  **上传词典文件夹**: 将词典文件夹(注意是`整个文件夹`)放在Ubuntu的`./local/share/exword/ja` 目录下(国行词典应该是`./local/share/exword/cn`，手头没有国行词典所以未测试)
2.  **启动libexword**: 大致过程如下，该步骤是整个过程中唯一门槛较高的操作，需要一定的开发技能，可能需要自己修改下代码并解决Ubuntu的环境问题
    ```bash
    ./autogen.sh

    ./configure
    
    make

    cd src

    ./exword
    ```
3.  **连接词典**: 使用数据线连接PC，点击词典上的"连接电脑"，在虚拟机选项里要保证设备连到Ubuntu而不是windows，切换到`root 用户`，然后按照以下流程操作
    ```bash
    connect:                连接词典
    dict reset <username>:  重置身份信息(仅初次连接需要，记住用户名)
    dict auth <username>:   认证身份
    setpath crd0://     :   设置数据路径为SD卡(可选，默认为词典内存)
    dict install SG014:     从Ubuntu向词典中安装附加词典，添加其他词典需改变对应编号
    ```
> **注意**: 如果没有切换成root，很有可能导致无法连接词典。

4.  **验证结果**: 待屏幕显示`[install] finished， rsp=0`后，证明安装成功，可以断开连接，检查是否新增词典


## libexword命令

* **connect** [mode] [region]  
    此命令连接到已连接的 EX-word 词典。它接受两个可选参数：第一个参数指定连接模式，可以是 library、text 或 CD；第二个参数是词典的区域，是一个两位字母的国家/地区代码。
    mode 和 region 的默认值分别为 library 和ja。

* **disconnect**  
  此命令会断开与当前连接的字典的连接。

* **model**  
  此命令显示有关已连接设备的原始型号信息。

* **capacity**  
  此命令显示当前所选存储介质的容量。

* **format**  
  此命令将格式化当前插入的 SD 卡。

* **list**  
  此命令将列出当前目录中的所有文件和目录。
  
  
## 文件操作

* **delete** <filename>  
  从已连接设备上当前指定的目录中删除指定的文件。

* **send** <filename>  
  此命令会将文件上传到连接的设备。应指定文件在本地文件系统中的完整路径。
* **get** <filename>  
  从连接的设备下载文件。应指定文件在本地文件系统中的完整保存路径。

* **setpath** <path>  
  此命令设置字典的当前路径。路径使用以下格式设置：<sd|mem://path>.

* **set** <option> [value]  
  此命令用于设置各种配置选项。如果未为指定选项指定值，则会打印其当前值。
	- Options:
		- debug - This option sets the debug level (0-5)
		- mkdir - This option tells setpath if it should create non-existent directories (yes|no)

* **dict** <sub-function>  
  此命令用于管理已安装的附加词典。它仅在以library mode连接时有效。 .
  
	* **reset** username 
		
		此操作将使用指定的用户名重置您的身份验证信息。完成后，它将显示新的身份验证密钥，并将用户名/身份验证密钥对保存到 users.dat 文件中。它还会删除所有已安装的插件。
		
	* **auth** username [key]  
		此函数执行身份验证，除 list 和 reset 之外，必须在任何其他子函数之前调用。如果未指定 key，则会尝试在 users.dat 文件中查找用户名。
		
	* **list**  
		此命令将列出当前已安装的字典。
		
	* **decrypt** <id>  
		此命令将下载并解密具有指定 ID 的字典。
		
	* **remove** <id>  
		此命令将删除具有指定 ID 的字典。
		
	* **install** <id>  
		此命令将安装具有指定 ID 的字典。

> **注意**: 部分词典的文件名为大写字母，如SG013，需先将文件名全部统一转为小写才能安装。

## 部分词典名称
以下为支持的附加词典列表：

| 标题 | 目录编号 |
|------|----------|
| 英辞郎 | AL100 |
| 脳鍛アプリ 反射神経 | CF006 |
| 脳鍛アプリ 順番記憶 | CF007 |
| 脳鍛アプリ 数字並べ替えパズル | CF009 |
| 脳鍛アプリ 漢字組み立てパズル | CF010 |
| 脳鍛アプリ 数字パズル 難問編 | CI001 |
| 脳鍛アプリ スピード計算 | CI002 |
| DUDEN独独辞典 | DD001 |
| 法律用語がわかる辞典 第2版 | JK004 |
| 現代用語の基礎知識2008 | JK007 |
| 理化学英和辞典 | KS001 |
| MACMILLAN English DICTIONARY | MM001 |
| Kanji Learner’s Dictionary | NT001 |
| [音声]中日辞典 第2版 | SG005 |
| 日中辞典 第2版 | SG006 |
| 中日･日中辞典 付録集 | SG007 |
| [音声]中日辞典 第2版 | SG013 |
| 日中辞典 第2版 | SG014 |
| 中日･日中辞典 付録集 | SG015 |
| [音声]朝鮮語辞典 | SG016 |
| ポケットプログレッシブ韓日辞典 | SG017 |
| ポケットプログレッシブ日韓辞典 | SG018 |
| 中日辞典 | SG029 |
| 日中辞典 | SG030 |
| 中日･日中辞典 付録集 | SG031 |
| 日汉大辞典 | ZH008 |
| 新编现代日语外来语词典 | ZH009 |
| 拉鲁斯法汉双解词典 | ZH010 |
| 精编汉法词典 | ZH011 |
| 新德汉词典(第三版) | ZH012 |
| 现代汉德词典(第二版) | ZH013 |
| [音声]クラウン仏和辞典 第5版 | SS013 |
| コンサイス和仏辞典 第3版 | SS014 |
| [音声]コンサイス露和辞典 第5版 | SS020 |
| コンサイス和露辞典 第3版 | SS021 |
| ラルース西西辞典 | SP001 |





# Credits
Brian Johnson [@brijohn](https://github.com/brijohn) is the creator of _exword_ power tool.  
Caesar Woo [@CaesarW](https://github.com/CaesarW) does the fork job and tries to bring modern compatibility to the software.  

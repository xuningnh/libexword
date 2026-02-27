[English](nand-and-nor.md) | [日本語](nand-and-nor.ja.md) | [简体中文](nand-and-nor.zh-Hans.md)

# NAND 与 NOR 闪存

本文档描述了 NAND 和 NOR 上使用的文件系统格式。主要区别在于 NOR 不使用块寻址，因此本文档中所有关于块的描述都可以理解为 NOR 内存中的字节地址。

## NAND 闪存更新镜像

NAND 闪存更新镜像包含两个头部和一个只读文件系统，该文件系统包含了内置词典的大部分数据文件。一小部分 FAT16 文件系统也位于实际的 NAND 闪存芯片上，这里存放着通过电脑传输到词典的文件和附加组件（Addons），但这不属于 NAND 更新镜像的一部分。

`nand_build` 是一个 NAND 构建工具，可以根据文件列表和 NAND 布局描述来创建一个用于 NAND 刷写的更新镜像。当前版本为 DP4, DP5 和 DP6 设备定义了布局。

### 布局

#### 头部 1 (Header 1)

更新镜像中实际存储了两个 `hdr1` 的副本。第一个位于偏移量 `0`，而第二个可以在偏移量 `(hdr1.hdr1_2_blk * hdr1.blocksize)` 处找到。

```c
struct hdr1 {
    char signature[12];    // CASIOPVOS???
    uint16_t magic;        // 0x55AA
    uint16_t blocksize ;   // 每块的字节数 (总是 0x200)
    uint32_t flash_nblks;  //
    uint32_t update_nblks; //
    uint32_t dword18;      // 总是 0
    uint32_t hdr1_2_blk;   // hdr1 第二个副本的块编号
    uint32_t dir_blk;      // 目录的起始块
    uint32_t dir_nblks;    // 目录的块长度
    uint32_t hdr2_blk;     // 第二个头部的块
    uint32_t hdr1_2_cpy;   // hdr1 第二个副本的块编号?? 总是和 hdr1_2_blk 相同
    uint32_t dword30;      // 总是 0xffffffff
    uint32_t dword34;      // 总是 0xffffffff
    uint32_t dword38;      //
    uint32_t dword3C;      //
    uint32_t dword40;      // 总是 0
    uint32_t dword44;      //
    char model[4];         // 4 个字符的型号字符串
    uint32_t dword4C;      //
    uint32_t img_len;      // OS 更新镜像文件的块长度
    uint32_t dword54;      //
    uint32_t dword58;      //
    uint32_t dword5C;      //
    uint32_t dword60;      //
    uint32_t dword64;      //
    char ext_model[8];     // 扩展型号字符串 - 仅限 DP5+，DP5 之前此字段用 0xff 填充
    char filler70[398];    //
    uint16_t checksum;     // 16 位头部校验和
};
```

#### 头部 2 (Header 2)

NAND 校验和 (`datachksum`) 是通过将 `hdr2` 之后到文件末尾的所有字节相加计算得出的。利用这一点，可以创建自定义或修改过的 NAND 镜像用于刷写。可以使用 `nand_chksums.py` 工具来检查 NAND 校验和。

```c
struct hdr2 {
    char signature[12];    // CASIOPVOS???
    uint32_t dwordC;       // 总是 0
    uint32_t dword10;      // 总是 0
    uint32_t dword14;      // 总是 0xffffffff
    char model[4];         // 型号字符串
    char datetime[8];      // 日期时间 (BCD 编码)
    uint16_t version;      // 版本号 (BCD 编码)
    char padding[2];       // 填充，应为零
    uint32_t dword28;      // 总是 0
    uint32_t datachksum;   // NAND 校验和 - hdr2 之后所有字节的总和
    char ext_model[8];     // 扩展型号字符串 - 仅限 DP5+，DP5 之前此字段用 0xff 填充
    char filler2C[454];    //
    uint16_t checksum;     // 16 位头部校验和
};
```

### NAND 文件夹目录结构

以下是 NAND 中找到的文件夹列表及其功能猜测。大多数文件夹，如 `cz???` (中国型号)，是词典文件夹。

-   `ttslib`: 富士通（FUJITSU LIMITED 1994）的文本转语音库。用于语音合成词典，采用 11.025 kHz 16bit 格式。
-   `txtv`: 未知。
-   `voice`: 词典语音文件。
-   `henka`: "henka" (変化) 在日语中意为变化、转换。移除此文件夹后，内置词典中的“模糊搜索”将显示“已取消”。
-   `fukuE`: "fuku" (副) 可能指副词或次要功能。用于英语单词的“跳查搜索”（左上角的第三个按钮）。如果移除，跳查英语单词会跳转到“我的图书馆”。
-   `fukuJ`: 用于中文或日文单词的“跳查搜索”。
-   `font`: 存放 `.cjf` 字体文件，可以被转储。
-   `czMDE`: 在中国型号中找到。可能是功能菜单中的“多词典英语搜索”。
-   `czMDJ`: 在中国型号中找到。可能是功能菜单中的“多词典日语搜索”。
-   `libr`: 未知。
-   `menu_cn`: 中文菜单资源。如果移除，菜单显示将为空白，但功能仍然有效。
-   `menu_jp`: 日语菜单资源。
-   `menu_en`: 英语菜单资源。
-   `sys`: 系统文件夹？
-   `demo`: 在中国型号中，此文件夹为空。

## TJFS 文件系统

NAND ROM 上使用的文件系统（推测为 "TJFS"）相当简单，因为它是只读的，并且一旦刷入 ROM 就不支持修改。由于文件系统是只读的，文件都存储在连续的块上，不存在文件碎片。文件系统的根目录只能包含目录，且不允许嵌套目录。

### 主目录表 (Master Directory Table)

该表包含文件系统中所有文件和目录的列表。每个条目长 32 字节，首先列出目录，然后是每个目录中包含的所有文件。所有值都以大端格式存储。

```c
struct DirEntry {
    char filename[16];
    /*
     * 以空字符结尾的字符串，包含目录或文件的名称。
     */
    short dir_id;
    /*
     * 对于文件，此值总是 0xffff。
     * 对于目录，这是一个唯一的 ID，用于将文件与目录关联。
     */
    short dir_ptr;
    /*
     * 对于目录，此值总是 0xffff。
     * 对于文件，此值指向其所在目录的 dir_id。
     */
    unsigned long location;
    /*
     * 如果条目是目录，此字段是一个指向主目录表的从 0 开始的索引，
     * 指示该目录的文件列表的起始位置。
     * 如果条目是文件，此字段表示文件的起始块号（块大小为 512 字节）。
     */
    unsigned long size;
    /*
     * 文件的字节大小。对于目录，此值总是 0。
     */
    unsigned long flags;
    /*
     * 条目的标志字段。有效值目前未知。
     */
};
```

## 测试菜单 (Test Menu)

卡西欧电子词典有一个隐藏的工程模式（Test Menu）。

**进入方法:**
1.  关机。
2.  按住 **退出 + 上翻页 + 电源键** 约五秒钟，直到屏幕出现显示 "Model" 的窗口。
3.  松开按键，然后按 **两下右键**，再按 **输入键**。

### 菜单选项说明

-   **TJFS 物理格式化 (TJFS Physical Format)**: 需要密码。似乎会对 TJFS 文件系统进行快速格式化。执行后，设备在开机时会卡在“请稍候…”界面。
-   **强制深度格式化 (Force Strong Format)**: 擦除 NAND。输入密码后有三个选项：
    -   `NACE1`: 擦除 `0000` 到 `0FFF`。过程很慢，逐块进行。可能会显示坏块计数。
    -   `NACE2`: 在某些设备上显示 “CAPA NG”。
    -   `NACE3`: 似乎工作不正常。
    执行后，开机可能会显示“错误代码 01”。奇怪的是，即使格式化后，设备似乎仍能正常启动，此功能的具体作用未知。
-   **数据写入 (DATA IN)**: 需要密码。从 TF 卡上的 `DATAOUT.BIN` 文件写入 NAND。如果遇到擦除错误（可能由坏块引起），进程会停止，并可能导致开机时出现错误代码。
-   **数据导出 (DATA OUT)**: 从 NAND 导出数据到 TF 卡，生成 `DATAOUT.BIN` 文件。导出的数据可能包含因 ECC（错误更正码）未能修复而产生的随机位翻转错误。多次导出的数据可能不一致。
    -   如果生成了多个 `DATAOUT` 文件（如 `DATAOUT1.BIN`, `DATAOUT2.BIN`），可以使用 `copy /b DATAOUT1.BIN+DATAOUT2.BIN DATAOUT.BIN` (Windows) 或 `cat DATAOUT1.BIN DATAOUT2.BIN > DATAOUT.BIN` (Linux/macOS) 将它们合并。
    -   可以使用 `test_menu_dump.py` 从 `DATAOUT.BIN` 中提取文件。

## 操作系统更新 (OS UPDATE)

以下是使用生成的 E-A800 的 NOR 和 NAND 文件成功进行操作系统更新的示例过程。

1.  **格式化 TF 卡**: 在词典的 Test Menu 中格式化一张 TF 卡（推荐 2GB）。
2.  **裁剪 NOR 文件**:
    -   使用已移除密码的 NOR 文件，例如 `CY123OSB.BIN`。
    -   使用十六进制编辑器（如 `hexdump`）找到文件末尾的有效数据边界，并使用 `truncate` 等工具裁剪掉末尾多余的 `FF` 填充字节。
        ```
        hexdump -C NOR.BIN | tail -n 10 查找文件的末尾。得到如下结果。

        016ecdb0  55 55 55 55 55 55 55 55  55 50 ff ff ff ff ff ff  |UUUUUUUUUP......|
        016ecdc0  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
        偏移量 016ecdc0 的十进制是 24038848。使用 truncate -s 24038848 NOR.BIN 来裁剪文件。
        ```
    
    -   将裁剪后的文件重命名为 `CY123OSB.BIN` 并放入 TF 卡。更新程序会选择字母顺序最新的文件（例如，`CY123OSE.BIN` 会优先于 `CY123OSB.BIN`）。
    -   使用 `nor_parse.py` 从 NOR 镜像中解包出 `UPDADN3.BIN`（或 `UPDADN2.BIN`）并也放入 TF 卡。
3.  **生成 NAND 镜像**:
    -   使用 `nand_build.py` 构建 NAND 刷机镜像：
        ```sh
        python nand_build.py --layout dp5 --model C123 --ext-model CY123 files_folder CY123D0B.BIN
        ```
    -   `files_folder` 中的文件推荐使用 `copy_nand` 插件从词典上复制，以避免 `DATAOUT` 引入的错误。
    -   将生成的 NAND 镜像（例如 `CY123D0B.BIN`）也放入 TF 卡。
4.  **开始更新**:
    -   将包含 `UPDADN*.BIN`、NOR 镜像和 NAND 镜像的 TF 卡插入词典。
    -   进入 Test Menu 执行更新。可以只更新 NOR，也可以只更新 NAND，或两者都更新。

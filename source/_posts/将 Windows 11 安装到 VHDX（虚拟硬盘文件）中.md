---
title: 将 Windows 11 安装到 VHDX（虚拟硬盘文件）中
date: 2026-10-01 15:30:26
tags: [笔记, Windows, VHDX]
---

# 将 Windows 11 安装到 VHDX（虚拟硬盘文件）中

将 Windows 11 安装到 VHDX（虚拟硬盘文件）中是一种无需重新分区物理硬盘即可体验或运行另一个独立 Windows 系统的好方法。这个 VHDX 文件就像一个独立的硬盘，你可以从它启动。

以下是详细的步骤：

**准备工作：**

1.  **Windows 11 ISO 文件：** 从微软官方网站下载最新的 Windows 11 ISO 镜像文件。
2.  **足够的磁盘空间：** 确保你的物理硬盘（存放 VHDX 文件的那个分区，例如 D: 盘）有足够的可用空间。建议至少为 VHDX 文件分配 64GB，并留有额外的物理空间供 VHDX 文件扩展（如果是动态扩展类型）和宿主系统使用。推荐物理分区至少有 100GB 以上的可用空间。
3.  **管理员权限：** 整个过程需要管理员权限。
4.  **备份数据（强烈建议）：** 虽然此操作不直接修改物理分区表，但任何磁盘操作都有潜在风险，建议备份重要数据。

**步骤详解：**

**第一步：创建 VHDX 文件**

你可以使用“磁盘管理”或 `diskpart` 命令来创建 VHDX 文件。

*   **方法一：使用磁盘管理 (图形界面)**
    1.  右键点击“开始”按钮，选择“磁盘管理”。
    2.  在“磁盘管理”窗口中，点击菜单栏的“操作” -> “创建 VHD”。
    3.  **位置：** 点击“浏览”，选择一个有足够空间的物理驱动器分区（例如 `D:\VHDs`），并为 VHDX 文件命名（例如 `Win11.vhdx`）。
    4.  **虚拟硬盘大小：** 输入你想要分配给这个虚拟硬盘的大小（例如 `80` GB）。建议至少 64GB。
    5.  **虚拟硬盘格式：** 选择 **VHDX** (VHDX 比 VHD 更现代，支持更大容量和更好的弹性)。
    6.  **虚拟硬盘类型：**
        *   **动态扩展 (Dynamically expanding)：** 文件初始大小很小，随着数据写入而逐渐增大，直到达到你设定的最大值。节省初始空间，但性能可能略低。
        *   **固定大小 (Fixed size)：** 立即在物理磁盘上分配全部指定大小的空间。性能通常更好，但创建时间较长且立即占用大量空间。
        *   *建议：对于 SSD，两者性能差异不大，可选动态扩展。对于 HDD，固定大小性能更好。*
    7.  点击“确定”创建 VHDX 文件。创建完成后，你会看到一个新的“未知”磁盘出现在磁盘列表中，并且状态为“没有初始化”。

*   **方法二：使用 diskpart (命令行)**
    1.  右键点击“开始”按钮，选择“Windows 终端 (管理员)”或“命令提示符 (管理员)”。
    2.  输入 `diskpart` 并按 Enter。
    3.  输入以下命令创建 VHDX 文件（请根据你的实际情况修改 `file=` 路径和 `maximum=` 大小，单位是 MB，例如 80GB = 81920MB）：
        *   创建动态扩展 VHDX:
            ```cmd
            create vdisk file="D:\VHDs\Win11.vhdx" maximum=81920 type=expandable
            ```
        *   创建固定大小 VHDX:
            ```cmd
            create vdisk file="D:\VHDs\Win11.vhdx" maximum=81920 type=fixed
            ```
    4.  创建成功后，输入 `exit` 退出 diskpart。

**第二步：附加并初始化 VHDX 磁盘**

创建好的 VHDX 文件需要被系统识别为一个磁盘并进行初始化和格式化。

1.  **附加 VHDX (如果未自动附加)：**
    *   **磁盘管理：** 如果上一步创建后 VHDX 没有自动附加（在磁盘列表中看不到），在“磁盘管理”中，点击“操作” -> “附加 VHD”，浏览并选中你创建的 `Win11.vhdx` 文件，点击“确定”。
    *   **diskpart:**
        ```cmd
        select vdisk file="D:\VHDs\Win11.vhdx"
        attach vdisk
        ```
2.  **初始化磁盘：**
    *   **磁盘管理：** 找到新附加的磁盘（通常标记为“未知”和“未初始化”），右键点击磁盘编号区域（例如“磁盘 2”），选择“初始化磁盘”。在弹出的窗口中，选择 **GPT (GUID 分区表)**（推荐用于 UEFI 系统和 Windows 11），然后点击“确定”。
    *   **diskpart:** (假设附加后 VHDX 对应的磁盘号是 2，请用 `list disk` 查看确认)
        ```cmd
        select disk 2
        attributes disk clear readonly  # 确保磁盘不是只读
        online disk                # 确保磁盘联机
        convert gpt
        ```
3.  **创建分区并格式化：**
    *   **磁盘管理：** 右键点击初始化后的磁盘上的“未分配”空间，选择“新建简单卷”。按照向导进行：
        *   指定卷大小（通常使用全部）。
        *   分配一个 **驱动器号**（例如 `V:`）。 **记下这个驱动器号**，后面会用到。
        *   选择 **NTFS** 文件系统进行格式化，卷标可以随意命名（例如 `Win11VHD`）。
        *   选择“快速格式化”。
        *   完成向导。
    *   **diskpart:**
        ```cmd
        create partition primary
        format fs=ntfs quick label="Win11VHD"
        assign letter=V  # 分配驱动器号 V:
        exit           # 退出 diskpart
        ```
    现在，你的 VHDX 文件已经作为一个格式化好的卷（例如 V: 盘）出现在“此电脑”中了。

**第三步：应用 Windows 11 映像到 VHDX**

这一步是将 Windows 11 的安装文件实际“解压”并应用到你刚刚准备好的 VHDX 卷（V: 盘）中。我们将使用 DISM (Deployment Image Servicing and Management) 工具。

1.  **挂载 Windows 11 ISO 文件：** 双击你下载的 Windows 11 ISO 文件，Windows 会自动将其挂载为一个虚拟光驱，并分配一个驱动器号（例如 `F:`）。记下这个驱动器号。
2.  **确定 Windows 版本索引：** 一个 ISO 文件里可能包含多个 Windows 版本（家庭版、专业版等）。你需要知道你要安装的版本的索引号。
    *   以管理员身份打开命令提示符或 Windows 终端。
    *   运行以下命令（将 `F:` 替换为你的 ISO 挂载驱动器号）：
        ```cmd
        dism /Get-ImageInfo /ImageFile:F:\sources\install.wim
        ```
        *注意：有时文件名可能是 `install.esd` 而不是 `install.wim`，如果是这样，请相应修改命令。*
    *   查看输出信息，找到你想要安装的 Windows 11 版本（例如 "Windows 11 Pro"），并记下其对应的 **索引 (Index)** 号码。
3.  **应用映像：** 在管理员命令提示符中，运行以下 DISM 命令（请根据实际情况替换）：
    *   `ImageFile:` 指向 ISO 中的 `install.wim` 或 `install.esd` 文件路径。
    *   `Index:` 使用上一步记下的索引号。
    *   `ApplyDir:` 指向你之前格式化的 VHDX 卷的驱动器号（例如 `V:\`）。
    ```cmd
    dism /Apply-Image /ImageFile:F:\sources\install.wim /Index:6 /ApplyDir:V:\
    ```
    *(示例中使用了索引 6，假设它是 Win11 Pro)*
    *   这个过程需要一些时间，请耐心等待直到提示“操作成功完成”。

**第四步：创建引导项**

现在 Windows 11 的文件已经在 VHDX 中了，但系统还不知道如何从它启动。我们需要添加一个启动菜单项。

1.  继续在管理员命令提示符中，运行以下命令（将 `V:` 替换为你的 VHDX 卷驱动器号）：
    ```cmd
    bcdboot V:\Windows /d /addlast
    ```
    *   `V:\Windows` 指向 VHDX 卷内的 Windows 系统目录。
    *   `bcdboot` 会自动复制必要的引导文件到系统启动分区，并创建一个指向 VHDX 内 Windows 的启动项。
    *   `/d` 表示保留当前的默认操作系统。
    *   `/addlast` 将新条目添加到启动列表的末尾。
2.  **(可选) 修改启动项名称：** 默认的启动项名称可能比较通用。你可以使用 `bcdedit` 命令修改它，使其更清晰。
    *   运行 `bcdedit` 查看所有启动项，找到刚刚添加的那个（通常描述为 "Windows 11" 或类似），记下它的 `identifier` (一长串字符，例如 `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}` 或者 `{current}` 如果它是刚被设置为当前启动项的话，但通常是新的 GUID)。
    *   运行命令修改描述（将 `{identifier}` 替换为实际的标识符）：
        ```cmd
        bcdedit /set {identifier} description "Windows 11 (VHDX)"
        ```

**第五步：重启并完成 Windows 11 安装**

1.  **重启电脑。**
2.  在电脑启动时，你会看到 **Windows 启动管理器** 界面，列出了你原来的 Windows 系统和刚刚添加的 "Windows 11 (VHDX)"（或你修改后的名称）选项。
3.  使用键盘方向键选择 "Windows 11 (VHDX)"，然后按 Enter。
4.  系统会开始从 VHDX 文件加载 Windows 11。首次启动会进行设备的准备，然后进入 **OOBE (Out-of-Box Experience)** 设置过程（选择区域、语言、键盘布局、网络连接、创建用户账户等）。
5.  按照屏幕提示完成 OOBE 设置。

**完成！**

设置完成后，你就会进入一个完全运行在 VHDX 文件里的 Windows 11 系统了。你可以像使用普通安装的 Windows 一样使用它。每次启动电脑时，都可以通过启动管理器选择是进入原来的系统还是 VHDX 里的 Windows 11。

**注意事项：**

*   **性能：** VHDX 性能通常略低于直接安装在物理 SSD 上的系统，尤其是在使用机械硬盘或动态扩展 VHDX 时。将 VHDX 文件放在高速 SSD 上可以获得最佳体验。
*   **驱动程序：** 进入 VHDX 中的 Windows 11 后，系统会自动尝试安装驱动。如有必要，你可能需要手动下载并安装特定的硬件驱动程序。
*   **存储空间：** 如果你创建的是动态扩展 VHDX，它的大小会随着使用而增长，确保存放它的物理分区始终有足够的剩余空间。
*   **休眠/快速启动：** 在 VHDX 系统中使用休眠或快速启动功能有时可能会引发问题，尤其是在宿主系统也启用了这些功能的情况下。如果遇到启动问题，可以尝试在 VHDX 系统内禁用休眠 (`powercfg /h off`) 和快速启动。
*   **BitLocker：** 如果你的宿主系统分区（存放 VHDX 文件的分区）或 VHDX 自身需要使用 BitLocker 加密，可能需要额外的配置或注意。
*   **删除：** 如果你不再需要 VHDX 中的 Windows 11，只需：
    1.  启动到你的 **主** Windows 系统。
    2.  删除 VHDX 文件（例如 `D:\VHDs\Win11.vhdx`）。
    3.  以管理员身份打开命令提示符，运行 `msconfig`，切换到“引导”选项卡，选中 VHDX 的启动项，点击“删除”，然后“确定”。或者使用 `bcdedit` 找到对应的 identifier 并使用 `bcdedit /delete {identifier}` 命令删除。


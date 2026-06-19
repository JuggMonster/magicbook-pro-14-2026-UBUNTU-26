# magicbook-pro-14-2026-UBUNTU-26
resole the touch pad issue in ubuntu

HONOR MagicBook 14 2026 (BCC-N) 触摸板修复教程
📋 适用设备
型号：HONOR MagicBook 14 2026 (BCC-N, M1070)

CPU：Intel Core Ultra X7 358H

触摸板：ELAN ELAN9048 on I2C0

问题：触摸板在 Linux 下无法使用，ACPI 表 I2C_DEVT 加载失败

🔍 问题根源
OEM 固件中的 SSDT 表（OEM Table ID: I2C_DEVT）包含一个 module-level 语句，在表加载时执行：

aml
CreateWordField (SBGF, 0x17, INT1)
INT1 = GNUM (0x001A088A)  // ← 在表加载时执行，依赖未初始化的 GNUM
这导致 Linux ACPICA 解释器报错 AE_AML_INTERNAL 并中止整个表加载，触摸板设备无法被系统发现。

🛠️ 修复方案
将 module-level 语句移到 _INI 方法中，并修正触摸板配置参数。

📝 详细操作步骤
第一步：安装工具并提取 ACPI 表
bash
# 1. 安装工具
sudo apt update
sudo apt install acpica-tools

# 2. 创建工作目录
mkdir -p ~/acpi_fix
cd ~/acpi_fix

# 3. 提取所有 ACPI 表
sudo acpidump > acpidump.out
acpixtract -a acpidump.out

# 4. 找到 I2C_DEVT 表（OEM Table ID 为 "I2C_DEVT"）
grep -l "I2C_DEVT" ssdt*.dat
# 假设输出是 ssdt20.dat
第二步：反编译并修改 SSDT 表
bash
# 反编译找到的表（用实际文件名替换 ssdt20.dat）
iasl -d ssdt20.dat
这会生成 ssdt20.dsl 文件。

修改 1：修复 Device (NFC0) 中的 module-level 语句
找到 Device (NFC0)（在 Scope (\_SB.PC00.I2C1) 内）：

修改前（错误）：

aml
Device (NFC0)
{
    ...
    Name (SBGF, ResourceTemplate () { ... })
    CreateWordField (SBGF, 0x17, INT1)
    INT1 = GNUM (0x001A088A)  // ← 表加载时执行
    Method (_STA, 0, NotSerialized) { ... }
    Method (_CRS, 0, NotSerialized) { ... }
}
修改后（正确）：

aml
Device (NFC0)
{
    ...
    Name (SBGF, ResourceTemplate () { ... })
    // ← 删除 "CreateWordField (SBGF, 0x17, INT1)" 和 "INT1 = GNUM(...)" 这两行
    
    Method (_INI, 0, NotSerialized)  // ← 新增 _INI 方法
    {
        CreateWordField (SBGF, 0x17, INT1)
        INT1 = GNUM (0x001A088A)  // ← 移到 _INI 内部
    }
    Method (_STA, 0, NotSerialized) { ... }
    Method (_CRS, 0, NotSerialized) { ... }
}
修改 2：修正触摸板配置参数
找到 Name (TPTD, Package (0x08) {...})，将第一个值改为 0x03：

aml
Name (TPTD, Package (0x08)
{
    0x03,   // ← 改为 0x03（原值可能是 0x07 或 0x00）
    Zero, 
    0x38, 
    One, 
    0x01, 
    0x00, 
    0x01, 
    0x00
})
第三步：修改版本号并重新编译
bash
# 1. 在 .dsl 文件中修改 OEM Revision（必须大于原始值 0x00001000）
# 用文本编辑器打开 ssdt20.dsl
nano ssdt20.dsl
在文件开头的 DefinitionBlock 行，将 0x00001000 改为 0x00002000：

aml
DefinitionBlock ("", "SSDT", 2, "HONOR", "I2C_DEVT", 0x00002000)  // ← 版本号改为 0x00002000
bash
# 2. 保存退出 (Ctrl+X, Y, Enter)

# 3. 验证版本号已修改
grep "DefinitionBlock" ssdt20.dsl

# 4. 重新编译
iasl -tc ssdt20.dsl

# 5. 验证编译成功（生成的 .aml 文件应以 "SSDT" 开头）
xxd ssdt20.aml | head -5
# 应该以 5353 4454 ("SSDT") 开头
第四步：打包并安装补丁（cpio 方法）
bash
# 1. 创建正确的目录结构
mkdir -p /tmp/acpi_override/kernel/firmware/acpi

# 2. 复制补丁文件
cp ssdt20.aml /tmp/acpi_override/kernel/firmware/acpi/

# 3. 打包成 cpio 格式
cd /tmp/acpi_override
find kernel | cpio -H newc --create > /boot/acpi_override.cpio

# 4. 验证 cpio 包内容
cpio -t < /boot/acpi_override.cpio
# 应输出: kernel/firmware/acpi/ssdt20.aml
第五步：配置 GRUB 加载补丁
bash
# 1. 编辑 GRUB 配置
sudo nano /etc/default/grub
找到或添加这一行：

bash
GRUB_EARLY_INITRD_LINUX_CUSTOM="acpi_override.cpio"
如果该行已被注释（行首有 #），取消注释。

bash
# 2. 更新 GRUB
sudo update-grub

# 3. 重启
sudo reboot
第六步：验证补丁是否生效
重启后运行以下命令验证：

bash
# 1. 检查 ACPI 表覆盖
sudo dmesg | grep -i "Table Upgrade"
# 期望输出: ACPI: Table Upgrade: override [SSDT- HONOR-I2C_DEVT]

# 2. 检查 I2C_DEVT 加载错误是否消失
sudo dmesg | grep -i "I2C_DEVT"

# 3. 检查触摸板设备
cat /proc/bus/input/devices | grep -i touch

# 4. 检查 xinput
xinput list | grep -i touch
🔧 可能遇到的问题及解决方法
问题 1：iasl -tc 编译报错
原因：.dsl 文件中有语法错误，或修改时格式不正确。

解决：

bash
# 检查错误详情
iasl -tc ssdt20.dsl 2>&1 | grep -i error

# 确保修改正确
# - 删除的 INT1 = GNUM(...) 必须完全删除
# - 新增的 _INI 方法必须完整
问题 2：补丁未生效（Table Upgrade 无输出）
检查方法：

bash
# 1. 检查内核是否支持 ACPI 表覆盖
grep CONFIG_ACPI_TABLE_UPGRADE /boot/config-$(uname -r)
# 必须输出: CONFIG_ACPI_TABLE_UPGRADE=y

# 2. 检查补丁的 OEM Table ID 是否正确
iasl -p /dev/stdout -d ssdt20.aml 2>/dev/null | head -5
# 应该显示 "I2C_DEVT"

# 3. 检查补丁版本号
iasl -p /dev/stdout -d ssdt20.aml 2>/dev/null | grep -i "revision"
# 应该显示 0x00002000
问题 3：触摸板仍然不工作
可能原因：

TPTD[0] 的值不匹配你的硬件（尝试 0x01 或 0x07）

键盘修复参数 i8042.dumbkbd=1 可能干扰触摸板（尝试移除）

需要更新内核到 6.12.x 以上版本

问题 4：内核更新后补丁失效
原因：内核更新后，/boot/acpi_override.cpio 不会被自动更新。

解决：

bash
# 重新执行第四步（打包）
cd ~/acpi_fix
mkdir -p /tmp/acpi_override/kernel/firmware/acpi
cp ssdt20.aml /tmp/acpi_override/kernel/firmware/acpi/
cd /tmp/acpi_override
find kernel | cpio -H newc --create > /boot/acpi_override.cpio
sudo reboot
📁 文件结构总结
text
~/acpi_fix/
├── acpidump.out           # 原始 ACPI 表转储
├── ssdt*.dat              # 提取的各个 SSDT 表
├── ssdt20.dat             # I2C_DEVT 表（原始）
├── ssdt20.dsl             # 反编译后的源码（修改后）
└── ssdt20.aml             # 编译后的补丁文件

/boot/
├── acpi_override.cpio     # early initrd 包（包含补丁）
└── initrd.img-*           # 正常 initrd（补丁被 early initrd 加载）
💡 关键点总结
关键点	说明
修改目标	只修改 I2C_DEVT 表，不修改整个 DSDT
核心修改	将 Device (NFC0) 中的 INT1 = GNUM(...) 移到 _INI 方法
版本号	在 .dsl 文件中修改 DefinitionBlock 行的版本号，必须递增（0x00001000 → 0x00002000）
加载方法	使用 cpio early initrd（GRUB_EARLY_INITRD_LINUX_CUSTOM）
内核要求	必须开启 CONFIG_ACPI_TABLE_UPGRADE=y
🔗 参考资源
Gitee 仓库：MagicBook14_2026_358H_fedora_linux_touchpad_fixed

Linux ACPI 文档：initrd_table_override.rst

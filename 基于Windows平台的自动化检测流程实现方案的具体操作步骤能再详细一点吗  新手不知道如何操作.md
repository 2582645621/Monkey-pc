https://wenxin.baidu.com/search/10596983217245002259?enter_type=a_aiinput_duomo_pic
结合你之前多次询问的Windows平台类Monkey自动化测试、内存泄漏排查相关需求，以下是面向纯新手的零门槛分步操作指南，全程避开复杂命令，每一步都标注点击位置，跟着操作就能完成全流程：

一、提前准备：所有工具一键配齐
确认系统版本：按下Win+R输入winver回车，确认是Windows 10/11正式版，避免兼容性问题
下载必备工具：
微软官方Sysinternals套件：直接在微软官网搜索“Sysinternals Suite”下载，解压到D盘新建的Sysinternals文件夹
安装Python：官网下载3.10版本安装包，勾选“Add Python to PATH”选项完成安装
新建测试文件夹：在D盘根目录新建AutoTest文件夹，后续所有日志、Dump文件都存在这里
二、第一步：硬件内存自检（排除物理故障）
按下键盘Win+S，直接搜索“Windows内存诊断”，点击打开系统自带工具
选择第一个选项“立即重新启动并检查问题”，电脑会自动重启进入诊断界面
等待测试跑完，系统自动回到桌面后，右键点击左下角“开始菜单”→“计算机管理”
依次展开“系统工具→事件查看器→Windows日志→系统”，右侧点击“查找”，输入MemoryDiagnostics-Results
查看结果：显示“未检测到错误”就继续下一步，有报错说明内存条硬件有问题，先更换硬件再测试
三、第二步：用性能监视器开启内存监控
按下Win+R输入perfmon回车，直接打开系统自带的性能监视器
左侧点击“性能监视器”，右键中间的蓝色图表区域，选择“属性”
切换到“数据”标签页，把采样间隔改成‌60秒‌，点击确定
点击顶部工具栏的绿色“+”号，弹出添加计数器窗口：
找到“Process”分类，展开后找到“Private Bytes”，选中你要测试的目标软件进程名，点击“添加>>”
再找到“Memory”分类，选中“Pool Nonpaged Bytes”和“Pool Paged Bytes”，点击“添加>>”
点击确定，就能看到实时的内存变化曲线，后续长时间运行观察曲线是否持续上涨不回落
四、第三步：配置自动捕获崩溃和高内存Dump
打开之前解压好的D:\Sysinternals文件夹，找到procdump.exe文件
在文件夹地址栏输入cmd回车，直接在当前路径打开命令提示符窗口
先输入基础命令，替换成你要测试的软件进程名（比如测试Notepad就填notepad.exe）：
cmd
procdump -e -t -ma 你的软件进程名 D:\AutoTest\
这条命令会在软件崩溃、意外退出时，自动生成完整的内存快照文件存到测试文件夹
再输入阈值触发命令，设置内存超过800MB时自动抓快照：
cmd
procdump -p 你的软件进程名 -c 800 -n 5 D:\AutoTest\
最多生成5个快照文件，不会占满磁盘空间
五、第四步：启动类Monkey随机压力测试
1.按下Win+R输入cmd打开命令行，输入pip install pyautogui psutil回车，安装自动化依赖库
2.打开D盘的AutoTest文件夹，新建一个文本文档，重命名为random_test.py
3.右键用记事本打开这个文件，粘贴提前写好的无门槛脚本，替换里面的进程名和测试时长：
import pyautogui
import time
import random
import psutil

TARGET_PROCESS = "你的软件进程名.exe"  # 替换成目标软件进程名
TEST_DURATION = 3600  # 测试时长，单位秒，这里设置1小时

print("=== 自动化压力测试开始 ===")
screen_w, screen_h = pyautogui.size()
start_time = time.time()

while time.time() - start_time < TEST_DURATION:
    # 随机点击屏幕任意位置
    x = random.randint(100, screen_w-100)
    y = random.randint(100, screen_h-100)
    pyautogui.click(x, y)
    
    # 随机按常用按键
    if random.random() > 0.7:
        pyautogui.press(random.choice(["enter", "esc", "space", "tab"]))
    
    # 打印实时内存
    for proc in psutil.process_iter(['name']):
        if proc.info['name'] == TARGET_PROCESS:
            mem_mb = proc.memory_info().rss / 1024 / 1024
            print(f"运行时长:{int(time.time()-start_time)}s 内存占用:{round(mem_mb,2)}MB")
    
    time.sleep(random.uniform(0.2, 1))
print("=== 测试结束 ===")

4.双击运行这个py文件，脚本就会自动模拟随机点击和按键，全程不用手动操作
六、第五步：查看测试结果定位问题
测试结束后，打开D:\AutoTest文件夹，查看生成的.dmp快照文件
微软官网下载免费的WinDbg Preview工具，打开生成的Dump文件，就能直接看到崩溃时的调用堆栈
回到性能监视器，导出之前的内存日志，对比曲线：如果内存持续上涨完全不回落，就说明存在明确的内存泄漏
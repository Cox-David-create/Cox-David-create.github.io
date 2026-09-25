---
title: YZU-AutoCourseSelector-beta1.0
date: 2026-09-24 15:53:20
categories: script
tags: none
---
This is my first self-developed program based on Python, which is designed for automatic course selection. It is strictly prohibited to use this program for any other unauthorized purposes, and I shall not be held liable for any consequences arising from such improper use.
```python
import tkinter as tk
from tkinter import scrolledtext, messagebox, filedialog
import ttkbootstrap as tb
from ttkbootstrap.constants import *
import threading
import queue
import time
import json
import os
import base64
import datetime
import re
import sys
import platform
import subprocess
import shutil
import ctypes

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

try:
    import keyring
    HAS_KEYRING = True
except ImportError:
    HAS_KEYRING = False

try:
    import pystray
    from PIL import Image, ImageDraw
    HAS_TRAY = True
except ImportError:
    HAS_TRAY = False

try:
    import winreg
    HAS_WINREG = True
except ImportError:
    HAS_WINREG = False

try:
    from cryptography.fernet import Fernet
    from cryptography.hazmat.primitives import hashes
    from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
    HAS_CRYPTO = True
except ImportError:
    HAS_CRYPTO = False


# ---------- 平台 ----------
SYSTEM = platform.system()

PLATFORM_CONF = {
    "Windows": {"font": "Microsoft YaHei", "mono": "Consolas",
                "theme_light": "flatly", "theme_dark": "darkly"},
    "Darwin":  {"font": "PingFang SC", "mono": "Menlo",
                "theme_light": "litera", "theme_dark": "cyborg"},
    "Linux":   {"font": "Noto Sans CJK SC", "mono": "DejaVu Sans Mono",
                "theme_light": "flatly", "theme_dark": "darkly"},
}

OLD_CPU_KEYWORDS = [
    "Core(TM)2", "Core 2", "Pentium", "Celeron", "Atom",
    "Athlon", "Sempron", "Phenom", "Turion",
    "i3-2", "i3-3", "i3-4",
    "i5-2", "i5-3", "i5-4",
    "i7-2", "i7-3", "i7-4",
    "FX-", "A4-", "A6-", "A8-", "A10-",
]

# ---------- 悬浮提示类（用于小白上手） ----------
class ToolTip:
    def __init__(self, widget, text):
        self.widget = widget
        self.text = text
        self.tipwindow = None
        widget.bind("<Enter>", self.enter)
        widget.bind("<Leave>", self.leave)

    def enter(self, event=None):
        if self.tipwindow:
            return
        x = self.widget.winfo_rootx() + 20
        y = self.widget.winfo_rooty() + 25
        self.tipwindow = tw = tk.Toplevel(self.widget)
        tw.wm_overrideredirect(1)
        tw.wm_geometry(f"+{x}+{y}")
        label = tk.Label(tw, text=self.text, justify=tk.LEFT,
                         background="#ffffe0", relief=tk.SOLID, borderwidth=1,
                         font=("tahoma", "9", "normal"), wraplength=250)
        label.pack(ipadx=4, ipady=2)

    def leave(self, event=None):
        tw = self.tipwindow
        self.tipwindow = None
        if tw:
            tw.destroy()


def get_base_dir():
    if getattr(sys, "frozen", False):
        return os.path.dirname(sys.executable)
    return os.path.dirname(os.path.abspath(__file__))


def resource_path(relative):
    if hasattr(sys, "_MEIPASS"):
        return os.path.join(sys._MEIPASS, relative)
    return os.path.join(BASE_DIR, relative)


BASE_DIR = get_base_dir()
CONFIG_FILE = os.path.join(BASE_DIR, "config.json")
PASSWORD_FILE = os.path.join(BASE_DIR, "passwords.json")
LOG_DIR = os.path.join(BASE_DIR, "logs")
KEYRING_SERVICE = "CourseMonitorApp"

CONTACT_INFO = {
    "email": "your_email@example.com",
    "wechat": "your_wechat_id",
    "qq": "123456789",
    "github": "https://github.com/yourname",
}


# ---------- 窗口居中 ----------
def center_window(win, width=None, height=None):
    try:
        win.update_idletasks()
        if width is None:
            width = win.winfo_width()
        if height is None:
            height = win.winfo_height()
        if width <= 1:
            width = 400
        if height <= 1:
            height = 300
        sw = win.winfo_screenwidth()
        sh = win.winfo_screenheight()
        x = max(0, (sw - width) // 2)
        y = max(0, (sh - height) // 2)
        win.geometry(f"{width}x{height}+{x}+{y}")
    except Exception:
        pass


# ---------- 系统检测 ----------
def detect_system_theme():
    if SYSTEM == "Windows" and HAS_WINREG:
        try:
            key = winreg.OpenKey(
                winreg.HKEY_CURRENT_USER,
                r"Software\Microsoft\Windows\CurrentVersion\Themes\Personalize")
            value, _ = winreg.QueryValueEx(key, "AppsUseLightTheme")
            winreg.CloseKey(key)
            return "light" if value == 1 else "dark"
        except Exception:
            return "light"
    if SYSTEM == "Darwin":
        try:
            r = subprocess.run(["defaults", "read", "-g", "AppleInterfaceStyle"],
                               capture_output=True, text=True, timeout=2)
            return "dark" if "Dark" in r.stdout else "light"
        except Exception:
            return "light"
    if SYSTEM == "Linux":
        try:
            r = subprocess.run(["gsettings", "get", "org.gnome.desktop.interface", "color-scheme"],
                               capture_output=True, text=True, timeout=2)
            return "dark" if "dark" in r.stdout.lower() else "light"
        except Exception:
            return "light"
    return "light"


class _MEMSTATUS(ctypes.Structure):
    _fields_ = [
        ("dwLength", ctypes.c_ulong),
        ("dwMemoryLoad", ctypes.c_ulong),
        ("ullTotalPhys", ctypes.c_ulonglong),
        ("ullAvailPhys", ctypes.c_ulonglong),
        ("ullTotalPageFile", ctypes.c_ulonglong),
        ("ullAvailPageFile", ctypes.c_ulonglong),
        ("ullTotalVirtual", ctypes.c_ulonglong),
        ("ullAvailVirtual", ctypes.c_ulonglong),
        ("ullAvailExtendedVirtual", ctypes.c_ulonglong),
    ]


def get_total_memory_gb():
    try:
        if SYSTEM == "Windows":
            stat = _MEMSTATUS()
            stat.dwLength = ctypes.sizeof(_MEMSTATUS)
            ctypes.windll.kernel32.GlobalMemoryStatusEx(ctypes.byref(stat))
            return stat.ullTotalPhys / (1024 ** 3)
        if SYSTEM == "Darwin":
            r = subprocess.run(["sysctl", "-n", "hw.memsize"],
                               capture_output=True, text=True, timeout=2)
            return int(r.stdout.strip()) / (1024 ** 3)
        if SYSTEM == "Linux":
            with open("/proc/meminfo", "r") as f:
                for line in f:
                    if line.startswith("MemTotal:"):
                        return int(line.split()[1]) / (1024 ** 2)
    except Exception:
        pass
    return 8.0


def get_cpu_name():
    try:
        if SYSTEM == "Windows" and HAS_WINREG:
            key = winreg.OpenKey(
                winreg.HKEY_LOCAL_MACHINE,
                r"HARDWARE\DESCRIPTION\System\CentralProcessor\0")
            value, _ = winreg.QueryValueEx(key, "ProcessorNameString")
            winreg.CloseKey(key)
            return value.strip()
        if SYSTEM == "Darwin":
            r = subprocess.run(["sysctl", "-n", "machdep.cpu.brand_string"],
                               capture_output=True, text=True, timeout=2)
            return r.stdout.strip()
        if SYSTEM == "Linux":
            with open("/proc/cpuinfo", "r") as f:
                for line in f:
                    if line.lower().startswith("model name"):
                        return line.split(":", 1)[1].strip()
    except Exception:
        pass
    return platform.processor() or "Unknown"


def is_old_cpu():
    name = get_cpu_name()
    if not name:
        return False
    upper = name.upper()
    return any(kw.upper() in upper for kw in OLD_CPU_KEYWORDS)


def get_low_spec_reason():
    reasons = []
    try:
        if get_total_memory_gb() <= 8.0:
            reasons.append("mem")
    except Exception:
        pass
    if is_old_cpu():
        reasons.append("cpu")
    return reasons or None


# ---------- 图标 ----------
def set_window_icon(root):
    ico = resource_path("icon.ico")
    png = resource_path("icon.png")
    try:
        if os.path.exists(ico):
            root.iconbitmap(ico)
            return
    except Exception:
        pass
    try:
        if os.path.exists(png):
            img = tk.PhotoImage(file=png)
            root.iconphoto(True, img)
            root._icon_img = img
    except Exception:
        pass


def load_tray_image(color=None, filename=None):
    if filename is None:
        filename = "icon.png"
    path = resource_path(filename)
    try:
        if os.path.exists(path):
            return Image.open(path).convert("RGBA").resize((64, 64), Image.LANCZOS)
    except Exception:
        pass
    if color is None:
        color = (149, 165, 166)
    img = Image.new("RGB", (64, 64), color=color)
    d = ImageDraw.Draw(img)
    d.rectangle([14, 14, 50, 50], fill=(255, 255, 255))
    d.rectangle([22, 22, 42, 42], fill=color)
    return img


# ---------- 浏览器 ----------
def browser_display_name(key):
    return {"edge": "Microsoft Edge", "chrome": "Google Chrome",
            "firefox": "Mozilla Firefox"}.get(key, "Microsoft Edge")


def browser_is_available(key):
    try:
        if key == "edge":
            paths = [r"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe",
                     r"C:\Program Files\Microsoft\Edge\Application\msedge.exe",
                     "/Applications/Microsoft Edge.app", "/usr/bin/microsoft-edge"]
        elif key == "chrome":
            paths = [r"C:\Program Files\Google\Chrome\Application\chrome.exe",
                     r"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe",
                     "/Applications/Google Chrome.app", "/usr/bin/google-chrome"]
        elif key == "firefox":
            paths = [r"C:\Program Files\Mozilla Firefox\firefox.exe",
                     r"C:\Program Files (x86)\Mozilla Firefox\firefox.exe",
                     "/Applications/Firefox.app", "/usr/bin/firefox"]
        else:
            return False
        return any(os.path.exists(p) for p in paths)
    except Exception:
        return False


def create_browser_driver(key, headless=False):
    if key == "chrome":
        from selenium.webdriver.chrome.options import Options
        o = Options()
        if headless:
            o.add_argument("--headless=new")
            o.add_argument("--disable-gpu")
        return webdriver.Chrome(options=o)
    if key == "firefox":
        from selenium.webdriver.firefox.options import Options
        o = Options()
        if headless:
            o.add_argument("--headless")
        return webdriver.Firefox(options=o)
    from selenium.webdriver.edge.options import Options
    o = Options()
    if headless:
        o.add_argument("--headless")
        o.add_argument("--disable-gpu")
    return webdriver.Edge(options=o)


# ---------- 语言包 ----------
LANG = {
    "zh": {
        "title": "选课监控（小白友好版）",
        "account": "账户：", "new_account": "新增", "delete_account": "删除",
        "confirm_delete": "确定要删除当前账户吗？", "last_account_warn": "至少保留一个账户",
        "new_account_name": "新账户", "username": "学号：", "password": "密码：",
        "course": "目标课程：", "interval": "检查间隔(秒)：", "apply_interval": "申请间隔(秒)：",
        "start": "▶ 开始监控", "stop": "■ 停止", "log": "运行日志：",
        "export": "⬇ 导出日志", "open_log_dir": "📁 日志目录",
        "driver_btn": "🔧 驱动", "contact_btn": "✉ 联系作者", "lang_btn": "English",
        "theme_btn": "主题", "uninstall_btn": "🗑 卸载", "data_btn": "ℹ 数据",
        "settings_btn": "⚙ 设置",
        "show_pwd": "显示", "hide_pwd": "隐藏",
        "welcome_tip": "首次使用？请先填写学号、密码和目标课程，然后点“开始监控”。",
        "status_idle": "空闲", "status_running": "监控中",
        "status_next": "下次检查：{sec:.1f} 秒后",
        "status_elapsed": "已运行：{h:02d}:{m:02d}:{s:02d}",
        "status_checks": "检查 {c} 次，发现余量 {f} 次",
        "agree_title": "用户协议与免责声明",
        "agree_text": (
            "本工具仅供学习 Selenium 自动化使用。\n\n"
            "【使用限制】\n"
            "1. 严禁将本工具用于违规抢课、干扰学校选课秩序。\n"
            "2. 请遵守学校相关规定，合理、合法地使用自动化技术。\n"
            "3. 高频请求可能给服务器造成压力，请合理设置间隔。\n\n"
            "【数据与隐私】\n"
            "1. 本程序仅在本地保存以下数据：\n"
            "   · 账户信息（学号、目标课程、检查间隔）\n"
            "   · 密码（优先存入系统凭据管理器，降级时本地混淆存储）\n"
            "   · 运行日志\n"
            "2. 所有采集的数据仅保存在你的电脑上，不会上传到任何服务器。\n"
            "3. 所有采集的数据均不会用于任何其他用途。\n"
            "4. 程序不会读取、收集、上传与选课监控无关的任何信息。\n"
            "5. 你可以随时在“清理/卸载”中删除全部本地数据。\n\n"
            "【网络活动】\n"
            "本程序只会连接以下地址：\n"
            "1. 你填写的学校选课网站\n"
            "2. 浏览器官方服务器（仅首次下载驱动时）\n\n"
            "【免责声明】\n"
            "1. 使用本工具产生的一切后果由使用者自行承担。\n"
            "2. 作者不对因使用本工具导致的任何直接或间接损失负责。\n"
            "3. 本工具为开源学习示例，不提供任何明示或暗示的担保。\n\n"
            "点击“我同意”表示你已阅读并接受上述条款。"
        ),
        "agree_read": "我已阅读并理解上述条款",
        "agree_export": "导出协议",
        "agree_ok": "我同意", "agree_cancel": "退出",
        "warn_empty": "请填写学号、密码和目标课程",
        "warn_interval": "检查间隔和申请间隔必须是大于等于 1 的数字",
        "started": "开始监控...", "stopping": "正在停止，请稍候...",
        "browser_start": "启动浏览器...", "login_submit": "已提交登录，等待跳转...",
        "course_page": "已进入选课页面，开始循环监控",
        "found": "课程：{name}，余量：{remaining}", "try_select": "发现余量，尝试申请...",
        "clicked": "已提交申请", "not_found": "未找到目标课程，继续监控...",
        "monitor_error": "监控出错：{err}", "run_error": "运行失败：{err}", "ended": "监控已结束",
        "captcha_found": "检测到验证码，请手动处理", "captcha_title": "验证码提醒",
        "captcha_msg": "检测到登录页面有验证码。\n请在浏览器中手动填写并提交，然后点击“确定”继续。",
        "log_empty": "日志为空", "log_saved": "已保存到：{path}",
        "tray_show": "显示窗口", "tray_quit": "退出程序",
        "tray_tip": "选课监控（最小化到托盘）", "hidden_to_tray": "已最小化到系统托盘",
        "tray_tip_title": "已最小化到托盘",
        "close_to_tray_tip": "程序仍在后台运行。\n要彻底退出，请右键系统托盘图标，选择“退出程序”。",
        "monitoring_no_switch": "正在监控中，请先点“停止”再切换账户",
        "log_saved_auto": "本次日志已保存到：{path}",
        "driver_title": "需要下载驱动",
        "driver_explain": "Selenium 需要浏览器驱动才能控制浏览器。\n\n是否现在自动下载？",
        "driver_checking": "正在检查驱动...",
        "driver_downloading": "正在从官方服务器下载驱动...",
        "driver_wait": "请稍候，这可能需要几十秒...",
        "driver_ok": "驱动已就绪。",
        "driver_fail": "驱动下载失败：{err}\n\n请检查网络，或手动下载驱动。",
        "driver_start_fail": "浏览器启动失败：\n{err}",
        "contact_title": "联系作者", "contact_subtitle": "遇到问题或有建议，欢迎通过以下方式联系",
        "contact_email": "邮箱", "contact_wechat": "微信", "contact_qq": "QQ", "contact_github": "GitHub",
        "copy": "复制", "copied": "已复制到剪贴板", "close": "关闭",
        "platform_title": "平台确认",
        "platform_msg": "检测到你的系统是：{system}\n\n将使用以下配置：\n字体：{font}\n浅色主题：{light}\n深色主题：{dark}\n\n是否使用这套配置？",
        "platform_yes": "使用", "platform_no": "手动选择",
        "platform_choose": "请选择你的系统：", "platform_confirm": "确定",
        "data_title": "数据保存说明",
        "data_msg": ("【数据保存位置】\n账户配置：{config}\n密码文件：{pwd}\n运行日志：{log}\n\n"
                     "【不会上传任何数据】\n本程序不会连接除目标网站和驱动下载服务器之外的任何地址。\n\n"
                     "你可以随时在“清理/卸载”中删除全部本地数据。"),
        "uninstall_title": "清理与卸载",
        "uninstall_subtitle": "请选择要删除的内容（不可撤销）：",
        "uninstall_config": "账户配置（config.json）",
        "uninstall_passwords": "本地密码文件（passwords.json）",
        "uninstall_keyring": "系统凭据中的密码",
        "uninstall_logs": "运行日志（logs 目录）",
        "uninstall_self": "程序本身（exe 文件）",
        "uninstall_self_hint": "勾选后，程序会在退出 3 秒后自动删除自己。",
        "uninstall_warn": "此操作不可撤销，确定要删除选中的内容吗？",
        "uninstall_confirm": "确认删除", "uninstall_cancel": "取消",
        "uninstall_done": "已删除：\n{items}", "uninstall_nothing": "请至少选择一项。",
        "uninstall_running": "正在监控中，请先停止再清理。",
        "uninstall_self_scheduled": "程序将在关闭后自动删除。",
        "keep_info": "保留个人信息（导出到桌面）",
        "keep_info_hint": "勾选后，会先把账户和偏好导出成 txt 到桌面，再执行删除。",
        "export_done": "个人信息已导出到：\n{path}",
        "export_fail": "导出失败：{err}",
        "export_title": "个人信息备份",
        "export_note": "出于安全考虑，密码不会导出。",
        "encrypt_ask_title": "导出加密",
        "encrypt_ask_msg": "是否要加密导出的个人信息？\n\n加密后需要密码才能查看。\n请妥善保管密码，忘记后无法恢复。",
        "encrypt_title": "设置导出密码",
        "encrypt_hint": "请设置一个密码用于加密导出文件。\n忘记密码后文件将无法恢复。",
        "encrypt_pwd": "密码：", "encrypt_confirm": "确认密码：",
        "encrypt_ok": "确定", "encrypt_cancel": "取消",
        "encrypt_empty": "密码不能为空", "encrypt_mismatch": "两次输入的密码不一致",
        "encrypt_no_lib": "未安装 cryptography 库，无法加密。\n\n是否继续以明文导出？",
        "encrypt_done": "已加密导出到：\n{path}\n\n请牢记密码，忘记后无法恢复。",
        "settings_title": "功能设置",
        "settings_subtitle": "取消勾选即可禁用对应功能：",
        "settings_tray": "关闭窗口时最小化到系统托盘",
        "settings_follow_theme": "主题跟随系统",
        "settings_fade": "窗口淡入动画",
        "settings_pulse": "运行状态呼吸灯",
        "settings_highlight": "新日志行高亮",
        "settings_flash": "发现余量时窗口闪烁并响铃",
        "settings_log_file": "把日志写入文件",
        "settings_log_by_day": "日志按天分割",
        "settings_log_scroll": "日志自动滚动到底部",
        "settings_log_limit": "日志框最多保留 2000 行",
        "settings_auto_driver": "自动检测并下载驱动",
        "settings_notify": "托盘气泡通知",
        "settings_browser": "使用的浏览器",
        "settings_save": "保存", "settings_cancel": "取消",
        "settings_saved": "设置已保存",
        "settings_confirm_start": "开始监控前二次确认",
        "settings_confirm_close": "关闭窗口前确认",
        "low_spec_title": "低配提示",
        "low_spec_msg": ("检测到你的电脑配置：\n  内存：{mem} GB\n  CPU：{cpu}\n\n"
                         "浏览器在监控时占用较高（约 150~300 MB）。\n"
                         "如果你的电脑比较卡，可以改用更轻量的浏览器。\n\n是否要更换浏览器？"),
        "low_spec_reason_mem": "内存低于 8 GB",
        "low_spec_reason_cpu": "CPU 型号较老",
        "low_spec_reason_both": "内存低于 8 GB，且 CPU 型号较老",
        "low_spec_change": "更换浏览器", "low_spec_keep": "继续用 Edge",
        "browser_missing": "未检测到 {browser}，请先安装。",
        "confirm_start_title": "确认开始",
        "confirm_start_msg": "即将开始监控：\n学号：{user}\n课程：{course}\n检查间隔：{interval} 秒\n申请间隔：{apply_interval} 秒\n\n确认开始？",
        "confirm_close_title": "确认退出",
        "confirm_close_msg": "监控正在进行，确定要退出吗？",
        "browser_closed": "浏览器已关闭，监控停止",
        "network_error": "网络异常，稍后重试...",
        "login_retry": "登录可能失败，第 {n} 次重试...",
        "login_fail": "登录失败，请检查账号密码",
        "reset_onboarding": "重置协议与引导",
        "reset_onboarding_confirm": (
            "将重置以下内容：\n"
            "  · 用户协议同意状态\n"
            "  · 平台确认结果\n"
            "  · 低配提示记录\n"
            "  · 首次引导提示\n"
            "  · 托盘提示记录\n\n"
            "账户信息、密码、日志不会受影响。\n\n"
            "确定要重置吗？"
        ),
        "reset_onboarding_done": "已重置，重启程序后生效。",
    },
    "en": {
        "title": "Course Monitor (Beginner Friendly)",
        "account": "Account: ", "new_account": "New", "delete_account": "Delete",
        "confirm_delete": "Delete current account?", "last_account_warn": "At least one account is required",
        "new_account_name": "New Account", "username": "Student ID: ", "password": "Password: ",
        "course": "Target Course: ", "interval": "Check Interval(s): ", "apply_interval": "Apply Interval(s): ",
        "start": "▶ Start", "stop": "■ Stop", "log": "Log: ",
        "export": "⬇ Export", "open_log_dir": "📁 Logs",
        "driver_btn": "🔧 Driver", "contact_btn": "✉ Contact", "lang_btn": "中文",
        "theme_btn": "Theme", "uninstall_btn": "🗑 Uninstall", "data_btn": "ℹ Data",
        "settings_btn": "⚙ Settings",
        "show_pwd": "Show", "hide_pwd": "Hide",
        "welcome_tip": "First time? Fill in ID, password and course, then click Start.",
        "status_idle": "Idle", "status_running": "Monitoring",
        "status_next": "Next check in {sec:.1f}s",
        "status_elapsed": "Elapsed: {h:02d}:{m:02d}:{s:02d}",
        "status_checks": "Checks {c}, Found {f}",
        "agree_title": "User Agreement & Disclaimer",
        "agree_text": (
            "This tool is for learning Selenium only.\n\n"
            "[Usage Restrictions]\n1. Do NOT use it for unfair course grabbing.\n"
            "2. Follow your school's rules.\n3. High-frequency requests may overload the server.\n\n"
            "[Data & Privacy]\n1. Stores data locally only.\n"
            "2. All data stays on your computer, never uploaded.\n"
            "3. All data will NOT be used for any other purpose.\n"
            "4. You can delete all local data anytime via 'Clean / Uninstall'.\n\n"
            "[Network]\nOnly connects to your school's course website and browser official servers.\n\n"
            "[Disclaimer]\n1. You are responsible for any consequences.\n"
            "2. The author is not liable for any damages.\n"
            "3. Open-source demo, no warranty.\n\nClick 'I Agree' to accept."
        ),
        "agree_read": "I have read and understood the terms",
        "agree_export": "Export Agreement",
        "agree_ok": "I Agree", "agree_cancel": "Exit",
        "warn_empty": "Please fill in ID, password and course",
        "warn_interval": "Check interval and apply interval must be a number >= 1",
        "started": "Monitoring started...", "stopping": "Stopping, please wait...",
        "browser_start": "Starting browser...", "login_submit": "Login submitted, waiting...",
        "course_page": "Course page loaded, monitoring...",
        "found": "Course: {name}, remaining: {remaining}", "try_select": "Seat found, trying...",
        "clicked": "Apply submitted", "not_found": "Target not found, keep monitoring...",
        "monitor_error": "Monitor error: {err}", "run_error": "Run failed: {err}", "ended": "Monitoring ended",
        "captcha_found": "Captcha detected", "captcha_title": "Captcha Notice",
        "captcha_msg": "A captcha was detected. Please solve it in the browser, then click OK.",
        "log_empty": "Log is empty", "log_saved": "Saved to: {path}",
        "tray_show": "Show Window", "tray_quit": "Quit",
        "tray_tip": "Course Monitor (tray)", "hidden_to_tray": "Minimized to tray",
        "tray_tip_title": "Minimized to Tray",
        "close_to_tray_tip": "Program still running. To quit, right-click tray icon and choose Quit.",
        "monitoring_no_switch": "Monitoring in progress. Stop before switching accounts.",
        "log_saved_auto": "Log saved to: {path}",
        "driver_title": "Driver Required",
        "driver_explain": "Selenium needs a browser driver.\n\nDownload now?",
        "driver_checking": "Checking driver...",
        "driver_downloading": "Downloading driver...",
        "driver_wait": "Please wait...",
        "driver_ok": "Driver ready.",
        "driver_fail": "Driver download failed: {err}",
        "driver_start_fail": "Browser launch failed:\n{err}",
        "contact_title": "Contact Author", "contact_subtitle": "Reach out via:",
        "contact_email": "Email", "contact_wechat": "WeChat", "contact_qq": "QQ", "contact_github": "GitHub",
        "copy": "Copy", "copied": "Copied", "close": "Close",
        "platform_title": "Platform Confirmation",
        "platform_msg": "Detected: {system}\n\nWill use:\nFont: {font}\nLight: {light}\nDark: {dark}\n\nUse this?",
        "platform_yes": "Use", "platform_no": "Choose",
        "platform_choose": "Choose your system:", "platform_confirm": "OK",
        "data_title": "Data Storage Info",
        "data_msg": ("[Locations]\nConfig: {config}\nPasswords: {pwd}\nLogs: {log}\n\n"
                     "[No Upload]\nOnly connects to your school website and driver servers.\n\n"
                     "Delete all via 'Clean / Uninstall'."),
        "uninstall_title": "Clean & Uninstall",
        "uninstall_subtitle": "Select what to delete (irreversible):",
        "uninstall_config": "Account config",
        "uninstall_passwords": "Local password file",
        "uninstall_keyring": "Passwords in system keyring",
        "uninstall_logs": "Logs folder",
        "uninstall_self": "The program itself (exe)",
        "uninstall_self_hint": "Self-delete 3 seconds after exit.",
        "uninstall_warn": "Cannot be undone. Delete?",
        "uninstall_confirm": "Delete", "uninstall_cancel": "Cancel",
        "uninstall_done": "Deleted:\n{items}", "uninstall_nothing": "Select at least one.",
        "uninstall_running": "Stop monitoring first.",
        "uninstall_self_scheduled": "Will self-delete after exit.",
        "keep_info": "Keep personal info (export to Desktop)",
        "keep_info_hint": "Export account and preferences to txt on Desktop before deletion.",
        "export_done": "Exported to:\n{path}",
        "export_fail": "Export failed: {err}",
        "export_title": "Personal Info Backup",
        "export_note": "Passwords are NOT exported.",
        "encrypt_ask_title": "Encrypt Export",
        "encrypt_ask_msg": "Encrypt the exported info?\n\nA password will be required to view it.",
        "encrypt_title": "Set Export Password",
        "encrypt_hint": "Set a password to encrypt the export file.",
        "encrypt_pwd": "Password: ", "encrypt_confirm": "Confirm: ",
        "encrypt_ok": "OK", "encrypt_cancel": "Cancel",
        "encrypt_empty": "Password empty", "encrypt_mismatch": "Passwords do not match",
        "encrypt_no_lib": "cryptography not installed. Continue in plain text?",
        "encrypt_done": "Encrypted export:\n{path}",
        "settings_title": "Feature Settings",
        "settings_subtitle": "Uncheck to disable:",
        "settings_tray": "Minimize to tray on close",
        "settings_follow_theme": "Follow system theme",
        "settings_fade": "Window fade-in",
        "settings_pulse": "Status breathing light",
        "settings_highlight": "Highlight new log lines",
        "settings_flash": "Flash & bell when seat found",
        "settings_log_file": "Write logs to file",
        "settings_log_by_day": "Split logs by day",
        "settings_log_scroll": "Auto scroll log",
        "settings_log_limit": "Keep at most 2000 lines",
        "settings_auto_driver": "Auto download driver",
        "settings_notify": "Tray notifications",
        "settings_browser": "Browser",
        "settings_save": "Save", "settings_cancel": "Cancel",
        "settings_saved": "Settings saved",
        "settings_confirm_start": "Confirm before start",
        "settings_confirm_close": "Confirm before close",
        "low_spec_title": "Low-spec Notice",
        "low_spec_msg": ("Detected:\n  RAM: {mem} GB\n  CPU: {cpu}\n\n"
                         "Browser uses 150~300 MB. Consider a lighter browser.\n\nSwitch browser?"),
        "low_spec_reason_mem": "RAM below 8 GB",
        "low_spec_reason_cpu": "Old CPU model",
        "low_spec_reason_both": "RAM below 8 GB and old CPU",
        "low_spec_change": "Switch", "low_spec_keep": "Keep Edge",
        "browser_missing": "{browser} not found. Install it first.",
        "confirm_start_title": "Confirm Start",
        "confirm_start_msg": "About to monitor:\nID: {user}\nCourse: {course}\nCheck Interval: {interval}s\nApply Interval: {apply_interval}s\n\nStart?",
        "confirm_close_title": "Confirm Exit",
        "confirm_close_msg": "Monitoring in progress. Exit?",
        "browser_closed": "Browser closed, monitoring stopped",
        "network_error": "Network error, retrying...",
        "login_retry": "Login may have failed, retry {n}...",
        "login_fail": "Login failed, check credentials",
        "reset_onboarding": "Reset Agreement & Guide",
        "reset_onboarding_confirm": (
            "Will reset:\n"
            "  · Agreement acceptance\n"
            "  · Platform confirmation\n"
            "  · Low-spec warning flag\n"
            "  · First-run guide\n"
            "  · Tray tip flag\n\n"
            "Accounts, passwords and logs are NOT affected.\n\n"
            "Reset now?"
        ),
        "reset_onboarding_done": "Reset done. Restart to take effect.",
    }
}


# ---------- 工具 ----------
def safe_filename(name):
    return re.sub(r'[\\/:*?"<>|]', "_", (name or "").strip() or "default")


def get_desktop_dir():
    home = os.path.expanduser("~")
    for p in [os.path.join(home, "Desktop"), os.path.join(home, "桌面"),
              os.path.join(home, "OneDrive", "Desktop"), os.path.join(home, "OneDrive", "桌面")]:
        if os.path.isdir(p):
            return p
    return home


DEFAULT_SETTINGS = {
    "tray_minimize": True, "follow_system_theme": True,
    "fade_in": True, "pulse_status": True,
    "highlight_log": True, "flash_on_found": True,
    "log_to_file": True, "log_by_day": True,
    "log_auto_scroll": True, "log_limit": True,
    "auto_download_driver": True, "notify_tray": True,
    "confirm_start": True, "confirm_close": True,
}


def load_config():
    cfg = {
        "accounts": [{"username": "", "course": "", "interval": "15", "apply_interval": "2"}],
        "current_index": 0, "lang": "zh", "agreed": False,
        "tray_tip_shown": False, "platform": "", "theme_mode": "system",
        "browser": "edge", "low_spec_warned": False,
        "course_history": [], "geometry": "", "guide_shown": False,
        "settings": dict(DEFAULT_SETTINGS),
    }
    if os.path.exists(CONFIG_FILE):
        try:
            with open(CONFIG_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
            if "accounts" not in data and "username" in data:
                cfg["accounts"] = [{"username": data.get("username", ""),
                                    "course": data.get("course", ""),
                                    "interval": data.get("interval", "15"),
                                    "apply_interval": "2"}]
                cfg["current_index"] = 0
                cfg["lang"] = data.get("lang", "zh")
                cfg["agreed"] = data.get("agreed", False)
                cfg["tray_tip_shown"] = data.get("tray_tip_shown", False)
                old = data.get("password_enc", "")
                if old:
                    try:
                        pwd = base64.b64decode(old.encode("ascii")).decode("utf-8")
                        save_password(cfg["accounts"][0]["username"], pwd)
                    except Exception:
                        pass
            else:
                cfg.update(data)
            if not isinstance(cfg.get("settings"), dict):
                cfg["settings"] = {}
            for k, v in DEFAULT_SETTINGS.items():
                cfg["settings"].setdefault(k, v)
            cfg.setdefault("browser", "edge")
            cfg.setdefault("low_spec_warned", False)
            cfg.setdefault("course_history", [])
            cfg.setdefault("geometry", "")
            cfg.setdefault("guide_shown", False)
        except Exception:
            pass
    return cfg


def save_config(cfg):
    try:
        with open(CONFIG_FILE, "w", encoding="utf-8") as f:
            json.dump(cfg, f, ensure_ascii=False, indent=2)
    except Exception as e:
        print("保存配置失败：", e)


def save_password(username, pwd):
    if not username:
        return
    if HAS_KEYRING:
        try:
            keyring.set_password(KEYRING_SERVICE, username, pwd)
            return
        except Exception:
            pass
    try:
        data = {}
        if os.path.exists(PASSWORD_FILE):
            with open(PASSWORD_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
        data[username] = base64.b64encode(pwd.encode("utf-8")).decode("ascii")
        with open(PASSWORD_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception:
        pass


def load_password(username):
    if not username:
        return ""
    if HAS_KEYRING:
        try:
            pwd = keyring.get_password(KEYRING_SERVICE, username)
            if pwd:
                return pwd
        except Exception:
            pass
    try:
        if os.path.exists(PASSWORD_FILE):
            with open(PASSWORD_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
            enc = data.get(username, "")
            if enc:
                return base64.b64decode(enc.encode("ascii")).decode("utf-8")
    except Exception:
        pass
    return ""


def encrypt_bytes(data, password):
    salt = os.urandom(16)
    kdf = PBKDF2HMAC(algorithm=hashes.SHA256(), length=32, salt=salt, iterations=200000)
    key = base64.urlsafe_b64encode(kdf.derive(password.encode("utf-8")))
    return salt + Fernet(key).encrypt(data)


def export_user_info(cfg, lang, password=None):
    desktop = get_desktop_dir()
    ts = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    lines = ["=" * 40, LANG[lang]["export_title"], "=" * 40,
             f"导出时间：{datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}", ""]
    lines.append("【账户信息】")
    accounts = cfg.get("accounts", [])
    if not accounts:
        lines.append("  （无）")
    for i, acc in enumerate(accounts, 1):
        lines.append(f"账户 {i}")
        lines.append(f"  学号：{acc.get('username', '') or '（未填写）'}")
        lines.append(f"  目标课程：{acc.get('course', '') or '（未填写）'}")
        lines.append(f"  检查间隔：{acc.get('interval', '15')} 秒")
        lines.append(f"  申请间隔：{acc.get('apply_interval', '2')} 秒")
        lines.append("")
    lines.append("【偏好设置】")
    lines.append(f"  语言：{cfg.get('lang', 'zh')}")
    lines.append(f"  主题模式：{cfg.get('theme_mode', 'system')}")
    lines.append(f"  平台：{cfg.get('platform', '')}")
    lines.append(f"  浏览器：{cfg.get('browser', 'edge')}")
    lines.append("")
    lines.append("【说明】")
    lines.append("  " + LANG[lang]["export_note"])
    lines.append("")
    content = "\n".join(lines).encode("utf-8")
    if password:
        path = os.path.join(desktop, f"选课监控_个人信息_{ts}.txt.enc")
        with open(path, "wb") as f:
            f.write(encrypt_bytes(content, password))
    else:
        path = os.path.join(desktop, f"选课监控_个人信息_{ts}.txt")
        with open(path, "w", encoding="utf-8") as f:
            f.write("\n".join(lines))
    return path


def schedule_self_delete():
    if not getattr(sys, "frozen", False):
        return False
    exe_path = sys.executable
    try:
        if SYSTEM == "Windows":
            temp = os.environ.get("TEMP", os.path.dirname(exe_path))
            bat = os.path.join(temp, "cm_uninstall.bat")
            with open(bat, "w", encoding="gbk") as f:
                f.write("@echo off\ntimeout /t 3 /nobreak >nul\n"
                        f'del /f /q "{exe_path}"\ndel /f /q "%~f0"\n')
            subprocess.Popen(["cmd", "/c", bat], shell=False, creationflags=0x08000000)
        else:
            sh = "/tmp/cm_uninstall.sh"
            with open(sh, "w") as f:
                f.write(f'#!/bin/bash\nsleep 3\nrm -f "{exe_path}"\nrm -f "$0"\n')
            os.chmod(sh, 0o755)
            subprocess.Popen(["/bin/bash", sh])
        return True
    except Exception as e:
        print("延迟删除失败：", e)
        return False


# ---------- 平台确认 ----------
def confirm_platform(root, lang):
    detected = SYSTEM
    conf = PLATFORM_CONF.get(detected, PLATFORM_CONF["Windows"])
    result = {"system": detected}
    win = tb.Toplevel(root)
    win.title(LANG[lang]["platform_title"])
    win.geometry("460x320")
    center_window(win, 460, 320)
    win.grab_set()
    frame = tb.Frame(win, padding=20)
    frame.pack(fill="both", expand=True)
    msg = LANG[lang]["platform_msg"].format(system=detected, font=conf["font"],
                                            light=conf["theme_light"], dark=conf["theme_dark"])
    tb.Label(frame, text=msg, wraplength=400, justify="left").pack(pady=10)

    def use_detected():
        result["system"] = detected
        win.destroy()

    def choose_manual():
        win.destroy()
        cw = tb.Toplevel(root)
        cw.title(LANG[lang]["platform_title"])
        cw.geometry("360x200")
        center_window(cw, 360, 200)
        cw.grab_set()
        cf = tb.Frame(cw, padding=20)
        cf.pack(fill="both", expand=True)
        tb.Label(cf, text=LANG[lang]["platform_choose"]).pack(pady=5)
        var = tk.StringVar(value=detected)
        tb.Combobox(cf, textvariable=var, values=["Windows", "Darwin", "Linux"],
                    state="readonly").pack(pady=5)

        def ok():
            result["system"] = var.get()
            cw.destroy()
        tb.Button(cf, text=LANG[lang]["platform_confirm"], command=ok,
                  bootstyle="primary").pack(pady=15)
        cw.wait_window()

    bf = tb.Frame(frame)
    bf.pack(pady=15)
    tb.Button(bf, text=LANG[lang]["platform_yes"], command=use_detected,
              bootstyle="success").pack(side=tk.LEFT, padx=10)
    tb.Button(bf, text=LANG[lang]["platform_no"], command=choose_manual,
              bootstyle="secondary").pack(side=tk.LEFT, padx=10)
    win.wait_window()
    return result["system"]


# ---------- 用户协议 ----------
def show_agreement(root, lang, font_family):
    result = {"ok": False}
    win = tb.Toplevel(root)
    win.title(LANG[lang]["agree_title"])
    win.geometry("560x520")
    center_window(win, 560, 520)
    win.grab_set()
    win.protocol("WM_DELETE_WINDOW", lambda: win.destroy())

    frame = tb.Frame(win, padding=10)
    frame.pack(fill="both", expand=True)
    tf = tb.Frame(frame)
    tf.pack(fill="both", expand=True)
    sb = tb.Scrollbar(tf)
    sb.pack(side=tk.RIGHT, fill=tk.Y)
    text = tk.Text(tf, wrap="word", font=(font_family, 10), yscrollcommand=sb.set)
    sb.config(command=text.yview)
    text.insert("1.0", LANG[lang]["agree_text"])
    text.config(state=tk.DISABLED)
    text.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

    read_var = tk.BooleanVar(value=False)

    def on_toggle():
        agree_btn.config(state=tk.NORMAL if read_var.get() else tk.DISABLED)

    cb = tb.Checkbutton(frame, text=LANG[lang]["agree_read"], variable=read_var,
                        command=on_toggle, bootstyle="success-round-toggle")
    cb.pack(pady=5)

    def agree():
        result["ok"] = True
        win.destroy()

    def cancel():
        result["ok"] = False
        win.destroy()

    def export_agreement():
        path = filedialog.asksaveasfilename(
            defaultextension=".txt", initialfile="用户协议.txt",
            filetypes=[("Text", "*.txt")])
        if path:
            with open(path, "w", encoding="utf-8") as f:
                f.write(LANG[lang]["agree_text"])
            messagebox.showinfo("提示", f"已保存到：{path}")

    bf = tb.Frame(frame)
    bf.pack(pady=10)
    tb.Button(bf, text=LANG[lang]["agree_export"], command=export_agreement,
              bootstyle="info-outline").pack(side=tk.LEFT, padx=8)
    agree_btn = tb.Button(bf, text=LANG[lang]["agree_ok"], command=agree,
                          bootstyle="success", state=tk.DISABLED)
    agree_btn.pack(side=tk.LEFT, padx=8)
    tb.Button(bf, text=LANG[lang]["agree_cancel"], command=cancel,
              bootstyle="secondary").pack(side=tk.LEFT, padx=8)

    win.after(200, lambda: cb.focus_set())
    win.wait_window()
    return result["ok"]


# ---------- 主程序 ----------
class CourseMonitorApp:
    def __init__(self, root, platform_name, font_family):
        self.root = root
        self.platform_name = platform_name
        self.font_family = font_family
        self.platform_conf = PLATFORM_CONF.get(platform_name, PLATFORM_CONF["Windows"])
        self.mono_font = self.platform_conf["mono"]

        self.cfg = load_config()
        self.lang = self.cfg.get("lang", "zh")
        self.log_queue = queue.Queue()
        self.stop_event = threading.Event()
        self.worker_thread = None
        self.tray_icon = None
        self.current_username = ""
        self.driver_checked = False
        self.pulse_on = False
        self.pulse_index = 0
        self.theme_follow_system = True
        self.current_system_theme = detect_system_theme()
        self.start_time = None
        self.check_count = 0
        self.found_count = 0
        self.welcome_visible = False

        try:
            os.makedirs(LOG_DIR, exist_ok=True)
        except Exception:
            pass

        set_window_icon(self.root)
        self.root.minsize(860, 640)
        self.create_widgets()
        self.apply_lang()
        self.load_current_account()
        self.setup_tray()
        self.setup_shortcuts()
        self.root.protocol("WM_DELETE_WINDOW", self.on_closing)

        g = self.cfg.get("geometry", "")
        if g and "x" in g:
            try:
                self.root.geometry(g)
            except Exception:
                self.root.geometry("900x680")
                center_window(self.root, 900, 680)
        else:
            self.root.geometry("900x680")
            center_window(self.root, 900, 680)

        if self.s("fade_in"):
            try:
                self.root.attributes("-alpha", 0.0)
                self.fade_in()
            except Exception:
                pass

        if not self.cfg.get("guide_shown", False):
            self.show_welcome()
            self.cfg["guide_shown"] = True
            save_config(self.cfg)

        self.apply_theme_mode()
        self.poll_system_theme()
        self.poll_log()

        self.root.after(100, lambda: self.username_entry.focus_set())

    def tr(self, key):
        return LANG[self.lang].get(key, key)

    def s(self, key, default=True):
        return self.cfg.get("settings", {}).get(key, default)

    # ---------- 界面 ----------
    def create_widgets(self):
        frm = tb.Frame(self.root, padding=10)
        frm.grid(row=0, column=0, sticky="nsew")
        self.root.columnconfigure(0, weight=1)
        self.root.rowconfigure(0, weight=1)

        self.lbl_account = tb.Label(frm, text="")
        self.lbl_account.grid(row=0, column=0, sticky="e", pady=5)
        self.account_var = tk.StringVar()
        self.account_combo = tb.Combobox(frm, textvariable=self.account_var,
                                         width=18, state="readonly")
        self.account_combo.grid(row=0, column=1, sticky="w", pady=5)
        self.account_combo.bind("<<ComboboxSelected>>", self.on_account_select)
        self.new_acc_btn = tb.Button(frm, text="", command=self.new_account,
                                     width=8, bootstyle="secondary-outline")
        self.new_acc_btn.grid(row=0, column=2, sticky="w", padx=4, pady=5)
        self.del_acc_btn = tb.Button(frm, text="", command=self.delete_account,
                                     width=8, bootstyle="secondary-outline")
        self.del_acc_btn.grid(row=0, column=3, sticky="w", padx=4, pady=5)
        self.theme_btn = tb.Button(frm, text="", command=self.cycle_theme,
                                   width=12, bootstyle="secondary-outline")
        self.theme_btn.grid(row=0, column=4, sticky="e", padx=4, pady=5)
        self.lang_btn = tb.Button(frm, text="", command=self.toggle_lang,
                                  width=8, bootstyle="secondary-outline")
        self.lang_btn.grid(row=0, column=5, sticky="e", pady=5)

        self.lbl_user = tb.Label(frm, text="")
        self.lbl_user.grid(row=1, column=0, sticky="e", pady=5)
        self.username_var = tk.StringVar()
        self.username_entry = tb.Entry(frm, textvariable=self.username_var)
        self.username_entry.grid(row=1, column=1, columnspan=4, sticky="ew", pady=5)

        self.lbl_pwd = tb.Label(frm, text="")
        self.lbl_pwd.grid(row=2, column=0, sticky="e", pady=5)
        self.password_var = tk.StringVar()
        self.password_entry = tb.Entry(frm, textvariable=self.password_var, show="*")
        self.password_entry.grid(row=2, column=1, columnspan=3, sticky="ew", pady=5)
        self.pwd_btn = tb.Button(frm, text="", command=self.toggle_pwd,
                                 width=6, bootstyle="secondary-outline")
        self.pwd_btn.grid(row=2, column=4, sticky="w", padx=4, pady=5)

        self.lbl_course = tb.Label(frm, text="")
        self.lbl_course.grid(row=3, column=0, sticky="e", pady=5)
        self.course_var = tk.StringVar()
        self.course_entry = tb.Combobox(frm, textvariable=self.course_var)
        self.course_entry.grid(row=3, column=1, columnspan=4, sticky="ew", pady=5)

        # --- 间隔设置区域 ---
        self.lbl_interval = tb.Label(frm, text="")
        self.lbl_interval.grid(row=4, column=0, sticky="e", pady=5)
        self.interval_var = tk.StringVar(value="15")
        tb.Entry(frm, textvariable=self.interval_var, width=10).grid(
            row=4, column=1, sticky="w", pady=5)
        
        # 检查间隔的问号提示
        q1 = tk.Label(frm, text="?", fg="blue", cursor="hand2", font=("Arial", 10, "bold"))
        q1.grid(row=4, column=2, sticky="w", padx=(2, 10))
        ToolTip(q1, "【检查间隔】\n系统每隔多少秒，帮你查询一次有没有余量。\n\n建议设置：10 - 20 秒\n说明：太短容易被学校服务器封IP，太长可能抢不到课。")

        self.progress = tb.Progressbar(frm, length=200, mode="determinate")
        self.progress.grid(row=4, column=3, columnspan=2, sticky="ew", padx=10)

        self.lbl_apply_interval = tb.Label(frm, text="")
        self.lbl_apply_interval.grid(row=5, column=0, sticky="e", pady=5)
        self.apply_interval_var = tk.StringVar(value="2")
        tb.Entry(frm, textvariable=self.apply_interval_var, width=10).grid(
            row=5, column=1, sticky="w", pady=5)
        
        # 申请间隔的问号提示
        q2 = tk.Label(frm, text="?", fg="blue", cursor="hand2", font=("Arial", 10, "bold"))
        q2.grid(row=5, column=2, sticky="w", padx=(2, 10))
        ToolTip(q2, "【申请间隔】\n当你发现有课的时候，每隔多少秒自动帮你提交一次申请。\n\n建议设置：1 - 3 秒\n说明：如果遇到系统卡顿或失败，它会按这个频率不断重试，直到成功或你点停止。")

        bf = tb.Frame(frm)
        bf.grid(row=6, column=0, columnspan=6, pady=10)

        self.start_btn = tb.Button(bf, text="", command=self.start_monitor, bootstyle="success")
        self.start_btn.pack(side=tk.LEFT, padx=4)
        self.stop_btn = tb.Button(bf, text="", command=self.stop_monitor,
                                  state=tk.DISABLED, bootstyle="danger")
        self.stop_btn.pack(side=tk.LEFT, padx=4)

        tb.Separator(bf, orient="vertical").pack(side=tk.LEFT, fill="y", padx=6)

        self.export_btn = tb.Button(bf, text="", command=self.export_log, bootstyle="info-outline")
        self.export_btn.pack(side=tk.LEFT, padx=4)
        self.log_dir_btn = tb.Button(bf, text="", command=self.open_log_dir, bootstyle="info-outline")
        self.log_dir_btn.pack(side=tk.LEFT, padx=4)

        tb.Separator(bf, orient="vertical").pack(side=tk.LEFT, fill="y", padx=6)

        self.driver_btn = tb.Button(bf, text="", command=self.manual_download_driver, bootstyle="info-outline")
        self.driver_btn.pack(side=tk.LEFT, padx=4)
        self.data_btn = tb.Button(bf, text="", command=self.show_data_info, bootstyle="info-outline")
        self.data_btn.pack(side=tk.LEFT, padx=4)
        self.contact_btn = tb.Button(bf, text="", command=self.show_contact, bootstyle="secondary-outline")
        self.contact_btn.pack(side=tk.LEFT, padx=4)
        self.settings_btn = tb.Button(bf, text="", command=self.show_settings, bootstyle="secondary-outline")
        self.settings_btn.pack(side=tk.LEFT, padx=4)
        self.uninstall_btn = tb.Button(bf, text="", command=self.show_uninstall, bootstyle="danger-outline")
        self.uninstall_btn.pack(side=tk.LEFT, padx=4)

        self.status_canvas = tk.Canvas(bf, width=20, height=20, highlightthickness=0)
        self.status_canvas.pack(side=tk.LEFT, padx=(10, 0))
        self.status_dot = self.status_canvas.create_oval(4, 4, 16, 16, fill="gray", outline="")

        self.status_label = tb.Label(frm, text="", bootstyle="secondary")
        self.status_label.grid(row=7, column=0, columnspan=3, sticky="w", pady=(0, 5))
        self.timer_label = tb.Label(frm, text="", bootstyle="secondary")
        self.timer_label.grid(row=7, column=3, columnspan=2, sticky="e", pady=(0, 5))
        self.count_label = tb.Label(frm, text="", bootstyle="secondary")
        self.count_label.grid(row=7, column=5, sticky="e", pady=(0, 5))

        self.lbl_log = tb.Label(frm, text="")
        self.lbl_log.grid(row=8, column=0, sticky="nw", pady=5)
        self.log_text = scrolledtext.ScrolledText(
            frm, width=80, height=16, state=tk.DISABLED, font=(self.mono_font, 9))
        self.log_text.grid(row=8, column=1, columnspan=5, sticky="nsew", pady=5)

        frm.columnconfigure(1, weight=1)
        frm.rowconfigure(8, weight=1)

    def apply_lang(self):
        self.root.title(self.tr("title"))
        self.lang_btn.config(text=self.tr("lang_btn"))
        self.lbl_account.config(text=self.tr("account"))
        self.new_acc_btn.config(text=self.tr("new_account"))
        self.del_acc_btn.config(text=self.tr("delete_account"))
        self.lbl_user.config(text=self.tr("username"))
        self.lbl_pwd.config(text=self.tr("password"))
        self.lbl_course.config(text=self.tr("course"))
        self.lbl_interval.config(text=self.tr("interval"))
        self.lbl_apply_interval.config(text=self.tr("apply_interval"))
        self.start_btn.config(text=self.tr("start"))
        self.stop_btn.config(text=self.tr("stop"))
        self.export_btn.config(text=self.tr("export"))
        self.log_dir_btn.config(text=self.tr("open_log_dir"))
        self.driver_btn.config(text=self.tr("driver_btn"))
        self.data_btn.config(text=self.tr("data_btn"))
        self.contact_btn.config(text=self.tr("contact_btn"))
        self.settings_btn.config(text=self.tr("settings_btn"))
        self.uninstall_btn.config(text=self.tr("uninstall_btn"))
        self.lbl_log.config(text=self.tr("log"))
        self.pwd_btn.config(text=self.tr("show_pwd"))
        accounts = self.cfg.get("accounts", [])
        self.account_combo["values"] = [a.get("username") or self.tr("new_account_name")
                                        for a in accounts]
        idx = self.cfg.get("current_index", 0)
        if 0 <= idx < len(self.account_combo["values"]):
            self.account_var.set(self.account_combo["values"][idx])
        self.update_theme_btn_text()
        self.course_entry["values"] = self.cfg.get("course_history", [])

    def toggle_lang(self):
        self.lang = "en" if self.lang == "zh" else "zh"
        self.cfg["lang"] = self.lang
        save_config(self.cfg)
        self.apply_lang()

    def toggle_pwd(self):
        if self.password_entry.cget("show") == "*":
            self.password_entry.config(show="")
            self.pwd_btn.config(text=self.tr("hide_pwd"))
        else:
            self.password_entry.config(show="*")
            self.pwd_btn.config(text=self.tr("show_pwd"))

    def show_welcome(self):
        if self.welcome_visible:
            return
        self.welcome_visible = True
        self.welcome_label = tb.Label(self.root, text=self.tr("welcome_tip"),
                                      bootstyle="warning", padding=6)
        self.welcome_label.place(relx=0.5, y=4, anchor="n")
        self.root.after(8000, self.hide_welcome)

    def hide_welcome(self):
        if self.welcome_visible:
            try:
                self.welcome_label.destroy()
            except Exception:
                pass
            self.welcome_visible = False

    # ---------- 主题 ----------
    def apply_system_theme(self):
        theme = self.platform_conf["theme_dark"] if self.current_system_theme == "dark" else self.platform_conf["theme_light"]
        try:
            self.root.style.theme_use(theme)
        except Exception:
            pass

    def apply_theme_mode(self):
        if not self.s("follow_system_theme"):
            self.theme_follow_system = False
            mode = self.cfg.get("theme_mode", "light")
            if mode == "system":
                mode = "light"
            theme = self.platform_conf["theme_light"] if mode == "light" else self.platform_conf["theme_dark"]
            try:
                self.root.style.theme_use(theme)
            except Exception:
                pass
            self.update_theme_btn_text()
            return
        mode = self.cfg.get("theme_mode", "system")
        if mode == "system":
            self.theme_follow_system = True
            self.current_system_theme = detect_system_theme()
            self.apply_system_theme()
        else:
            self.theme_follow_system = False
            theme = self.platform_conf["theme_light"] if mode == "light" else self.platform_conf["theme_dark"]
            try:
                self.root.style.theme_use(theme)
            except Exception:
                pass
        self.update_theme_btn_text()

    def update_theme_btn_text(self):
        mode = self.cfg.get("theme_mode", "system")
        if not self.s("follow_system_theme"):
            mode = "light" if mode in ("system", "light") else "dark"
        if self.lang == "zh":
            m = {"system": "跟随系统", "light": "浅色", "dark": "深色"}
        else:
            m = {"system": "System", "light": "Light", "dark": "Dark"}
        try:
            self.theme_btn.config(text=f"{self.tr('theme_btn')}：{m[mode]}")
        except Exception:
            pass

    def cycle_theme(self):
        mode = self.cfg.get("theme_mode", "system")
        mode = {"system": "light", "light": "dark", "dark": "system"}[mode]
        self.cfg["theme_mode"] = mode
        save_config(self.cfg)
        self.apply_theme_mode()

    def poll_system_theme(self):
        if self.s("follow_system_theme") and self.theme_follow_system:
            new_theme = detect_system_theme()
            if new_theme != self.current_system_theme:
                self.current_system_theme = new_theme
                self.apply_system_theme()
            interval = 5000 if SYSTEM == "Linux" else 2000
        else:
            interval = 5000
        self.root.after(interval, self.poll_system_theme)

    # ---------- 快捷键 ----------
    def setup_shortcuts(self):
        self.root.bind("<F5>", lambda e: self.start_monitor())
        self.root.bind("<Escape>", lambda e: self.stop_monitor())
        self.root.bind("<Control-s>", lambda e: self.export_log())
        self.root.bind("<Control-comma>", lambda e: self.show_settings())
        self.course_entry.bind("<Return>", lambda e: self.start_monitor())
        self.password_entry.bind("<Return>", lambda e: self.start_monitor())

    # ---------- 账户 ----------
    def load_current_account(self):
        accounts = self.cfg.get("accounts", [])
        idx = self.cfg.get("current_index", 0)
        if not accounts:
            accounts = [{"username": "", "course": "", "interval": "15", "apply_interval": "2"}]
            self.cfg["accounts"] = accounts
            idx = 0
        if idx < 0 or idx >= len(accounts):
            idx = 0
            self.cfg["current_index"] = 0
        acc = accounts[idx]
        self.account_combo["values"] = [a.get("username") or self.tr("new_account_name")
                                        for a in accounts]
        self.account_var.set(self.account_combo["values"][idx])
        self.username_var.set(acc.get("username", ""))
        self.course_var.set(acc.get("course", ""))
        self.interval_var.set(acc.get("interval", "15"))
        self.apply_interval_var.set(acc.get("apply_interval", "2"))
        self.password_var.set(load_password(acc.get("username", "")))

    def is_monitoring(self):
        return self.worker_thread is not None and self.worker_thread.is_alive()

    def on_account_select(self, event=None):
        if self.is_monitoring():
            messagebox.showwarning("提示", self.tr("monitoring_no_switch"))
            idx = self.cfg.get("current_index", 0)
            values = self.account_combo["values"]
            if 0 <= idx < len(values):
                self.account_var.set(values[idx])
            return
        idx = self.account_combo.current()
        if idx < 0:
            return
        self.cfg["current_index"] = idx
        save_config(self.cfg)
        acc = self.cfg["accounts"][idx]
        self.username_var.set(acc.get("username", ""))
        self.course_var.set(acc.get("course", ""))
        self.interval_var.set(acc.get("interval", "15"))
        self.apply_interval_var.set(acc.get("apply_interval", "2"))
        self.password_var.set(load_password(acc.get("username", "")))

    def new_account(self):
        if self.is_monitoring():
            messagebox.showwarning("提示", self.tr("monitoring_no_switch"))
            return
        accounts = self.cfg.get("accounts", [])
        accounts.append({"username": "", "course": "", "interval": "15", "apply_interval": "2"})
        self.cfg["accounts"] = accounts
        self.cfg["current_index"] = len(accounts) - 1
        save_config(self.cfg)
        self.load_current_account()
        self.username_var.set("")
        self.password_var.set("")
        self.course_var.set("")
        self.interval_var.set("15")
        self.apply_interval_var.set("2")

    def delete_account(self):
        if self.is_monitoring():
            messagebox.showwarning("提示", self.tr("monitoring_no_switch"))
            return
        accounts = self.cfg.get("accounts", [])
        if len(accounts) <= 1:
            messagebox.showwarning("提示", self.tr("last_account_warn"))
            return
        if not messagebox.askyesno("提示", self.tr("confirm_delete")):
            return
        idx = self.cfg.get("current_index", 0)
        accounts.pop(idx)
        self.cfg["accounts"] = accounts
        if idx >= len(accounts):
            idx = len(accounts) - 1
        self.cfg["current_index"] = idx
        save_config(self.cfg)
        self.load_current_account()

    # ---------- 数据/联系 ----------
    def show_data_info(self):
        msg = self.tr("data_msg").format(
            config=CONFIG_FILE,
            pwd=PASSWORD_FILE if not HAS_KEYRING else "系统凭据管理器",
            log=LOG_DIR)
        win = tb.Toplevel(self.root)
        win.title(self.tr("data_title"))
        win.geometry("520x340")
        center_window(win, 520, 340)
        win.transient(self.root)
        win.grab_set()
        frame = tb.Frame(win, padding=20)
        frame.pack(fill="both", expand=True)
        tb.Label(frame, text=msg, wraplength=480, justify="left").pack(pady=10)
        tb.Button(frame, text=self.tr("close"), command=win.destroy,
                  bootstyle="secondary").pack(pady=10)

    def copy_to_clipboard(self, text):
        try:
            self.root.clipboard_clear()
            self.root.clipboard_append(text)
            self.root.update()
            messagebox.showinfo("提示", self.tr("copied"))
        except Exception:
            pass

    def show_contact(self):
        win = tb.Toplevel(self.root)
        win.title(self.tr("contact_title"))
        win.geometry("480x400")
        center_window(win, 480, 400)
        win.transient(self.root)
        win.grab_set()
        frame = tb.Frame(win, padding=20)
        frame.pack(fill="both", expand=True)
        tb.Label(frame, text=self.tr("contact_title"),
                 font=(self.font_family, 14, "bold"), bootstyle="primary").pack(pady=(0, 5))
        tb.Label(frame, text=self.tr("contact_subtitle"),
                 bootstyle="secondary").pack(pady=(0, 15))
        for key, label in [("email", self.tr("contact_email")),
                           ("wechat", self.tr("contact_wechat")),
                           ("qq", self.tr("contact_qq")),
                           ("github", self.tr("contact_github"))]:
            value = CONTACT_INFO.get(key, "").strip()
            if not value:
                continue
            row = tb.Frame(frame)
            row.pack(fill="x", pady=4)
            tb.Label(row, text=f"{label}：", width=8, anchor="e").pack(side="left")
            e = tb.Entry(row, width=26)
            e.insert(0, value)
            e.config(state="readonly")
            e.pack(side="left", padx=5)
            tb.Button(row, text=self.tr("copy"), bootstyle="secondary-outline", width=6,
                      command=lambda v=value: self.copy_to_clipboard(v)).pack(side="left")
        tb.Button(frame, text=self.tr("close"), bootstyle="secondary",
                  command=win.destroy, width=10).pack(pady=20)

    # ---------- 设置 ----------
    def reset_onboarding(self):
        if not messagebox.askyesno(
                self.tr("reset_onboarding"),
                self.tr("reset_onboarding_confirm")):
            return
        self.cfg["agreed"] = False
        self.cfg["platform"] = ""
        self.cfg["low_spec_warned"] = False
        self.cfg["guide_shown"] = False
        self.cfg["tray_tip_shown"] = False
        save_config(self.cfg)
        messagebox.showinfo("提示", self.tr("reset_onboarding_done"))

    def show_settings(self):
        win = tb.Toplevel(self.root)
        win.title(self.tr("settings_title"))
        win.geometry("560x620")
        center_window(win, 560, 620)
        win.transient(self.root)
        win.grab_set()
        frame = tb.Frame(win, padding=20)
        frame.pack(fill="both", expand=True)
        tb.Label(frame, text=self.tr("settings_title"),
                 font=(self.font_family, 14, "bold"), bootstyle="primary").pack(pady=(0, 5))
        tb.Label(frame, text=self.tr("settings_subtitle"),
                 bootstyle="secondary").pack(pady=(0, 10))

        canvas = tk.Canvas(frame, highlightthickness=0)
        sb = tb.Scrollbar(frame, orient="vertical", command=canvas.yview)
        inner = tb.Frame(canvas)
        inner.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.create_window((0, 0), window=inner, anchor="nw")
        canvas.configure(yscrollcommand=sb.set)
        canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        sb.pack(side=tk.RIGHT, fill=tk.Y)

        browser_row = tb.Frame(inner)
        browser_row.pack(anchor="w", pady=8)
        tb.Label(browser_row, text=self.tr("settings_browser") + "：",
                 font=(self.font_family, 10)).pack(side=tk.LEFT)
        browser_map = {"Microsoft Edge": "edge", "Google Chrome": "chrome",
                       "Mozilla Firefox": "firefox"}
        current_name = {v: k for k, v in browser_map.items()}.get(
            self.cfg.get("browser", "edge"), "Microsoft Edge")
        name_var = tk.StringVar(value=current_name)
        tb.Combobox(browser_row, textvariable=name_var,
                    values=list(browser_map.keys()), state="readonly",
                    width=20).pack(side=tk.LEFT, padx=5)

        items = [
            ("tray_minimize", self.tr("settings_tray")),
            ("follow_system_theme", self.tr("settings_follow_theme")),
            ("fade_in", self.tr("settings_fade")),
            ("pulse_status", self.tr("settings_pulse")),
            ("highlight_log", self.tr("settings_highlight")),
            ("flash_on_found", self.tr("settings_flash")),
            ("log_to_file", self.tr("settings_log_file")),
            ("log_by_day", self.tr("settings_log_by_day")),
            ("log_auto_scroll", self.tr("settings_log_scroll")),
            ("log_limit", self.tr("settings_log_limit")),
            ("auto_download_driver", self.tr("settings_auto_driver")),
            ("notify_tray", self.tr("settings_notify")),
            ("confirm_start", self.tr("settings_confirm_start")),
            ("confirm_close", self.tr("settings_confirm_close")),
        ]
        vars = {}
        for key, text in items:
            v = tk.BooleanVar(value=self.s(key))
            vars[key] = v
            tb.Checkbutton(inner, text=text, variable=v,
                           bootstyle="primary-round-toggle").pack(anchor="w", pady=4)

        def save():
            new_browser = browser_map.get(name_var.get(), "edge")
            if new_browser != self.cfg.get("browser", "edge"):
                if not browser_is_available(new_browser):
                    if not messagebox.askyesno(
                            "提示",
                            self.tr("browser_missing").format(
                                browser=browser_display_name(new_browser))):
                        return
                self.cfg["browser"] = new_browser
                self.driver_checked = False
            for k, v in vars.items():
                self.cfg["settings"][k] = v.get()
            save_config(self.cfg)
            if not self.s("follow_system_theme"):
                self.theme_follow_system = False
            else:
                self.apply_theme_mode()
            messagebox.showinfo("提示", self.tr("settings_saved"))
            win.destroy()

        bf = tb.Frame(win)
        bf.pack(pady=10)
        tb.Button(bf, text=self.tr("settings_save"), command=save,
                  bootstyle="success").pack(side=tk.LEFT, padx=8)
        tb.Button(bf, text=self.tr("settings_cancel"), command=win.destroy,
                  bootstyle="secondary").pack(side=tk.LEFT, padx=8)
        tb.Button(bf, text=self.tr("reset_onboarding"),
                  command=self.reset_onboarding,
                  bootstyle="warning-outline").pack(side=tk.LEFT, padx=8)

    # ---------- 清理/卸载 ----------
    def ask_export_password(self):
        win = tb.Toplevel(self.root)
        win.title(self.tr("encrypt_title"))
        win.geometry("420x280")
        center_window(win, 420, 280)
        win.transient(self.root)
        win.grab_set()
        frame = tb.Frame(win, padding=20)
        frame.pack(fill="both", expand=True)
        tb.Label(frame, text=self.tr("encrypt_hint"), wraplength=360,
                 justify="left").pack(pady=(0, 15))
        tb.Label(frame, text=self.tr("encrypt_pwd")).pack(anchor="w")
        pwd_var = tk.StringVar()
        tb.Entry(frame, textvariable=pwd_var, show="*").pack(fill="x", pady=5)
        tb.Label(frame, text=self.tr("encrypt_confirm")).pack(anchor="w")
        pwd2_var = tk.StringVar()
        tb.Entry(frame, textvariable=pwd2_var, show="*").pack(fill="x", pady=5)
        result = {"pwd": None}

        def ok():
            pwd = pwd_var.get()
            if not pwd:
                messagebox.showwarning("提示", self.tr("encrypt_empty"))
                return
            if pwd != pwd2_var.get():
                messagebox.showwarning("提示", self.tr("encrypt_mismatch"))
                return
            result["pwd"] = pwd
            win.destroy()

        bf = tb.Frame(frame)
        bf.pack(pady=15)
        tb.Button(bf, text=self.tr("encrypt_ok"), command=ok,
                  bootstyle="success").pack(side=tk.LEFT, padx=10)
        tb.Button(bf, text=self.tr("encrypt_cancel"), command=win.destroy,
                  bootstyle="secondary").pack(side=tk.LEFT, padx=10)
        win.wait_window()
        return result["pwd"]

    def show_uninstall(self):
        if self.is_monitoring():
            messagebox.showwarning("提示", self.tr("uninstall_running"))
            return
        win = tb.Toplevel(self.root)
        win.title(self.tr("uninstall_title"))
        win.geometry("500x540")
        center_window(win, 500, 540)
        win.transient(self.root)
        win.grab_set()
        frame = tb.Frame(win, padding=20)
        frame.pack(fill="both", expand=True)
        tb.Label(frame, text=self.tr("uninstall_title"),
                 font=(self.font_family, 14, "bold"), bootstyle="danger").pack(pady=(0, 5))
        tb.Label(frame, text=self.tr("uninstall_subtitle"),
                 bootstyle="secondary").pack(pady=(0, 10))
        keep_var = tk.BooleanVar(value=True)
        tb.Checkbutton(frame, text=self.tr("keep_info"), variable=keep_var,
                       bootstyle="info-round-toggle").pack(anchor="w", pady=(0, 2))
        tb.Label(frame, text=self.tr("keep_info_hint"), bootstyle="secondary",
                 wraplength=440, justify="left").pack(anchor="w", pady=(0, 10))
        vars = {}

        def add_check(key, text, default=True):
            v = tk.BooleanVar(value=default)
            vars[key] = v
            tb.Checkbutton(frame, text=text, variable=v,
                           bootstyle="danger-round-toggle").pack(anchor="w", pady=3)

        add_check("config", self.tr("uninstall_config"))
        if os.path.exists(PASSWORD_FILE):
            add_check("passwords", self.tr("uninstall_passwords"))
        if HAS_KEYRING:
            add_check("keyring", self.tr("uninstall_keyring"))
        if os.path.isdir(LOG_DIR):
            add_check("logs", self.tr("uninstall_logs"))
        is_frozen = getattr(sys, "frozen", False)
        if is_frozen:
            add_check("self", self.tr("uninstall_self"), default=False)
            tb.Label(frame, text=self.tr("uninstall_self_hint"), bootstyle="warning",
                     wraplength=440).pack(anchor="w", pady=(0, 5))

        def do_delete():
            selected = [k for k, v in vars.items() if v.get()]
            if not selected:
                messagebox.showinfo("提示", self.tr("uninstall_nothing"))
                return
            if not messagebox.askyesno("提示", self.tr("uninstall_warn")):
                return
            if keep_var.get():
                password = None
                want_encrypt = messagebox.askyesno(self.tr("encrypt_ask_title"),
                                                   self.tr("encrypt_ask_msg"))
                if want_encrypt:
                    if not HAS_CRYPTO:
                        if not messagebox.askyesno("提示", self.tr("encrypt_no_lib")):
                            return
                    else:
                        password = self.ask_export_password()
                        if password is None:
                            return
                try:
                    path = export_user_info(self.cfg, self.lang, password)
                    if password:
                        messagebox.showinfo("提示", self.tr("encrypt_done").format(path=path))
                    else:
                        messagebox.showinfo("提示", self.tr("export_done").format(path=path))
                except Exception as e:
                    messagebox.showerror("提示", self.tr("export_fail").format(err=e))
                    return
            deleted = []
            if "config" in selected:
                try:
                    if os.path.exists(CONFIG_FILE):
                        os.remove(CONFIG_FILE)
                        deleted.append("config.json")
                except Exception as e:
                    print(e)
            if "passwords" in selected:
                try:
                    if os.path.exists(PASSWORD_FILE):
                        os.remove(PASSWORD_FILE)
                        deleted.append("passwords.json")
                except Exception as e:
                    print(e)
            if "keyring" in selected and HAS_KEYRING:
                for acc in self.cfg.get("accounts", []):
                    uname = acc.get("username", "").strip()
                    if uname:
                        try:
                            keyring.delete_password(KEYRING_SERVICE, uname)
                        except Exception:
                            pass
                deleted.append("keyring")
            if "logs" in selected:
                try:
                    if os.path.isdir(LOG_DIR):
                        shutil.rmtree(LOG_DIR)
                        deleted.append("logs/")
                except Exception as e:
                    print(e)
            schedule_self = "self" in selected and is_frozen
            messagebox.showinfo("提示", self.tr("uninstall_done").format(items="\n".join(deleted) or "无"))
            if schedule_self:
                if schedule_self_delete():
                    messagebox.showinfo("提示", self.tr("uninstall_self_scheduled"))
            win.destroy()
            self.quit_app()

        bf = tb.Frame(frame)
        bf.pack(pady=15)
        tb.Button(bf, text=self.tr("uninstall_confirm"), command=do_delete,
                  bootstyle="danger").pack(side=tk.LEFT, padx=10)
        tb.Button(bf, text=self.tr("uninstall_cancel"), command=win.destroy,
                  bootstyle="secondary").pack(side=tk.LEFT, padx=10)

    # ---------- 托盘 ----------
    def setup_tray(self):
        if not HAS_TRAY:
            return
        try:
            image = load_tray_image()
            menu = pystray.Menu(
                pystray.MenuItem(self.tr("tray_show"), self.show_window, default=True),
                pystray.MenuItem(self.tr("tray_quit"), self.quit_app))
            self.tray_icon = pystray.Icon("course_monitor", image, self.tr("tray_tip"), menu)
            threading.Thread(target=self.tray_icon.run, daemon=True).start()
        except Exception as e:
            print("托盘初始化失败：", e)
            self.tray_icon = None

    def update_tray_icon(self, running):
        if not self.tray_icon:
            return
        try:
            self.tray_icon.icon = load_tray_image(color=(46, 204, 113) if running else None)
            self.tray_icon.title = self.tr("tray_tip")
        except Exception:
            pass

    def show_window(self, icon=None, item=None):
        self.root.after(0, self.root.deiconify)

    def quit_app(self, icon=None, item=None):
        self.stop_event.set()
        try:
            self.cfg["geometry"] = self.root.geometry()
            save_config(self.cfg)
        except Exception:
            pass
        if self.tray_icon:
            try:
                self.tray_icon.stop()
            except Exception:
                pass
        self.root.after(0, self.root.destroy)

    def on_closing(self):
        if self.is_monitoring() and self.s("confirm_close"):
            if not messagebox.askyesno(self.tr("confirm_close_title"),
                                       self.tr("confirm_close_msg")):
                return
        try:
            self.cfg["geometry"] = self.root.geometry()
            save_config(self.cfg)
        except Exception:
            pass
        if self.tray_icon and self.s("tray_minimize"):
            self.root.withdraw()
            self.log(self.tr("hidden_to_tray"))
            if self.s("notify_tray") and not self.cfg.get("tray_tip_shown", False):
                self.cfg["tray_tip_shown"] = True
                save_config(self.cfg)
                try:
                    self.tray_icon.notify(self.tr("close_to_tray_tip"),
                                          self.tr("tray_tip_title"))
                except Exception:
                    pass
        else:
            self.quit_app()

    # ---------- 动画 ----------
    def fade_in(self, step=0.0):
        if step >= 1.0:
            try:
                self.root.attributes("-alpha", 1.0)
            except Exception:
                pass
            return
        try:
            self.root.attributes("-alpha", step)
        except Exception:
            return
        self.root.after(30, lambda: self.fade_in(step + 0.1))

    def start_pulse(self):
        if not self.s("pulse_status"):
            try:
                self.status_canvas.itemconfig(self.status_dot, fill="#2ecc71")
            except Exception:
                pass
            return
        self.pulse_on = True
        self.pulse_index = 0
        self._pulse()

    def stop_pulse(self):
        self.pulse_on = False
        try:
            self.status_canvas.itemconfig(self.status_dot, fill="gray")
        except Exception:
            pass

    def _pulse(self):
        if not self.pulse_on:
            return
        colors = ["#2ecc71", "#27ae60", "#1e8449", "#27ae60"]
        self.pulse_index = (self.pulse_index + 1) % len(colors)
        try:
            self.status_canvas.itemconfig(self.status_dot, fill=colors[self.pulse_index])
        except Exception:
            pass
        self.root.after(400, self._pulse)

    def flash_window(self):
        try:
            self.root.bell()
        except Exception:
            pass

        def do_flash(count):
            if count >= 6:
                try:
                    self.root.config(bg="")
                except Exception:
                    pass
                return
            color = "#ffcccc" if count % 2 == 0 else ""
            try:
                self.root.config(bg=color)
            except Exception:
                pass
            self.root.after(150, lambda: do_flash(count + 1))
        do_flash(0)

    # ---------- 日志 ----------
    def log(self, msg, level="info"):
        ts = datetime.datetime.now().strftime("%H:%M:%S")
        line = f"[{ts}] {msg}"
        self.log_queue.put((line, level))
        if self.current_username and self.s("log_to_file"):
            try:
                if self.s("log_by_day"):
                    today = datetime.datetime.now().strftime("%Y%m%d")
                    fname = f"{safe_filename(self.current_username)}_{today}.log"
                else:
                    fname = f"{safe_filename(self.current_username)}.log"
                path = os.path.join(LOG_DIR, fname)
                if os.path.exists(path) and os.path.getsize(path) > 5 * 1024 * 1024:
                    backup = path + ".1"
                    if os.path.exists(backup):
                        os.remove(backup)
                    os.rename(path, backup)
                with open(path, "a", encoding="utf-8") as f:
                    f.write(line + "\n")
            except Exception:
                pass

    def poll_log(self):
        colors = {"info": None, "success": "#27ae60",
                  "warn": "#e67e22", "error": "#e74c3c"}
        try:
            while True:
                item = self.log_queue.get_nowait()
                if isinstance(item, tuple):
                    msg, level = item
                else:
                    msg, level = item, "info"
                self.log_text.config(state=tk.NORMAL)
                do_hl = self.s("highlight_log")
                if do_hl:
                    start = self.log_text.index("end-1c")
                self.log_text.insert(tk.END, msg + "\n", level)
                if colors.get(level):
                    self.log_text.tag_config(level, foreground=colors[level])
                if do_hl:
                    end = self.log_text.index("end-1c")
                    self.log_text.tag_add("new_line", start, end)
                    self.log_text.tag_config("new_line", background="#fff9c4")
                    self.root.after(800, lambda s=start, e=end: self.log_text.tag_remove("new_line", s, e))
                if self.s("log_auto_scroll"):
                    self.log_text.see(tk.END)
                if self.s("log_limit"):
                    try:
                        total = int(self.log_text.index("end-1c").split(".")[0])
                        if total > 2000:
                            self.log_text.delete("1.0", "500.0")
                    except Exception:
                        pass
                self.log_text.config(state=tk.DISABLED)
        except queue.Empty:
            pass
        if self.worker_thread and not self.worker_thread.is_alive():
            self.start_btn.config(state=tk.NORMAL)
            self.stop_btn.config(state=tk.DISABLED)
            self.account_combo.config(state="readonly")
            self.new_acc_btn.config(state=tk.NORMAL)
            self.del_acc_btn.config(state=tk.NORMAL)
            self.worker_thread = None
            self.stop_pulse()
            self.update_tray_icon(False)
            try:
                self.progress.config(value=0)
            except Exception:
                pass
            self.status_label.config(text=self.tr("status_idle"))
            self.timer_label.config(text="")
        self.root.after(100, self.poll_log)

    def update_status_timer(self):
        if not self.is_monitoring():
            return
        elapsed = int(time.time() - self.start_time)
        h, m, s = elapsed // 3600, elapsed % 3600 // 60, elapsed % 60
        try:
            self.timer_label.config(text=self.tr("status_elapsed").format(h=h, m=m, s=s))
            self.count_label.config(text=self.tr("status_checks").format(
                c=self.check_count, f=self.found_count))
        except Exception:
            pass
        self.root.after(1000, self.update_status_timer)

    # ---------- 驱动 ----------
    def driver_cache_exists(self):
        home = os.path.expanduser("~")
        b = self.cfg.get("browser", "edge")
        name = {"chrome": "chromedriver", "firefox": "geckodriver"}.get(b, "msedgedriver")
        cache_dir = os.path.join(home, ".cache", "selenium", name)
        if os.path.exists(cache_dir):
            try:
                return any(os.path.isdir(os.path.join(cache_dir, item))
                           for item in os.listdir(cache_dir))
            except Exception:
                pass
        return False

    def ensure_driver(self, on_success):
        if not self.s("auto_download_driver"):
            self.driver_checked = True
            on_success()
            return
        if self.driver_checked:
            on_success()
            return
        if self.driver_cache_exists():
            self.driver_checked = True
            on_success()
            return
        self.log(self.tr("driver_checking"))

        def check():
            try:
                b = self.cfg.get("browser", "edge")
                d = create_browser_driver(b, headless=True)
                d.quit()
                self.root.after(0, lambda: self._on_driver_ready(on_success))
            except Exception as e:
                err = str(e)
                self.root.after(0, lambda err=err: self._on_driver_missing(err, on_success))

        threading.Thread(target=check, daemon=True).start()

    def _on_driver_ready(self, on_success):
        self.driver_checked = True
        on_success()

    def _on_driver_missing(self, err, on_success):
        keywords = ["Unable to obtain driver", "NoSuchDriverException",
                    "executable needs to be in PATH", "Unable to find",
                    "msedgedriver", "chromedriver", "geckodriver"]
        is_missing = any(k in err for k in keywords)
        if not is_missing:
            messagebox.showerror("提示", self.tr("driver_start_fail").format(err=err))
            self._reset_buttons()
            return
        if not messagebox.askyesno(self.tr("driver_title"), self.tr("driver_explain")):
            self._reset_buttons()
            return
        self._download_driver(on_success)

    def _download_driver(self, on_success):
        win = tb.Toplevel(self.root)
        win.title(self.tr("driver_title"))
        win.geometry("440x180")
        center_window(win, 440, 180)
        win.transient(self.root)
        win.grab_set()
        frame = tb.Frame(win, padding=20)
        frame.pack(fill="both", expand=True)
        tb.Label(frame, text=self.tr("driver_downloading"), wraplength=380).pack(pady=(0, 10))
        pb = tb.Progressbar(frame, mode="indeterminate", length=360)
        pb.pack(pady=5)
        pb.start(12)
        tb.Label(frame, text=self.tr("driver_wait")).pack(pady=5)

        def do_download():
            try:
                b = self.cfg.get("browser", "edge")
                d = create_browser_driver(b, headless=True)
                d.quit()
                self.root.after(0, lambda: self._on_download_done(win, pb, True, "", on_success))
            except Exception as e:
                err = str(e)
                self.root.after(0, lambda err=err: self._on_download_done(win, pb, False, err, on_success))

        threading.Thread(target=do_download, daemon=True).start()

    def _on_download_done(self, win, pb, success, err, on_success):
        try:
            pb.stop()
        except Exception:
            pass
        win.destroy()
        if success:
            self.driver_checked = True
            self.log(self.tr("driver_ok"), "success")
            messagebox.showinfo("提示", self.tr("driver_ok"))
            on_success()
        else:
            messagebox.showerror("提示", self.tr("driver_fail").format(err=err))
            self._reset_buttons()

    def _reset_buttons(self):
        self.start_btn.config(state=tk.NORMAL)
        self.stop_btn.config(state=tk.DISABLED)
        self.account_combo.config(state="readonly")
        self.new_acc_btn.config(state=tk.NORMAL)
        self.del_acc_btn.config(state=tk.NORMAL)
        self.stop_pulse()
        self.update_tray_icon(False)

    def manual_download_driver(self):
        self.driver_checked = False
        self.ensure_driver(lambda: messagebox.showinfo("提示", self.tr("driver_ok")))

    # ---------- 低配提示 ----------
    def maybe_warn_low_spec(self):
        if self.cfg.get("low_spec_warned", False):
            return True
        reasons = get_low_spec_reason()
        if not reasons:
            self.cfg["low_spec_warned"] = True
            save_config(self.cfg)
            return True
        mem = round(get_total_memory_gb(), 1)
        cpu = get_cpu_name()
        if len(reasons) == 2:
            reason_text = self.tr("low_spec_reason_both")
        elif reasons[0] == "mem":
            reason_text = self.tr("low_spec_reason_mem")
        else:
            reason_text = self.tr("low_spec_reason_cpu")
        answer = messagebox.askyesno(
            self.tr("low_spec_title"),
            f"（{reason_text}）\n\n" + self.tr("low_spec_msg").format(mem=mem, cpu=cpu))
        self.cfg["low_spec_warned"] = True
        save_config(self.cfg)
        if not answer:
            return True
        self.show_settings()
        return False

    # ---------- 开始 / 停止 ----------
    def start_monitor(self):
        username = self.username_var.get().strip()
        password = self.password_var.get().strip()
        course = self.course_var.get().strip()
        interval_str = self.interval_var.get().strip()
        apply_interval_str = self.apply_interval_var.get().strip()
        
        if not username or not password or not course:
            messagebox.showwarning("提示", self.tr("warn_empty"))
            return
            
        try:
            interval = float(interval_str)
            apply_interval = float(apply_interval_str)
            if interval < 1 or apply_interval < 1:
                raise ValueError
        except ValueError:
            messagebox.showwarning("提示", self.tr("warn_interval"))
            return

        if not self.maybe_warn_low_spec():
            return
        if self.s("confirm_start"):
            masked = username[:3] + "***" if len(username) > 3 else username
            if not messagebox.askyesno(
                    self.tr("confirm_start_title"),
                    self.tr("confirm_start_msg").format(
                        user=masked, course=course, interval=interval, apply_interval=apply_interval)):
                return
        self.hide_welcome()

        idx = self.cfg.get("current_index", 0)
        accounts = self.cfg.get("accounts", [])
        if idx < 0 or idx >= len(accounts):
            idx = 0
            self.cfg["current_index"] = 0
            if not accounts:
                accounts = [{}]
                self.cfg["accounts"] = accounts
        accounts[idx]["username"] = username
        accounts[idx]["course"] = course
        accounts[idx]["interval"] = interval_str
        accounts[idx]["apply_interval"] = apply_interval_str
        save_password(username, password)
        history = self.cfg.get("course_history", [])
        if course and course not in history:
            history.append(course)
            self.cfg["course_history"] = history[-20:]
        save_config(self.cfg)
        self.account_combo["values"] = [a.get("username") or self.tr("new_account_name")
                                        for a in accounts]
        self.account_var.set(username or self.tr("new_account_name"))
        self.course_entry["values"] = self.cfg.get("course_history", [])
        self.current_username = username
        self.check_count = 0
        self.found_count = 0

        self.start_btn.config(state=tk.DISABLED)
        self.stop_btn.config(state=tk.NORMAL)
        self.account_combo.config(state=tk.DISABLED)
        self.new_acc_btn.config(state=tk.DISABLED)
        self.del_acc_btn.config(state=tk.DISABLED)
        self.stop_event.clear()
        self.start_pulse()
        self.update_tray_icon(True)
        self.status_label.config(text=self.tr("status_running"))
        self.start_time = time.time()
        self.update_status_timer()

        def really_start():
            self.worker_thread = threading.Thread(
                target=self.worker,
                args=(username, password, course, interval, apply_interval), daemon=True)
            self.worker_thread.start()
            self.log(self.tr("started"))
        self.ensure_driver(really_start)

    def stop_monitor(self):
        self.stop_event.set()
        self.log(self.tr("stopping"))
        self.stop_btn.config(state=tk.DISABLED)

    def export_log(self):
        content = self.log_text.get("1.0", tk.END).strip()
        if not content:
            messagebox.showinfo("提示", self.tr("log_empty"))
            return
        path = filedialog.asksaveasfilename(defaultextension=".txt",
                                            filetypes=[("Text", "*.txt")])
        if path:
            with open(path, "w", encoding="utf-8") as f:
                f.write(content)
            messagebox.showinfo("提示", self.tr("log_saved").format(path=path))

    def open_log_dir(self):
        path = os.path.abspath(LOG_DIR)
        try:
            os.makedirs(path, exist_ok=True)
            if SYSTEM == "Windows":
                os.startfile(path)
            elif SYSTEM == "Darwin":
                subprocess.Popen(["open", path])
            else:
                subprocess.Popen(["xdg-open", path])
        except Exception as e:
            messagebox.showinfo("提示", f"{path}\n{e}")

    # ---------- 验证码/浏览器存活 ----------
    def has_captcha(self, driver):
        selectors = ["img[src*='captcha']", "img[src*='verify']",
                     "img[src*='checkcode']", "input[id*='captcha']",
                     "input[id*='verify']", "input[name*='captcha']",
                     "#captcha", ".captcha", ".verify-code"]
        for sel in selectors:
            try:
                for e in driver.find_elements(By.CSS_SELECTOR, sel):
                    if e.is_displayed():
                        return True
            except Exception:
                continue
        return False

    def wait_captcha_done(self):
        event = threading.Event()

        def ask():
            self.root.bell()
            messagebox.showinfo(self.tr("captcha_title"), self.tr("captcha_msg"))
            event.set()
        self.root.after(0, ask)
        event.wait(timeout=180)

    def is_browser_alive(self, driver):
        try:
            _ = driver.current_url
            return True
        except Exception:
            return False

    # ---------- 子线程 (含申请内层循环) ----------
    def worker(self, username, password, course, check_interval, apply_interval):
        driver = None
        try:
            self.log(self.tr("browser_start"))
            b = self.cfg.get("browser", "edge")
            driver = create_browser_driver(b)

            # ================== 第一步：统一身份认证登录 ==================
            LOGIN_URL = "https://i.yzu.edu.cn"
            driver.get(LOGIN_URL)
            time.sleep(3)

            try:
                username_input = WebDriverWait(driver, 15).until(
                    EC.presence_of_element_located((By.CSS_SELECTOR, "#login-username input, input[name='username'], input[type='text']")))
                username_input.clear()
                username_input.send_keys(username)

                password_input = WebDriverWait(driver, 10).until(
                    EC.presence_of_element_located((By.CSS_SELECTOR, "input[type='password'], input[name='password']")))
                password_input.clear()
                password_input.send_keys(password)

                if self.has_captcha(driver):
                    self.log(self.tr("captcha_found"), "warn")
                    self.wait_captcha_done()

                login_btn = driver.find_element(By.CSS_SELECTOR, "button[type='submit'], input[type='submit'], #login-btn")
                login_btn.click()
                self.log(self.tr("login_submit"))
                time.sleep(5)

            except Exception as e:
                self.log(f"登录过程出错: {e}", "error")
                return

            if "login" in driver.current_url.lower() or "i.yzu.edu.cn" in driver.current_url:
                self.log(self.tr("login_fail"), "error")
                return

            # ================== 第二步：进入重修选课页面 ==================
            COURSE_PAGE_URL = "http://ydjwxs.yzu.edu.cn/student/courseSelect/cfxkcjf/index"
            driver.get(COURSE_PAGE_URL)
            time.sleep(3)
            self.log(self.tr("course_page"))

            # ================== 第三步：循环监控与选课 ==================
            while not self.stop_event.is_set():
                if not self.is_browser_alive(driver):
                    self.log(self.tr("browser_closed"), "warn")
                    break

                try:
                    # 1. 强制确保在“全部可选课程”选项卡（dealType=5）
                    driver.execute_script("$('#dealType').val('5');")

                    # 2. 在课程名输入框填入目标课程并查询
                    kcm_input = WebDriverWait(driver, 5).until(
                        EC.presence_of_element_located((By.ID, "kcm")))
                    kcm_input.clear()
                    kcm_input.send_keys(course)

                    query_btn = driver.find_element(By.XPATH, "//button[contains(@onclick, 'queryQbCourse')]")
                    query_btn.click()
                    time.sleep(2)

                    # 3. 等待课程表格加载
                    try:
                        WebDriverWait(driver, 5).until(
                            EC.presence_of_element_located((By.CSS_SELECTOR, "#qbkxkc_tbody tr")))
                    except Exception:
                        self.log(self.tr("not_found"))
                        self._wait_interval(check_interval)
                        continue

                    # 4. 遍历表格寻找目标课程
                    rows = driver.find_elements(By.CSS_SELECTOR, "#qbkxkc_tbody tr")
                    found = False

                    for row in rows:
                        try:
                            course_text = row.find_element(By.CSS_SELECTOR, "td:nth-child(3)").text
                            if course in course_text:
                                found = True
                                remaining_text = row.find_element(By.CSS_SELECTOR, "td:nth-child(8)").text
                                remaining = remaining_text.split("/")[0].strip()
                                self.log(self.tr("found").format(name=course_text, remaining=remaining))

                                if remaining != "0" and "已满" not in remaining:
                                    self.log(self.tr("try_select"), "success")
                                    self.found_count += 1
                                    
                                    # ================= 内层申请循环 =================
                                    apply_count = 0
                                    while not self.stop_event.is_set():
                                        if not self.is_browser_alive(driver):
                                            break
                                        
                                        # 勾选复选框
                                        checkbox = row.find_element(By.CSS_SELECTOR, "input[name='kcId']")
                                        if not checkbox.is_selected():
                                            checkbox.click()

                                        # 点击提交
                                        submit_btn = driver.find_element(By.XPATH, "//button[contains(@onclick, 'submitCourse')]")
                                        submit_btn.click()
                                        
                                        apply_count += 1
                                        self.log(f"第 {apply_count} 次申请已提交，等待 {apply_interval} 秒...")
                                        
                                        # 处理选课结果弹窗
                                        is_success = False
                                        try:
                                            WebDriverWait(driver, 8).until(
                                                EC.visibility_of_element_located((By.ID, "view-xkjg")))
                                            time.sleep(1.5)
                                            result_rows = driver.find_elements(By.CSS_SELECTOR, "#xkresult tr")
                                            for r in result_rows:
                                                res_text = r.text
                                                if "选课成功" in res_text:
                                                    is_success = True
                                                    self.log(f"🎉 选课成功: {course}", "success")
                                                    if self.s("flash_on_found"):
                                                        self.root.after(0, self.flash_window)
                                                elif "失败" in res_text or "已满" in res_text:
                                                    self.log(f"❌ 申请失败: {res_text}", "warn")
                                            
                                            # 关闭弹窗
                                            close_btn = driver.find_element(By.CSS_SELECTOR, "#view-xkjg .close")
                                            close_btn.click()
                                            time.sleep(1)
                                        except Exception as e:
                                            self.log(f"获取结果弹窗超时: {e}", "error")
                                            # 尝试强行关闭可能卡住的弹窗
                                            try:
                                                driver.execute_script("$('#view-xkjg').modal('hide');")
                                            except: pass
                                        
                                        if is_success:
                                            self.stop_event.set() # 成功即停止脚本
                                            break
                                            
                                        # 申请间隔等待
                                        for _ in range(int(apply_interval * 10)):
                                            if self.stop_event.is_set(): break
                                            time.sleep(0.1)
                                    # ====================================================
                                    break  # 处理完目标课程，跳出表格遍历
                        except Exception:
                            continue

                    if not found:
                        self.log(self.tr("not_found"))

                except Exception as e:
                    err = str(e)
                    if "net::" in err or "ERR_" in err:
                        self.log(self.tr("network_error"), "warn")
                        time.sleep(5)
                        continue
                    self.log(self.tr("monitor_error").format(err=e), "error")

                # 检查间隔等待
                self._wait_interval(check_interval)

        except Exception as e:
            self.log(self.tr("run_error").format(err=e), "error")
        finally:
            if driver:
                try:
                    driver.quit()
                except Exception:
                    pass
            self.log(self.tr("ended"))
            if self.current_username:
                today = datetime.datetime.now().strftime("%Y%m%d")
                path = os.path.abspath(os.path.join(LOG_DIR, f"{safe_filename(self.current_username)}_{today}.log"))
                self.log(self.tr("log_saved_auto").format(path=path))

    def _wait_interval(self, interval):
        """辅助函数：间隔等待与进度条更新"""
        total = max(1, int(interval * 10))
        for i in range(total):
            if self.stop_event.is_set():
                break
            val = (i + 1) / total * 100
            remain = interval - i * 0.1
            self.root.after(0, lambda v=val: self.progress.config(value=v))
            self.root.after(0, lambda r=remain: self.status_label.config(
                text=self.tr("status_next").format(sec=r)))
            time.sleep(0.1)
        self.root.after(0, lambda: self.progress.config(value=0))


# ---------- 入口 ----------
if __name__ == "__main__":
    if SYSTEM == "Windows":
        try:
            ctypes.windll.shcore.SetProcessDpiAwareness(1)
        except Exception:
            pass

    cfg = load_config()
    lang = cfg.get("lang", "zh")

    # 平台确认
    if not cfg.get("platform"):
        temp = tb.Window(themename="flatly")
        temp.withdraw()
        center_window(temp, 1, 1)
        chosen = confirm_platform(temp, lang)
        temp.destroy()
        cfg["platform"] = chosen
        save_config(cfg)
    else:
        chosen = cfg["platform"]

    conf = PLATFORM_CONF.get(chosen, PLATFORM_CONF["Windows"])
    initial_theme = conf["theme_dark"] if detect_system_theme() == "dark" else conf["theme_light"]

    # 用户协议：独立临时窗口
    if not cfg.get("agreed", False):
        temp2 = tb.Window(themename=initial_theme)
        temp2.withdraw()
        center_window(temp2, 1, 1)
        ok = show_agreement(temp2, lang, conf["font"])
        temp2.destroy()
        if ok:
            cfg["agreed"] = True
            save_config(cfg)
        else:
            sys.exit(0)

    root = tb.Window(themename=initial_theme)
    root.geometry("900x680")
    center_window(root, 900, 680)
    app = CourseMonitorApp(root, chosen, conf["font"])
    root.mainloop()
```
This is the first beta version,if you have good suggestions,please feel free to contact me at 3599287137@qq.com

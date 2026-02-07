# 🎯 文件码机器人 - 完整VIP版（跳转页面支付版）【新增备注功能】
# ==================== 导入部分 ====================
import telebot
import random
import sqlite3
import os
import time
import threading
import math
import re
import calendar
import base64
import json  # <-- 添加这行
# ==================== 【新增】E盘日志系统 ====================
from datetime import datetime, timedelta

class E盘日志:
    """专门保存到E盘的日志系统"""
    
    def __init__(self):
        # 🎯 指定E盘目录
        self.e盘路径 = r"E:\文件码机器人日志"
        
        # 尝试创建目录
        try:
            os.makedirs(self.e盘路径, exist_ok=True)
            print(f"✅ E盘日志系统已启动")
            print(f"📁 日志将保存到: {self.e盘路径}")
            
            # 测试是否能写入
            测试文件 = os.path.join(self.e盘路径, "测试.txt")
            with open(测试文件, "w", encoding="utf-8") as f:
                f.write("日志系统测试成功！\n")
            os.remove(测试文件)
            
        except Exception as e:
            print(f"❌ E盘不可用: {e}")
            print("💡 将使用当前目录")
            self.e盘路径 = "logs"  # 备用方案
            os.makedirs(self.e盘路径, exist_ok=True)
    
    def 记录文件包(self, 用户id, 文件码, 文件数量, 文件类型):
        """记录文件包信息到E盘"""
        try:
            # 今天的日期
            今天 = datetime.now().strftime("%Y-%m-%d")
            
            # 日志文件名
            文件名 = f"文件包日志_{今天}.txt"
            完整路径 = os.path.join(self.e盘路径, 文件名)
            
            # 时间戳
            时间 = datetime.now().strftime("%H:%M:%S")
            
            # 日志内容
            日志内容 = f"[{时间}] 用户{用户id} 创建文件包: {文件码} 文件数: {文件数量} 类型: {文件类型}\n"
            
            # 写入文件
            with open(完整路径, "a", encoding="utf-8") as 文件:
                文件.write(日志内容)
            
            print(f"📝 已保存到E盘: {完整路径}")
            
        except Exception as e:
            print(f"⚠️  记录失败（不影响机器人运行）: {e}")

# 创建日志对象
e盘日志器 = E盘日志()
from collections import defaultdict
# ==================== 【新增】统一客服配置 ====================
SUPPORT_BOT_USERNAME = "SUPPORT_BOT_LINK"  # 你的客服机器人用户名
SUPPORT_BOT_LINK = f"https://t.me/{SUPPORT_BOT_USERNAME}"
SUPPORT_CONTACT_TEXT = f"📞 联系客服：@{SUPPORT_BOT_USERNAME}"
# ==================== 【新增】机器人信息配置 ====================
BOT_USERNAME = "yixiang1bot"  # 修改为您的机器人用户名
BOT_LINK = f"https://t.me/{BOT_USERNAME}"

# ==================== 系统监控依赖 ====================
try:
    import psutil
    PSUTIL_AVAILABLE = True
except ImportError:
    PSUTIL_AVAILABLE = False
    print("⚠️  psutil 模块未安装，系统监控功能将受限")
    print("💡 安装命令: pip install psutil")
# ==================== 【新增】代理服务器配置部分 ====================
import requests
import ssl
import urllib3
from telebot import apihelper
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
from urllib3.util.ssl_ import create_urllib3_context
import warnings

# 禁用SSL警告
warnings.filterwarnings("ignore", message="Unverified HTTPS request")
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

print("=" * 60)
print("🤖 文件码机器人 - 完整VIP版（代理集成版）【新增备注功能】")
print("=" * 60)

# ==================== 【重要】配置你的代理 ====================
# 请根据你的代理软件修改下面的配置（选择适合你的一款）：

# 🔹 选项1：使用Clash（默认端口50520） - 大多数用户用这个
PROXY_CONFIG = {
    'http': 'http://127.0.0.1:50520',
    'https': 'http://127.0.0.1:50520'
}

# 🔹 选项2：使用V2Ray/SOCKS5代理（端口1080）
# PROXY_CONFIG = {
#     'http': 'socks5://127.0.0.1:1080',
#     'https': 'socks5://127.0.0.1:1080'
# }

# 🔹 选项3：使用V2RayN（端口10808）
# PROXY_CONFIG = {
#     'http': 'socks5://127.0.0.1:10808',
#     'https': 'socks5://127.0.0.1:10808'
# }

# 🔹 选项4：如果你不使用代理，用这个（但可能无法连接Telegram）
# PROXY_CONFIG = None

# ==================== 代理连接设置 ====================
class SSLAdapter(HTTPAdapter):
    """自定义SSL适配器解决连接问题"""
    def init_poolmanager(self, *args, **kwargs):
        ctx = create_urllib3_context()
        ctx.check_hostname = False
        ctx.verify_mode = ssl.CERT_NONE
        kwargs['ssl_context'] = ctx
        return super().init_poolmanager(*args, **kwargs)

def setup_telegram_connection():
    """配置Telegram API连接"""
    print("🔗 正在配置网络连接...")
    
    # 1. 创建自定义session
    session = requests.Session()
    
    # 2. 配置代理
    if PROXY_CONFIG:
        proxy_url = list(PROXY_CONFIG.values())[0]
        print(f"🌐 使用代理服务器: {proxy_url}")
        session.proxies.update(PROXY_CONFIG)
        apihelper.proxy = PROXY_CONFIG
    else:
        print("⚠️  未配置代理，直连模式（可能无法连接Telegram）")
    
    # 3. 配置SSL/TLS - 解决SSL错误
    session.verify = False
    session.mount('https://', SSLAdapter())
    
    # 4. 配置重试策略 - 增强稳定性
    retry_strategy = Retry(
        total=5,  # 最大重试次数
        backoff_factor=2,  # 重试间隔：1, 2, 4, 8, 16秒
        status_forcelist=[429, 500, 502, 503, 504],
        allowed_methods=["GET", "POST", "DELETE", "PUT"]
    )
    
    adapter = HTTPAdapter(
        max_retries=retry_strategy,
        pool_connections=100,
        pool_maxsize=100,
        pool_block=False
    )
    
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    
    # 5. 设置超时
    session.timeout = 30
    
    # 6. 应用到Telebot
    apihelper.REQUEST_SESSION = session
    apihelper.CERT_VERIFY = False  # 禁用SSL验证
    
    # 7. 设置API超时时间
    apihelper.READ_TIMEOUT = 30
    apihelper.CONNECT_TIMEOUT = 30
    
    print("✅ 网络连接配置完成")

# 执行连接配置
try:
    setup_telegram_connection()
    print("🔄 代理配置已应用")
except Exception as e:
    print(f"⚠️  代理配置失败: {e}")
    print("⚠️  将尝试无代理运行...")

# ==================== 【继续你的原代码配置】====================
# ⚠️ 需要你修改的配置 ⚠️
TOKEN = "8222876901:AAEA1VNNV4HprgxY13G7-HDXoCbmNL7ZVrg"  # 你的机器人Token
ADMIN_USER_IDS = [7654524960]  # 替换为你的Telegram用户ID
ADMIN_NOTIFY_CHAT_ID = 7654524960  # 替换为你的Telegram用户ID（接收通知）

# ==================== 支付页面配置 ====================
PAYMENT_BASE_URL = "sparkly-kleicha-be03f1.netlify.app"  # 修改为你的HTML页面地址

# 创建bot实例（使用代理配置）
bot = telebot.TeleBot(TOKEN)

# ==================== 【从这里开始是你原来的所有代码，只添加备注功能】====================
# ==================== 用户限制配置 ====================
class UserLimitConfig:
    """用户限制配置"""
    
    # 普通用户限制
    NORMAL_USER = {
        "daily_decode_limit": 50,          # 每日解码50次
        "daily_file_receive_limit": 50,    # 每日接收50个文件
        "next_page_delay": 5,              # 下一页点击间隔5秒
        "max_files_per_pack": 50,          # 每包最多文件数
        "max_file_size_mb": 20,            # 单个文件最大20MB
        "ads_enabled": True,               # 显示广告
    }
    
    # VIP用户特权（无限制）
    VIP_USER = {
        "daily_decode_limit": 99999,       # 无限解码
        "daily_file_receive_limit": 99999, # 无限接收文件
        "next_page_delay": 0,              # 无点击间隔限制
        "max_files_per_pack": 500,         # 每包最多500个文件
        "max_file_size_mb": 500,           # 单个文件最大500MB
        "ads_enabled": False,              # 无广告
    }

# ==================== VIP套餐配置 ====================
class VIPPackageConfig:
    """VIP套餐配置"""
    
    # 支付方式
    PAYMENT_METHODS = {
        "wechat": {"name": "微信支付", "icon": "💳"},
        "alipay": {"name": "支付宝", "icon": "💳"},
        "usdt": {"name": "USDT", "icon": "₿"}
    }
# 【新增】获取启用的支付方式
    def get_payment_methods(self):
        """获取启用的支付方式"""
        methods = payment_method_manager.get_enabled_methods()
        
        icons = {
            "wechat": "💳",
            "alipay": "💳",
            "usdt": "₿"
        }
        
        result = {}
        for method in methods:
            method_id = method['method_id']
            result[method_id] = {
                "name": method['method_name'],
                "icon": icons.get(method_id, "💳")
            }
        
        return result
    
    # 【新增】获取所有支付方式信息
    def get_all_payment_methods_info(self):
        """获取所有支付方式信息（包括禁用的）"""
        methods = payment_method_manager.get_all_methods()
        
        icons = {
            "wechat": "💳",
            "alipay": "💳",
            "usdt": "₿"
        }
        
        result = {}
        for method in methods:
            method_id = method['method_id']
            result[method_id] = {
                "name": method['method_name'],
                "icon": icons.get(method_id, "💳"),
                "is_enabled": method['is_enabled'],
                "display_order": method['display_order']
            }
        
        return result

    # 默认套餐（后台可修改）
    DEFAULT_PACKAGES = {
        "monthly": {
            "name": "20元包月VIP",
            "price_cny": 20.0,
            "months": 1,
            "description": "适合短期使用，灵活方便"
        },
        "quarterly": {
            "name": "60元包季VIP",
            "price_cny": 60.0,
            "months": 3,
            "description": "节省10%，性价比之选"
        },
        "yearly": {
            "name": "240元包年VIP",
            "price_cny": 240.0,
            "months": 12,
            "description": "节省20%，最划算选择"
        }
    }
# ==================== 【新增】套餐管理方法 ====================
    
    def get_all_packages(self):
        """获取所有套餐"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
        SELECT id, name, price_cny, months, description, is_active, display_order
        FROM vip_packages
        ORDER BY display_order
        ''')
        
        packages = cursor.fetchall()
        
        result = []
        for pkg in packages:
            result.append({
                'id': pkg[0],
                'name': pkg[1],
                'price_cny': pkg[2],
                'months': pkg[3],
                'description': pkg[4],
                'is_active': bool(pkg[5]),
                'display_order': pkg[6]
            })
        
        return result
    
    def update_package_price(self, package_id, new_price):
        """修改套餐价格"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute('''
            UPDATE vip_packages 
            SET price_cny = ?, updated_at = ?
            WHERE id = ?
            ''', (new_price, datetime.now().isoformat(), package_id))
            
            conn.commit()
            return True, "✅ 套餐价格修改成功"
        except Exception as e:
            conn.rollback()
            return False, f"❌ 修改失败: {e}"
    
    def update_package_name(self, package_id, new_name):
        """修改套餐名称"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute('''
            UPDATE vip_packages 
            SET name = ?, updated_at = ?
            WHERE id = ?
            ''', (new_name, datetime.now().isoformat(), package_id))
            
            conn.commit()
            return True, "✅ 套餐名称修改成功"
        except Exception as e:
            conn.rollback()
            return False, f"❌ 修改失败: {e}"
    
    def update_package_months(self, package_id, new_months):
        """修改套餐时长"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute('''
            UPDATE vip_packages 
            SET months = ?, updated_at = ?
            WHERE id = ?
            ''', (new_months, datetime.now().isoformat(), package_id))
            
            conn.commit()
            return True, "✅ 套餐时长修改成功"
        except Exception as e:
            conn.rollback()
            return False, f"❌ 修改失败: {e}"
    
    def update_package_description(self, package_id, new_description):
        """修改套餐描述"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute('''
            UPDATE vip_packages 
            SET description = ?, updated_at = ?
            WHERE id = ?
            ''', (new_description, datetime.now().isoformat(), package_id))
            
            conn.commit()
            return True, "✅ 套餐描述修改成功"
        except Exception as e:
            conn.rollback()
            return False, f"❌ 修改失败: {e}"
    
    def update_package_order(self, package_id, new_order):
        """修改套餐排序"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute('''
            UPDATE vip_packages 
            SET display_order = ?, updated_at = ?
            WHERE id = ?
            ''', (new_order, datetime.now().isoformat(), package_id))
            
            conn.commit()
            return True, "✅ 套餐排序修改成功"
        except Exception as e:
            conn.rollback()
            return False, f"❌ 修改失败: {e}"
    
    def toggle_package_status(self, package_id):
        """切换套餐激活状态"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute('SELECT is_active FROM vip_packages WHERE id = ?', (package_id,))
            current_status = cursor.fetchone()
            
            if not current_status:
                return False, "❌ 套餐不存在"
            
            new_status = not bool(current_status[0])
            
            cursor.execute('''
            UPDATE vip_packages 
            SET is_active = ?, updated_at = ?
            WHERE id = ?
            ''', (new_status, datetime.now().isoformat(), package_id))
            
            conn.commit()
            
            status_text = "激活" if new_status else "禁用"
            return True, f"✅ 套餐已{status_text}"
        except Exception as e:
            conn.rollback()
            return False, f"❌ 修改失败: {e}"
class PaymentMethodManager:
    """支付方式管理器"""
    
    def get_all_methods(self):
        """获取所有支付方式"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT method_id, method_name, is_enabled, display_order FROM payment_methods_status ORDER BY display_order')
        methods = cursor.fetchall()
        
        result = []
        for method in methods:
            result.append({
                'method_id': method[0],
                'method_name': method[1],
                'is_enabled': bool(method[2]),
                'display_order': method[3]
            })
        
        return result
    
    def get_enabled_methods(self):
        """获取已启用的支付方式"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('SELECT method_id, method_name, display_order FROM payment_methods_status WHERE is_enabled = 1 ORDER BY display_order')
        methods = cursor.fetchall()
        
        result = []
        for method in methods:
            result.append({
                'method_id': method[0],
                'method_name': method[1],
                'display_order': method[2]
            })
        
        return result
    
    def toggle_method_status(self, method_id):
        """切换支付方式状态（启用/禁用）"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute('SELECT is_enabled FROM payment_methods_status WHERE method_id = ?', (method_id,))
            current_status = cursor.fetchone()
            
            if not current_status:
                return False, "❌ 支付方式不存在"
            
            new_status = not bool(current_status[0])
            
            cursor.execute('UPDATE payment_methods_status SET is_enabled = ?, updated_at = ? WHERE method_id = ?', 
                          (new_status, datetime.now().isoformat(), method_id))
            
            conn.commit()
            
            status_text = "启用" if new_status else "禁用"
            return True, f"✅ 支付方式已{status_text}"
        except Exception as e:
            conn.rollback()
            return False, f"❌ 修改失败: {e}"
    
    def update_method_name(self, method_id, new_name):
        """修改支付方式名称"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute('UPDATE payment_methods_status SET method_name = ?, updated_at = ? WHERE method_id = ?', 
                          (new_name, datetime.now().isoformat(), method_id))
            
            conn.commit()
            return True, "✅ 支付方式名称修改成功"
        except Exception as e:
            conn.rollback()
            return False, f"❌ 修改失败: {e}"
    
    def update_method_order(self, method_id, new_order):
        """修改支付方式排序"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute('UPDATE payment_methods_status SET display_order = ?, updated_at = ? WHERE method_id = ?', 
                          (new_order, datetime.now().isoformat(), method_id))
            
            conn.commit()
            return True, "✅ 支付方式排序修改成功"
        except Exception as e:
            conn.rollback()
            return False, f"❌ 修改失败: {e}"

# 创建实例
payment_method_manager = PaymentMethodManager()
# ==================== 数据库连接池 ====================
class SimpleConnectionPool:
    def __init__(self, db_path, pool_size=300):  # 🟢 修改1：增加到300
        self.db_path = db_path
        self.pool_size = pool_size
        self.connections = []
        self.lock = threading.Lock()
        self.current = 0
        
        print(f"💾 创建数据库连接池 ({pool_size}个连接)")
        print(f"⚡ SQLite优化参数:")
        print(f"   - WAL模式: 已启用")
        print(f"   - 缓存大小: 256MB")
        print(f"   - 内存映射: 1GB")
        print(f"   - 页大小: 16KB")
        
        for _ in range(pool_size):
            conn = sqlite3.connect(db_path, check_same_thread=False)
            # 🟢 修改2：优化SQLite性能参数
            conn.execute('PRAGMA journal_mode=WAL')
            conn.execute('PRAGMA synchronous=NORMAL')  # 安全和性能的平衡
            conn.execute('PRAGMA cache_size=-262144')   # 256MB缓存（原2MB）
            conn.execute('PRAGMA mmap_size=1073741824') # 1GB内存映射
            conn.execute('PRAGMA page_size=16384')      # 16KB页大小（原默认4KB）
            conn.execute('PRAGMA temp_store=MEMORY')    # 临时表存内存
            conn.execute('PRAGMA busy_timeout=30000')   # 30秒超时（原5秒）
            conn.execute('PRAGMA wal_autocheckpoint=100') # WAL自动检查点
            conn.execute('PRAGMA foreign_keys=ON')       # 外键约束
            
            # 立即应用设置
            conn.commit()
            self.connections.append(conn)
        
        print(f"✅ 数据库连接池已创建 ({pool_size}个连接)")
    
    def get_connection(self):
        """获取一个数据库连接"""
        with self.lock:
            conn = self.connections[self.current]
            self.current = (self.current + 1) % self.pool_size
            return conn
    
    def close_all(self):
        """关闭所有连接"""
        for conn in self.connections:
            conn.close()

# ==================== 主菜单函数 ====================
def create_main_menu():
    """创建主菜单"""
    markup = telebot.types.ReplyKeyboardMarkup(
        resize_keyboard=True,
        row_width=2
    )
    markup.row("📖 使用说明", "📜 用户协议")
    markup.row("📊 当前状态", "📁 我的代码")
    markup.row("⭐ VIP服务", "👤 我的账户")
    return markup

# ==================== 数据库初始化 ====================
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DB_PATH = os.path.join(BASE_DIR, "files_vip_final.db")
db_pool = SimpleConnectionPool(DB_PATH, pool_size=180)

def init_database():
    """初始化数据库"""
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 原有的文件相关表 - 【修改】添加decode_count字段
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS packs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        code TEXT UNIQUE NOT NULL,
        user_id INTEGER NOT NULL,
        file_count INTEGER NOT NULL,
        file_types TEXT NOT NULL,
        decode_count INTEGER DEFAULT 0,  -- 【新增】解码次数统计
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
    ''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS files (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        pack_id INTEGER NOT NULL,
        file_id TEXT NOT NULL,
        file_name TEXT NOT NULL,
        file_type TEXT NOT NULL,
        FOREIGN KEY (pack_id) REFERENCES packs (id) ON DELETE CASCADE
    )
    ''')
    
    # VIP相关表
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS daily_stats (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        date DATE NOT NULL,
        decode_count INTEGER DEFAULT 0,
        file_receive_count INTEGER DEFAULT 0,
        video_count INTEGER DEFAULT 0,
        photo_count INTEGER DEFAULT 0,
        document_count INTEGER DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        UNIQUE(user_id, date)
    )
    ''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS vip_users (
        user_id INTEGER PRIMARY KEY,
        is_vip BOOLEAN DEFAULT FALSE,
        vip_expire TIMESTAMP,
        total_spent REAL DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
    ''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS vip_packages (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        price_cny REAL NOT NULL,
        months INTEGER NOT NULL,
        description TEXT,
        is_active BOOLEAN DEFAULT TRUE,
        display_order INTEGER DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
    ''')
    
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS payment_orders (
        order_id TEXT PRIMARY KEY,
        user_id INTEGER NOT NULL,
        package_id INTEGER NOT NULL,
        amount REAL NOT NULL,
        payment_method TEXT NOT NULL,  -- wechat/alipay/usdt
        status TEXT DEFAULT 'pending',  -- pending/paid/cancelled/activated
        qr_shown BOOLEAN DEFAULT FALSE,
        user_confirmed BOOLEAN DEFAULT FALSE,
        admin_notified BOOLEAN DEFAULT FALSE,
        activated BOOLEAN DEFAULT FALSE,
        payment_time TIMESTAMP,
        activated_time TIMESTAMP,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
    ''')
    
    # ========== 【新增】文件备注表 ==========
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS pack_remarks (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        pack_id INTEGER NOT NULL,
        user_id INTEGER NOT NULL,
        remark TEXT NOT NULL,
        tags TEXT,  -- 标签，用逗号分隔
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        UNIQUE(pack_id, user_id),
        FOREIGN KEY (pack_id) REFERENCES packs (id) ON DELETE CASCADE
    )
    ''')
        # 支付方式状态表（新增）
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS payment_methods_status (
        method_id TEXT PRIMARY KEY,
        method_name TEXT NOT NULL,
        is_enabled BOOLEAN DEFAULT TRUE,
        display_order INTEGER DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
    ''')
    
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_method_enabled ON payment_methods_status(is_enabled)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_method_order ON payment_methods_status(display_order)')
    
    # 初始化默认支付方式
    cursor.execute('SELECT COUNT(*) FROM payment_methods_status')
    if cursor.fetchone()[0] == 0:
        default_methods = [
            ('wechat', '微信支付', 1, 1),
            ('alipay', '支付宝', 1, 2),
            ('usdt', 'USDT', 1, 3)
        ]
        
        for method_id, method_name, is_enabled, order in default_methods:
            cursor.execute('''
            INSERT OR IGNORE INTO payment_methods_status (method_id, method_name, is_enabled, display_order)
            VALUES (?, ?, ?, ?)
            ''', (method_id, method_name, is_enabled, order))
    
    # 创建索引
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_user_id ON packs(user_id)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_code ON packs(code)')
    # 🟢 新增优化索引
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_packs_decode ON packs(decode_count)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_packs_created ON packs(created_at)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_packs_user_date ON packs(user_id, created_at)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_files_pack ON files(pack_id, file_type)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_daily_user_date ON daily_stats(user_id, date DESC)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_payment_user ON payment_orders(user_id, created_at DESC)')
    
    # 分析表以优化查询计划
    cursor.execute('ANALYZE')
    
    print("✅ 数据库索引优化完成")
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_pack_id ON files(pack_id)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_daily_user ON daily_stats(user_id)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_daily_date ON daily_stats(date)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_vip_expire ON vip_users(vip_expire)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_order_status ON payment_orders(status)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_order_user ON payment_orders(user_id)')
    
    # 【新增】备注表索引
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_remark_user ON pack_remarks(user_id)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_remark_pack ON pack_remarks(pack_id)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_remark_text ON pack_remarks(remark)')
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_remark_tags ON pack_remarks(tags)')
    
    # 初始化默认VIP套餐
    cursor.execute('SELECT COUNT(*) FROM vip_packages')
    if cursor.fetchone()[0] == 0:
        default_packages = [
            ('20元包月VIP', 20.0, 1, '适合短期使用，灵活方便', 1),
            ('60元包季VIP', 60.0, 3, '节省10%，性价比之选', 2),
            ('240元包年VIP', 240.0, 12, '节省20%，最划算选择', 3)
        ]
        
        for name, price, months, desc, order in default_packages:
            cursor.execute('''
            INSERT INTO vip_packages (name, price_cny, months, description, display_order)
            VALUES (?, ?, ?, ?, ?)
            ''', (name, price, months, desc, order))
    
    conn.commit()
    
    # 【新增】为已存在的packs表添加decode_count列
    try:
        cursor.execute('ALTER TABLE packs ADD COLUMN decode_count INTEGER DEFAULT 0')
        print("✅ 已添加decode_count列到packs表")
    except sqlite3.OperationalError as e:
        if "duplicate column name" not in str(e):
            print(f"⚠️  添加列可能已存在: {e}")
    
    # 添加解码次数索引
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_decode_count ON packs(decode_count)')
    
    conn.commit()
    print("✅ 数据库初始化完成")
# ==================== 数据库结构检查 ====================
def check_database_tables():
    """检查数据库表结构"""
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    print("\n" + "="*60)
    print("🔍 检查数据库表结构")
    print("="*60)
    
    # 获取所有表名
    cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
    tables = cursor.fetchall()
    
    print(f"📋 共找到 {len(tables)} 个表:")
    for table in tables:
        table_name = table[0]
        print(f"\n📄 表名: {table_name}")
        
        # 获取表结构
        cursor.execute(f"PRAGMA table_info({table_name})")
        columns = cursor.fetchall()
        
        print(f"  列名:")
        for col in columns:
            col_id, col_name, col_type, not_null, default_val, pk = col
            print(f"    {col_id}. {col_name} ({col_type}) {'PRIMARY KEY' if pk else ''}")
    
    conn.close()
    print("\n" + "="*60)
    print("✅ 数据库检查完成")
    print("="*60 + "\n")

# 在数据库初始化后调用
check_database_tables()
init_database()

# ==================== 用户限制管理器 ====================
class UserLimitManager:
    """用户限制管理器"""
    
    def __init__(self):
        self.cache = {}
        self.cache_timeout = 300
    
    def is_vip(self, user_id):
        """检查用户是否是VIP"""
        cache_key = f"vip_{user_id}"
        if cache_key in self.cache:
            cached_data, cache_time = self.cache[cache_key]
            if time.time() - cache_time < self.cache_timeout:
                return cached_data
        
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
        SELECT is_vip, vip_expire FROM vip_users WHERE user_id = ?
        ''', (user_id,))
        
        row = cursor.fetchone()
        
        is_vip = False
        if row and row[0]:
            if row[1]:  # 有到期时间
                expire_time = datetime.fromisoformat(row[1])
                if datetime.now() < expire_time:
                    is_vip = True
            else:  # 永久VIP
                is_vip = True
        
        # 更新缓存
        self.cache[cache_key] = (is_vip, time.time())
        
        return is_vip
    
    def check_decode_limit(self, user_id):
        """检查解码限制"""
        if self.is_vip(user_id):
            return True, ""  # VIP用户无限制
        
        daily_stats = self.get_daily_stats(user_id)
        limit = UserLimitConfig.NORMAL_USER["daily_decode_limit"]
        
        if daily_stats["decode_count"] >= limit:
            return False, f"❌ 已达到普通用户限制（{limit}次/天），等明日再来"
        
        return True, ""
    
    def check_file_receive_limit(self, user_id, file_count=1):
        """检查文件接收限制"""
        if self.is_vip(user_id):
            return True, ""  # VIP用户无限制
        
        daily_stats = self.get_daily_stats(user_id)
        limit = UserLimitConfig.NORMAL_USER["daily_file_receive_limit"]
        current = daily_stats["file_receive_count"]
        
        if current >= limit:
            return False, f"❌ 普通用户接收文件已上限（{limit}个/天），等明日再来"
        
        if current + file_count > limit:
            remaining = limit - current
            return False, f"❌ 普通用户每日最多接收{limit}个文件，今日还可接收{remaining}个"
        
        return True, ""
    
    def get_next_page_delay(self, user_id):
        """获取下一页点击延迟时间"""
        if self.is_vip(user_id):
            return 0  # VIP用户无延迟
        return UserLimitConfig.NORMAL_USER["next_page_delay"]
    
    def get_daily_stats(self, user_id):
        """获取用户今日统计"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        today = datetime.now().strftime('%Y-%m-%d')
        
        cursor.execute('''
        SELECT decode_count, file_receive_count, video_count, photo_count, document_count
        FROM daily_stats
        WHERE user_id = ? AND date = ?
        ''', (user_id, today))
        
        row = cursor.fetchone()
        
        if row:
            return {
                "decode_count": row[0] or 0,
                "file_receive_count": row[1] or 0,
                "video_count": row[2] or 0,
                "photo_count": row[3] or 0,
                "document_count": row[4] or 0
            }
        
        # 如果没有记录，返回默认值
        return {
            "decode_count": 0,
            "file_receive_count": 0,
            "video_count": 0,
            "photo_count": 0,
            "document_count": 0
        }
    
    def increment_decode_count(self, user_id):
        """增加解码计数"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        today = datetime.now().strftime('%Y-%m-%d')
        
        cursor.execute('''
        INSERT INTO daily_stats (user_id, date, decode_count)
        VALUES (?, ?, 1)
        ON CONFLICT(user_id, date) 
        DO UPDATE SET decode_count = decode_count + 1
        ''', (user_id, today))
        
        conn.commit()
    
    def increment_file_receive_count(self, user_id, file_count=1, file_type=None):
        """增加文件接收计数"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        today = datetime.now().strftime('%Y-%m-%d')
        
        # 更新总文件数
        cursor.execute('''
        INSERT INTO daily_stats (user_id, date, file_receive_count)
        VALUES (?, ?, ?)
        ON CONFLICT(user_id, date) 
        DO UPDATE SET file_receive_count = file_receive_count + ?
        ''', (user_id, today, file_count, file_count))
        
        # 更新具体类型计数
        if file_type:
            column = f"{file_type}_count"
            try:
                cursor.execute(f'''
                INSERT INTO daily_stats (user_id, date, {column})
                VALUES (?, ?, ?)
                ON CONFLICT(user_id, date) 
                DO UPDATE SET {column} = {column} + ?
                ''', (user_id, today, file_count, file_count))
            except:
                # 如果列不存在，先添加列
                cursor.execute(f'ALTER TABLE daily_stats ADD COLUMN {column} INTEGER DEFAULT 0')
                conn.commit()
                cursor.execute(f'''
                INSERT INTO daily_stats (user_id, date, {column})
                VALUES (?, ?, ?)
                ON CONFLICT(user_id, date) 
                DO UPDATE SET {column} = {column} + ?
                ''', (user_id, today, file_count, file_count))
        
        conn.commit()

user_limit_manager = UserLimitManager()

# ==================== 【新增】文件备注管理器 ====================
class PackRemarkManager:
    """文件包备注管理器"""
    
    def add_remark(self, user_id, code, remark, tags=""):
        """为文件码添加备注"""
        conn = None
        cursor = None
        try:
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            # 获取pack_id
            cursor.execute('SELECT id FROM packs WHERE code = ? AND user_id = ?', (code, user_id))
            pack_row = cursor.fetchone()
            
            if not pack_row:
                return False, "❌ 文件码不存在或不属于您"
            
            pack_id = pack_row[0]
            
            # 检查备注长度
            remark = remark.strip()
            if len(remark) > 200:
                return False, "❌ 备注太长（最多200字）"
            
            # 清理标签格式
            if tags:
                tags = ",".join([tag.strip() for tag in tags.split(",") if tag.strip()])
                if len(tags) > 100:
                    return False, "❌ 标签太长（最多100字）"
            
            # 添加或更新备注
            cursor.execute('''
            INSERT OR REPLACE INTO pack_remarks (pack_id, user_id, remark, tags, updated_at)
            VALUES (?, ?, ?, ?, ?)
            ''', (pack_id, user_id, remark, tags, datetime.now().isoformat()))
            
            conn.commit()
            return True, "✅ 备注添加成功"
            
        except Exception as e:
            print(f"❌ 添加备注失败: {e}")
            if conn:
                conn.rollback()
            return False, f"❌ 添加备注失败: {e}"
        finally:
            if cursor:
                cursor.close()
    
    def get_remark(self, user_id, code):
        """获取文件码的备注"""
        conn = None
        cursor = None
        try:
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            cursor.execute('''
            SELECT pr.remark, pr.tags, pr.created_at, pr.updated_at
            FROM pack_remarks pr
            JOIN packs p ON pr.pack_id = p.id
            WHERE p.code = ? AND pr.user_id = ?
            ''', (code, user_id))
            
            row = cursor.fetchone()
            
            if row:
                return {
                    "remark": row[0],
                    "tags": row[1],
                    "created_at": row[2],
                    "updated_at": row[3]
                }
            return None
            
        except Exception as e:
            print(f"❌ 获取备注失败: {e}")
            return None
        finally:
            if cursor:
                cursor.close()
    
    def search_user_remarks(self, user_id, keyword):
        """用户搜索自己的备注"""
        conn = None
        cursor = None
        try:
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            # 在备注和标签中搜索关键词
            search_pattern = f"%{keyword}%"
            
            cursor.execute('''
            SELECT p.code, pr.remark, pr.tags, p.file_count, p.file_types, p.created_at
            FROM pack_remarks pr
            JOIN packs p ON pr.pack_id = p.id
            WHERE pr.user_id = ? 
            AND (pr.remark LIKE ? OR pr.tags LIKE ?)
            ORDER BY pr.updated_at DESC
            LIMIT 50
            ''', (user_id, search_pattern, search_pattern))
            
            results = []
            for row in cursor.fetchall():
                results.append({
                    "code": row[0],
                    "remark": row[1],
                    "tags": row[2],
                    "file_count": row[3],
                    "file_types": row[4],
                    "created_at": row[5]
                })
            
            return results
            
        except Exception as e:
            print(f"❌ 搜索备注失败: {e}")
            return []
        finally:
            if cursor:
                cursor.close()
    
    def search_all_remarks_admin(self, keyword):
        """管理员搜索所有用户的备注"""
        conn = None
        cursor = None
        try:
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            search_pattern = f"%{keyword}%"
            
            cursor.execute('''
            SELECT p.code, pr.remark, pr.tags, p.user_id, 
                   p.file_count, p.file_types, p.created_at
            FROM pack_remarks pr
            JOIN packs p ON pr.pack_id = p.id
            WHERE pr.remark LIKE ? OR pr.tags LIKE ?
            ORDER BY pr.updated_at DESC
            LIMIT 100
            ''', (search_pattern, search_pattern))
            
            results = []
            for row in cursor.fetchall():
                results.append({
                    "code": row[0],
                    "remark": row[1],
                    "tags": row[2],
                    "user_id": row[3],
                    "file_count": row[4],
                    "file_types": row[5],
                    "created_at": row[6]
                })
            
            return results
            
        except Exception as e:
            print(f"❌ 管理员搜索备注失败: {e}")
            return []
        finally:
            if cursor:
                cursor.close()
    
    def delete_remark(self, user_id, code):
        """删除文件码备注"""
        conn = None
        cursor = None
        try:
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            cursor.execute('''
            DELETE FROM pack_remarks
            WHERE pack_id IN (
                SELECT id FROM packs WHERE code = ? AND user_id = ?
            )
            ''', (code, user_id))
            
            conn.commit()
            return cursor.rowcount > 0
            
        except Exception as e:
            print(f"❌ 删除备注失败: {e}")
            if conn:
                conn.rollback()
            return False
        finally:
            if cursor:
                cursor.close()

# ==================== 【新增】文件码解码次数管理器 ====================
class PackDecodeManager:
    """文件码解码次数管理器"""
    
    def increment_decode_count(self, code):
        """增加文件码的解码次数"""
        conn = None
        cursor = None
        try:
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            cursor.execute('''
            UPDATE packs 
            SET decode_count = decode_count + 1
            WHERE code = ?
            ''', (code,))
            
            conn.commit()
            return True
        except Exception as e:
            print(f"❌ 更新解码次数失败: {e}")
            if conn:
                conn.rollback()
            return False
        finally:
            if cursor:
                cursor.close()
    def get_decode_count(self, code):
        """获取文件码的解码次数"""
        conn = None
        cursor = None
        try:
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            cursor.execute('SELECT decode_count FROM packs WHERE code = ?', (code,))
            row = cursor.fetchone()
            
            return row[0] if row else 0
        except Exception as e:
            print(f"❌ 获取解码次数失败: {e}")
            return 0
        finally:
            if cursor:
                cursor.close()
    def increment_decode_count_with_cache(self, code):
        """【新增】使用缓存的解码次数增加方法"""
        # 🟢 使用内存缓存，不直接写数据库
        decode_cache.increment(code)
        return True
    def get_decode_count_with_cache(self, code):
        """【新增】使用缓存的解码次数获取方法"""
        return decode_cache.get_count(code)
    
    def get_user_packs_with_stats(self, user_id):
        """获取用户的所有文件包（包含解码次数）"""
        conn = None
        cursor = None
        try:
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            cursor.execute('''
            SELECT code, user_id, file_count, file_types, decode_count, created_at
            FROM packs
            WHERE user_id = ?
            ORDER BY created_at DESC
            LIMIT 50
            ''', (user_id,))
            
            rows = cursor.fetchall()
            
            packs = []
            for row in rows:
                packs.append({
                    'code': row[0],
                    'user_id': row[1],
                    'file_count': row[2],
                    'file_types': row[3],
                    'decode_count': row[4] or 0,  # 解码次数
                    'created_at': row[5]
                })
            
            return packs
            
        except Exception as e:
            print(f"❌ 获取用户文件包失败: {e}")
            return []
        finally:
            if cursor:
                cursor.close()

# 创建解码管理器实例
pack_decode_manager = PackDecodeManager()
# ==================== 【新增】解码次数内存缓存 ====================
class DecodeCountCache:
    """解码次数内存缓存，大幅减少数据库写操作"""
    
    def __init__(self, flush_interval=30):
        self.cache = {}
        self.lock = threading.Lock()
        self.flush_interval = flush_interval
        self.last_flush = time.time()
        self.total_cached = 0
        
        # 启动后台刷新线程
        threading.Thread(target=self._flush_worker, daemon=True).start()
        print(f"🔄 解码次数缓存已启动（刷新间隔：{flush_interval}秒）")
    
    def increment(self, code):
        """增加解码次数（内存操作，极快）"""
        with self.lock:
            self.cache[code] = self.cache.get(code, 0) + 1
            self.total_cached += 1
            
            # 如果缓存太大，立即刷新
            if len(self.cache) > 1000:
                self.flush()
    
    def get_count(self, code):
        """获取解码次数（先查缓存，再查数据库）"""
        # 先从缓存获取
        cache_count = 0
        with self.lock:
            cache_count = self.cache.get(code, 0)
        
        # 再从数据库获取
        db_count = pack_decode_manager.get_decode_count(code)  # 改为这个方法
        return db_count + cache_count
    
    def flush(self):
        """立即刷新到数据库"""
        with self.lock:
            if not self.cache:
                return 0
            
            conn = None
            cursor = None
            try:
                conn = db_pool.get_connection()
                cursor = conn.cursor()
                
                updated = 0
                total_increments = 0
                
                # 🟢 使用事务批量更新
                cursor.execute('BEGIN TRANSACTION')
                for code, count in self.cache.items():
                    cursor.execute('''
                    UPDATE packs 
                    SET decode_count = decode_count + ?
                    WHERE code = ?
                    ''', (count, code))
                    updated += 1
                    total_increments += count
                
                cursor.execute('COMMIT')
                conn.commit()
                
                if updated > 0:
                    print(f"💾 缓存刷新：{updated}个文件码，{total_increments}次解码")
                
                self.cache.clear()
                self.last_flush = time.time()
                return updated
                
            except Exception as e:
                print(f"❌ 缓存刷新失败: {e}")
                if conn:
                    try:
                        cursor.execute('ROLLBACK')
                        conn.rollback()
                    except:
                        pass
                return 0
            finally:
                if cursor:
                    cursor.close()
    
    def _flush_worker(self):
        """后台刷新线程"""
        while True:
            time.sleep(self.flush_interval)
            try:
                if self.cache:
                    self.flush()
            except Exception as e:
                print(f"❌ 缓存刷新线程错误: {e}")

# 创建缓存实例
decode_cache = DecodeCountCache(flush_interval=30)

# 创建实例
pack_remark_manager = PackRemarkManager()

# ==================== VIP支付系统（跳转页面版）====================
class VIPPaymentSystem:
    """VIP支付系统 - 跳转页面版本"""
    
    def __init__(self):
        self.pending_orders = {}
        self.package_manager = VIPPackageConfig()  # <-- 添加这行
    
    def show_vip_packages(self, user_id, chat_id):
        """显示VIP套餐"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
    
        cursor.execute('''
        SELECT id, name, price_cny, months, description 
        FROM vip_packages 
        WHERE is_active = 1 
        ORDER BY display_order
        ''')
    
        packages = cursor.fetchall()
    
        if not packages:
            bot.send_message(chat_id, "❌ 暂时没有可用的VIP套餐")
            return
        
        # 获取用户当前状态
        is_vip = user_limit_manager.is_vip(user_id)
        daily_stats = user_limit_manager.get_daily_stats(user_id)
    
        text = "💰 **VIP套餐选择**\n\n"
    
        if is_vip:
            text += "✅ **您已经是VIP用户**\n\n"
        else:
            text += "📊 **您今日使用情况：**\n"
            text += f"• 解码次数：{daily_stats['decode_count']}/50\n"
            text += f"• 接收文件：{daily_stats['file_receive_count']}/50\n"
            text += f"• VIP状态：普通用户\n\n"
    
        text += "请选择套餐：\n\n"
        # 显示套餐列表和描述
        for pkg in packages:
            pkg_id, name, price, months, desc = pkg
            text += f"**{name}** - {price}元\n"
            if desc:  # ✅ 添加描述显示
                text += f"📝 {desc}\n"
            text += f"⏰ {months}个月 | "
            # 计算日均价格
            daily_price = price / (months * 30)
            text += f"日均约{daily_price:.2f}元\n\n"

        markup = telebot.types.InlineKeyboardMarkup(row_width=1)
    
        for pkg in packages:
            pkg_id, name, price, months, desc = pkg
            button_text = f"{name} - {price}元"
            markup.add(
                telebot.types.InlineKeyboardButton(
                    button_text,
                    callback_data=f"vip_package_{pkg_id}"
                )
            )
        
        markup.add(
            telebot.types.InlineKeyboardButton("📋 套餐详情对比", callback_data="vip_compare"),
            telebot.types.InlineKeyboardButton("🏠 返回", callback_data="vip_center")
        )
    
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    def show_payment_methods(self, user_id, chat_id, package_id, call=None):
        """显示支付方式"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
    
        cursor.execute('SELECT name, price_cny, months, description FROM vip_packages WHERE id = ?', (package_id,))
        package = cursor.fetchone()
        
        if not package:
            if call:
                bot.answer_callback_query(call.id, "❌ 套餐不存在")
            return
    
        name, price, months, description = package
    
        text = f"""
    ✅ **已选择：{name}**

    💰 **价格：{price}元**
    📅 **时长：{months}个月**
    """
    
        if description:  # ✅ 添加描述显示
            text += f"📝 **描述：**{description}\n\n"
        else:
            text += "\n"
    
        text += "请选择支付方式："
        
        markup = telebot.types.InlineKeyboardMarkup(row_width=1)
    # 【修改这里】使用动态获取的支付方式
        payment_methods = self.package_manager.get_payment_methods()
        
        if not payment_methods:
            markup.add(
                telebot.types.InlineKeyboardButton(
                    "❌ 暂无可用支付方式",
                    callback_data="no_payment_methods"
                )
            )
        else:
            for method_id, method in payment_methods.items():
                markup.add(
                    telebot.types.InlineKeyboardButton(
                        f"{method['icon']} {method['name']}",
                        callback_data=f"vip_pay_{package_id}_{method_id}"
                    )
                )    
        
        markup.add(
            telebot.types.InlineKeyboardButton("⬅️ 重新选择套餐", callback_data="vip_packages"),
            telebot.types.InlineKeyboardButton("🏠 返回首页", callback_data="back_to_main")
        )
        
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    def create_payment_order(self, user_id, chat_id, package_id, method_id):
        """创建支付订单"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        # 获取套餐信息
        cursor.execute('SELECT name, price_cny, months FROM vip_packages WHERE id = ?', (package_id,))
        package = cursor.fetchone()
        
        if not package:
            return None
        
        name, price, months = package
        method_name = VIPPackageConfig.PAYMENT_METHODS[method_id]["name"]
        
        # 生成订单号
        order_id = f"VIP{user_id}_{int(time.time())}_{random.randint(1000, 9999)}"
        
        # 保存订单到数据库
        cursor.execute('''
        INSERT INTO payment_orders 
        (order_id, user_id, package_id, amount, payment_method, status)
        VALUES (?, ?, ?, ?, ?, 'pending')
        ''', (order_id, user_id, package_id, price, method_id))
        
        conn.commit()
        
        # 保存到内存缓存
        self.pending_orders[order_id] = {
            "user_id": user_id,
            "package_id": package_id,
            "method_id": method_id,
            "amount": price,
            "months": months,
            "created_at": time.time(),
            "status": "pending"
        }
        
        # 显示支付页面（跳转页面版）
        self.show_web_payment_page(user_id, chat_id, order_id, name, price, months, method_id, method_name)
        
        # 通知管理员
        self.notify_admin_new_order(user_id, order_id, name, price, months, method_name)
        
        return order_id
    
    def show_web_payment_page(self, user_id, chat_id, order_id, name, price, months, method_id, method_name):
        """显示跳转支付页面"""
        
        # 生成支付页面URL
        payment_url = self.get_payment_url(order_id, price, method_id, months)
        
        text = f"""
💰 **{method_name} - {name}**

📋 **订单信息**
• 订单号：`{order_id}`
• 金额：{price}{'元' if method_id != 'usdt' else ' USDT'}
• 时长：{months}个月
• 用户ID：`{user_id}`

🌐 **支付步骤**
1. 点击下方"前往支付页面"按钮
2. 在浏览器中打开支付页面
3. 页面会显示{method_name}收款码
4. 扫码支付 **{price}{'元' if method_id != 'usdt' else ' USDT'}**
5. 支付完成后返回Telegram
6. 点击"✅ 我已支付"按钮

⚡ **预计激活时间**
• 工作时间：1-5分钟
• 非工作时间：30分钟内

📞 **遇到问题请联系客服**
"""
        
        markup = telebot.types.InlineKeyboardMarkup(row_width=1)
        
        # 跳转到支付页面按钮
        markup.add(
            telebot.types.InlineKeyboardButton(
                f"🌐 前往{method_name}支付页面", 
                url=payment_url
            )
        )
        
        # 支付确认按钮
        markup.add(
            telebot.types.InlineKeyboardButton(
                "✅ 我已支付完成", 
                callback_data=f"vip_confirm_{order_id}"
            )
        )
        
        # 使用统一的客服配置
        markup.add(
            telebot.types.InlineKeyboardButton("📞 联系客服", url="https://t.me/kfjdfkjdd_bot"),
            telebot.types.InlineKeyboardButton("🏠 返回首页", callback_data="back_to_main")
)
        
        # 发送支付页面
        msg = bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    def get_payment_url(self, order_id, price, method_id, months):
        """生成支付页面URL"""
        
        # 根据支付方式选择页面
        page_map = {
            "wechat": "wechat_pay.html",
            "alipay": "alipay_pay.html",
            "usdt": "usdt_pay.html"
        }
        
        page = page_map.get(method_id, "wechat_pay.html")
        
        # 构建支付页面URL
        base_url = PAYMENT_BASE_URL
        if "your-domain.com" in base_url:
            # 如果还没配置，使用临时页面
            return self.create_temp_payment_page(method_id, order_id, price, months)
        
        # 完整的支付页面URL
        payment_url = f"{base_url}/{page}?order={order_id}&amount={price}&months={months}"
        return payment_url
    
    def create_temp_payment_page(self, method_id, order_id, price, months):
        """创建临时支付页面（如果没有配置base_url）"""
        
        method_name = VIPPackageConfig.PAYMENT_METHODS[method_id]["name"]
        
        html_content = f"""
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="UTF-8">
            <title>{method_name}支付</title>
            <style>
                body {{ 
                    font-family: Arial, sans-serif; 
                    padding: 30px; 
                    text-align: center; 
                    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                    min-height: 100vh;
                    display: flex;
                    justify-content: center;
                    align-items: center;
                }}
                .container {{ 
                    background: white; 
                    border-radius: 20px; 
                    padding: 40px; 
                    max-width: 500px; 
                    box-shadow: 0 20px 60px rgba(0,0,0,0.3);
                }}
                .header {{ 
                    color: #4f46e5; 
                    margin-bottom: 20px; 
                }}
                .order-info {{ 
                    background: #f8fafc; 
                    padding: 20px; 
                    border-radius: 15px; 
                    margin: 25px 0; 
                    border-left: 5px solid #3b82f6;
                }}
                .alert {{ 
                    color: #ef4444; 
                    background: #fef2f2; 
                    padding: 15px; 
                    border-radius: 10px; 
                    margin: 20px 0; 
                    border: 2px solid #fca5a5;
                }}
                .steps {{ 
                    text-align: left; 
                    background: #f0f9ff; 
                    padding: 20px; 
                    border-radius: 10px; 
                    margin: 20px 0; 
                }}
                .button {{ 
                    background: #3b82f6; 
                    color: white; 
                    padding: 15px 40px; 
                    border: none; 
                    border-radius: 10px; 
                    font-size: 16px; 
                    font-weight: 600; 
                    cursor: pointer; 
                    margin-top: 20px;
                }}
            </style>
        </head>
        <body>
            <div class="container">
                <h1 class="header">💰 {method_name}支付</h1>
                
                <div class="order-info">
                    <p><strong>订单号：</strong>{order_id}</p>
                    <p><strong>金额：</strong>{price}{'元' if method_id != 'usdt' else ' USDT'}</p>
                    <p><strong>时长：</strong>{months}个月</p>
                </div>
                
                <div class="alert">
                    <h3>⚠️ 支付页面配置提示</h3>
                    <p>请按以下步骤配置完整支付页面：</p>
                </div>
                
                <div class="steps">
                    <h3>📋 配置步骤：</h3>
                    <ol>
                        <li>将{method_id}_pay.html上传到服务器</li>
                        <li>修改bot.py中的PAYMENT_BASE_URL</li>
                        <li>重启机器人</li>
                    </ol>
                    
                    <h3>📱 当前支付方式：</h3>
                    <p>请通过{method_name}支付{price}{'元' if method_id != 'usdt' else ' USDT'}</p>
                    <p>支付后返回Telegram点击确认</p>
                </div>
                
                <p>客服：@your_service_bot</p>
                
                <button class="button" onclick="window.close()">返回Telegram</button>
            </div>
        </body>
        </html>
        """
        
        # 编码为data URL
        encoded = base64.b64encode(html_content.encode()).decode()
        return f"data:text/html;base64,{encoded}"
    
    def notify_admin_new_order(self, user_id, order_id, name, price, months, method_name):
        """通知管理员有新订单"""
        # 获取用户名
        username = "未知用户"
        try:
            user = bot.get_chat(user_id)
            username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
        except:
            pass
        
        admin_message = f"""
🛒 **【新VIP订单通知】**

👤 **用户信息**
• 用户ID：`{user_id}`
• 用户名：{username}

📋 **订单详情**
• 订单号：`{order_id}`
• 套餐：{name}
• 金额：{price}元
• 时长：{months}个月
• 支付方式：{method_name}
• 时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

💳 **支付状态**
• 状态：等待用户支付
• 应收到：{price}元

✅ **操作指南**
1. 等待用户支付完成
2. 用户支付后会点击"我已支付"按钮
3. 您将收到支付确认通知
4. 检查{method_name}是否收到{price}元
5. 确认收款后点击激活按钮
"""
        
        # 发送给管理员
        for admin_id in ADMIN_USER_IDS:
            try:
                bot.send_message(
                    admin_id,
                    admin_message,
                    parse_mode='Markdown'
                )
            except Exception as e:
                print(f"通知管理员失败: {e}")
    
    def confirm_payment(self, call, order_id):
        """用户确认支付"""
        user_id = call.from_user.id
        chat_id = call.message.chat.id
        
        # 更新订单状态
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
        UPDATE payment_orders 
        SET status = 'paid', user_confirmed = 1, payment_time = ?
        WHERE order_id = ?
        ''', (datetime.now().isoformat(), order_id))
        
        conn.commit()
        
        # 更新内存缓存
        if order_id in self.pending_orders:
            self.pending_orders[order_id]["status"] = "paid"
            self.pending_orders[order_id]["user_confirmed"] = True
        
        # 通知用户
        bot.answer_callback_query(
            call.id,
            "✅ 已收到您的支付通知，管理员将在5分钟内为您激活VIP",
            show_alert=True
        )
        
        # 更新消息按钮状态
        try:
            markup = telebot.types.InlineKeyboardMarkup()
            markup.add(
                telebot.types.InlineKeyboardButton(
                    "⏳ 等待管理员激活...", 
                    callback_data="waiting_activation"
                )
            )
            
            bot.edit_message_reply_markup(
                chat_id=chat_id,
                message_id=call.message.message_id,
                reply_markup=markup
            )
        except:
            pass
        
        # 再次通知管理员（支付确认）
        self.notify_admin_payment_confirmed(order_id)
    # ==================== 【新增】套餐管理方法 ====================
    
    def show_package_management(self, user_id, chat_id, message_id=None):
        """显示套餐管理菜单"""
        if not is_admin(user_id):
            bot.send_message(chat_id, "❌ 权限不足", reply_markup=create_main_menu())
            return
        
        packages = self.package_manager.get_all_packages()
        
        if not packages:
            text = "📭 暂无VIP套餐"
        else:
            text = "💰 **VIP套餐管理**\n\n"
            
            for pkg in packages:
                status = "🟢" if pkg['is_active'] else "🔴"
                text += f"{status} **{pkg['name']}**\n"
                text += f"├─ 价格：{pkg['price_cny']}元\n"
                text += f"├─ 时长：{pkg['months']}个月\n"
                text += f"├─ 描述：{pkg['description']}\n"
                text += f"└─ 排序：{pkg['display_order']}\n\n"
        
        text += "请选择要管理的套餐："
        
        markup = telebot.types.InlineKeyboardMarkup(row_width=2)
        
        for pkg in packages:
            btn_text = f"📝 {pkg['name']}"
            markup.add(
                telebot.types.InlineKeyboardButton(
                    btn_text,
                    callback_data=f"manage_package_{pkg['id']}"
                )
            )
        
        markup.add(
            telebot.types.InlineKeyboardButton("➕ 添加新套餐", callback_data="add_new_package"),
            telebot.types.InlineKeyboardButton("📊 套餐统计", callback_data="package_stats")
        )
        markup.add(
            telebot.types.InlineKeyboardButton("⬅️ 返回管理面板", callback_data="back_to_stats"),
            telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
        )
        
        if message_id:
            try:
                bot.edit_message_text(
                    text,
                    chat_id=chat_id,
                    message_id=message_id,
                    parse_mode='Markdown',
                    reply_markup=markup
                )
            except:
                bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
        else:
            bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    def show_package_detail_management(self, user_id, chat_id, package_id, message_id):
        """显示套餐详情管理"""
        if not is_admin(user_id):
            return
        
        packages = self.package_manager.get_all_packages()
        package = next((p for p in packages if p['id'] == package_id), None)
        
        if not package:
            return
        
        status = "🟢 已激活" if package['is_active'] else "🔴 已禁用"
        
        text = f"""
📋 **套餐详情管理**

**{package['name']}**
├─ {status}
├─ 价格：{package['price_cny']}元
├─ 时长：{package['months']}个月
├─ 描述：{package['description']}
└─ 排序：{package['display_order']}

请选择要修改的项目：
"""
        
        markup = telebot.types.InlineKeyboardMarkup(row_width=2)
        
        markup.add(
            telebot.types.InlineKeyboardButton("✏️ 修改名称", callback_data=f"edit_name_{package_id}"),
            telebot.types.InlineKeyboardButton("💰 修改价格", callback_data=f"edit_price_{package_id}")
        )
        markup.add(
            telebot.types.InlineKeyboardButton("📅 修改时长", callback_data=f"edit_months_{package_id}"),
            telebot.types.InlineKeyboardButton("📝 修改描述", callback_data=f"edit_desc_{package_id}")
        )
        markup.add(
            telebot.types.InlineKeyboardButton("🔢 修改排序", callback_data=f"edit_order_{package_id}"),
            telebot.types.InlineKeyboardButton(
                "🔄 激活/禁用" if package['is_active'] else "🔄 激活/禁用",
                callback_data=f"toggle_status_{package_id}"
            )
        )
        markup.add(
            telebot.types.InlineKeyboardButton("🗑️ 删除套餐", callback_data=f"delete_package_{package_id}"),
            telebot.types.InlineKeyboardButton("⬅️ 返回列表", callback_data="manage_packages")
        )
        
        try:
            bot.edit_message_text(
                text,
                chat_id=chat_id,
                message_id=message_id,
                parse_mode='Markdown',
                reply_markup=markup
            )
        except:
            bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    def add_new_package(self, user_id, chat_id):
        """添加新套餐"""
        if not is_admin(user_id):
            return
        
        text = """
➕ **添加新VIP套餐**

请按顺序发送以下信息（每行一项）：

1. 套餐名称（例如：30元包月VIP）
2. 价格（元，例如：30.00）
3. 时长（月，例如：1）
4. 描述（例如：适合短期使用）
5. 排序号（数字，例如：4）

示例：
30元包月VIP
30.00
1
适合短期使用，性价比高
4

⚠️ 注意：请严格按顺序发送，每行一项
"""
        
        markup = telebot.types.InlineKeyboardMarkup()
        markup.add(
            telebot.types.InlineKeyboardButton("❌ 取消", callback_data="manage_packages")
        )
        
        msg = bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
        
        user_sessions = getattr(bot, 'user_sessions', {})
        user_sessions[user_id] = {
            'action': 'add_new_package',
            'step': 1,
            'data': {}
        }
        bot.user_sessions = user_sessions
    def notify_admin_payment_confirmed(self, order_id):
        """通知管理员用户已支付"""
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        cursor.execute('''
        SELECT po.user_id, po.amount, po.payment_method, vp.name, vp.months
        FROM payment_orders po
        JOIN vip_packages vp ON po.package_id = vp.id
        WHERE po.order_id = ?
        ''', (order_id,))
        
        row = cursor.fetchone()
        
        if not row:
            return
        
        user_id, amount, method_id, name, months = row
        method_name = VIPPackageConfig.PAYMENT_METHODS.get(method_id, {}).get("name", method_id)
        
        # 获取用户名
        username = "未知用户"
        try:
            user = bot.get_chat(user_id)
            username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
        except:
            pass
        
        admin_message = f"""
💳 **【用户支付确认通知】**

✅ **用户已确认支付**
• 用户ID：`{user_id}`
• 用户名：{username}
• 订单号：`{order_id}`
• 套餐：{name}
• 金额：{amount}元
• 时长：{months}个月
• 支付方式：{method_name}
• 确认时间：{datetime.now().strftime('%H:%M:%S')}

⚠️ **请立即操作**
1. 检查{method_name}是否收到 {amount}元
2. 核实无误后激活用户VIP
3. 用户正在等待激活
"""
        
        # 激活按钮
        markup = telebot.types.InlineKeyboardMarkup()
        markup.add(
            telebot.types.InlineKeyboardButton(
                f"✅ 立即激活VIP ({months}个月)", 
                callback_data=f"admin_activate_{user_id}_{order_id}_{months}"
            )
        )
        
        # 发送给管理员
        for admin_id in ADMIN_USER_IDS:
            try:
                bot.send_message(
                    admin_id,
                    admin_message,
                    parse_mode='Markdown',
                    reply_markup=markup
                )
            except Exception as e:
                print(f"通知管理员失败: {e}")

vip_payment_system = VIPPaymentSystem()

# ==================== VIP用户管理器 ====================
class VIPUserManager:
    """VIP用户管理器"""
    
    def activate_vip(self, user_id, months):
        """激活用户VIP"""
        print(f"🎯 [VIP激活] 开始 - 用户ID: {user_id}, 月份: {months}")
        
        # 参数验证
        try:
            months = int(months)
            print(f"📅 [VIP激活] 参数转换后月份: {months}")
        except:
            print(f"⚠️ [VIP激活] 参数转换失败，使用默认值1")
            months = 1
        
        # 月份合理性检查
        if months <= 0:
            months = 1
            print(f"⚠️ [VIP激活] 月份<=0，调整为1")
        elif months > 36:
            months = 36
            print(f"⚠️ [VIP激活] 月份>36，限制为36")
        
        print(f"✅ [VIP激活] 最终使用月份: {months}")
        
        conn = None
        cursor = None
        try:
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            now = datetime.now()
            print(f"⏰ [VIP激活] 当前时间: {now}")
            
            # 获取当前VIP状态
            cursor.execute('SELECT vip_expire FROM vip_users WHERE user_id = ?', (user_id,))
            row = cursor.fetchone()
            
            # 确定起始时间
            start_date = now
            if row and row[0]:
                try:
                    current_expire = datetime.fromisoformat(row[0])
                    print(f"📅 [VIP激活] 现有到期时间: {current_expire}")
                    
                    if current_expire > now:
                        # 续费：从现有时间增加
                        print(f"🔄 [VIP激活] 模式：续费")
                        start_date = current_expire
                    else:
                        # 已过期：从当前时间开始
                        print(f"🔄 [VIP激活] 模式：重新激活")
                        start_date = now
                except Exception as e:
                    print(f"⚠️ [VIP激活] 解析失败: {e}")
                    start_date = now
            else:
                # 新用户
                print(f"🔄 [VIP激活] 模式：新用户")
                start_date = now
            
            # 计算新到期时间
            try:
                target_year = start_date.year
                target_month = start_date.month + months
                
                if target_month > 12:
                    target_year += (target_month - 1) // 12
                    target_month = (target_month - 1) % 12 + 1
                
                _, last_day_of_month = calendar.monthrange(target_year, target_month)
                
                if start_date.day > last_day_of_month:
                    target_day = last_day_of_month
                else:
                    target_day = start_date.day
                
                new_expire = datetime(
                    target_year, target_month, target_day,
                    start_date.hour, start_date.minute, start_date.second,
                    start_date.microsecond
                )
                
            except Exception as e:
                print(f"⚠️ [VIP激活] 日期计算失败，使用备选方案: {e}")
                # 备选方案：按30天每月计算
                days_to_add = months * 30
                new_expire = start_date + timedelta(days=days_to_add)
            
            # 计算实际天数
            days_left = (new_expire - now).days
            total_days = (new_expire - start_date).days
            
            print(f"✅ [VIP激活] 新到期时间: {new_expire}")
            print(f"📊 [VIP激活] 实际增加天数: {total_days}天")
            print(f"📊 [VIP激活] 剩余天数: {days_left}天")
            
            # 保存到数据库
            cursor.execute('''
            INSERT OR REPLACE INTO vip_users (user_id, is_vip, vip_expire, updated_at, total_spent)
            VALUES (?, 1, ?, ?, 
                COALESCE((SELECT total_spent FROM vip_users WHERE user_id = ?), 0))
            ''', (user_id, new_expire.isoformat(), now.isoformat(), user_id))
            
            conn.commit()
            print(f"💾 [VIP激活] 数据库保存成功")
            
            # 清除缓存
            cache_key = f"vip_{user_id}"
            if cache_key in user_limit_manager.cache:
                del user_limit_manager.cache[cache_key]
                print(f"🗑️ [VIP激活] 清除缓存成功")
            
            return new_expire
            
        except Exception as e:
            print(f"❌ [VIP激活] 数据库操作失败: {e}")
            import traceback
            traceback.print_exc()
            if conn:
                conn.rollback()
            raise e
        finally:
            if cursor:
                cursor.close()
    
    def get_vip_info(self, user_id):
        """获取用户VIP信息"""
        conn = None
        cursor = None
        try:
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            cursor.execute('''
            SELECT is_vip, vip_expire, total_spent, created_at
            FROM vip_users
            WHERE user_id = ?
            ''', (user_id,))
            
            row = cursor.fetchone()
            
            info = {
                "is_vip": False,
                "vip_expire": None,
                "total_spent": 0,
                "created_at": None,
                "days_left": 0,
                "expire_date_str": "无",
                "is_expired": True
            }
            
            if row and row[0]:  # is_vip为True
                info["is_vip"] = True
                info["vip_expire"] = row[1]
                info["total_spent"] = row[2] or 0
                info["created_at"] = row[3]
                
                if row[1]:
                    try:
                        expire_time = datetime.fromisoformat(row[1])
                        days_left = (expire_time - datetime.now()).days
                        info["days_left"] = max(0, days_left)
                        info["expire_date_str"] = expire_time.strftime('%Y-%m-%d')
                        info["is_expired"] = days_left <= 0
                    except:
                        info["days_left"] = 0
                        info["expire_date_str"] = "永久"
                        info["is_expired"] = False
            
            return info
            
        except Exception as e:
            print(f"❌ 获取VIP信息失败: {e}")
            return {
                "is_vip": False,
                "vip_expire": None,
                "total_spent": 0,
                "created_at": None,
                "days_left": 0,
                "expire_date_str": "获取失败",
                "is_expired": True
            }
        finally:
            if cursor:
                cursor.close()

vip_user_manager = VIPUserManager()

# ==================== Telegram API限流器 ====================
class TelegramRateLimiter:
    """Telegram API请求限流器"""
    
    def __init__(self, max_per_second=25):
        self.max_rps = max_per_second
        self.request_times = []
        self.lock = threading.Lock()
        print(f"⚡ API限流器已启用 ({max_per_second}次/秒)")
    
    def wait_if_needed(self):
        """如果需要则等待"""
        with self.lock:
            now = time.time()
            self.request_times = [t for t in self.request_times if now - t < 1]
            
            if len(self.request_times) >= self.max_rps:
                oldest_request = self.request_times[0]
                time_to_wait = 1.0 - (now - oldest_request)
                if time_to_wait > 0:
                    time.sleep(time_to_wait)
                    now = time.time()
                self.request_times = [t for t in self.request_times if now - t < 1]
            
            self.request_times.append(now)

api_limiter = TelegramRateLimiter(max_per_second=25)

# ==================== 智能文件码识别器 ====================
class EnhancedFileCodeExtractor:
    """增强版文件码提取器"""
    
    def __init__(self):
        print("🔍 增强版智能文件码识别器已启用")
    
    def extract_code_from_text(self, text):
        """从文本中提取文件码（兼容原方法）"""
        if not text:
            return None
        
        codes = self.extract_codes_from_text(text)
        return codes[0] if codes else None
    
    def extract_codes_from_text(self, text):
        """从文本中提取所有文件码"""
        if not text:
            return []
        
        codes = []
        
        # 1. 标准35位码（带yixiangjiqiren_）
        standard_pattern = r'yixiangjiqiren_[a-zA-Z0-9_]{20}'
        matches = re.findall(standard_pattern, text, re.IGNORECASE)
        codes.extend([m for m in matches if len(m) == 35])
        
        # 2. 纯35位码（不带前缀）
        pure_35_pattern = r'[A-HJ-NP-Z2-9]{35}'  # 排除易混淆字符
        matches = re.findall(pure_35_pattern, text)
        codes.extend([m for m in matches if self._is_valid_code(m)])
        
        # 3. 包裹在引号/括号中的文件码
        quoted_pattern = r'["\'`【】（](yixiangjiqiren_[a-zA-Z0-9_]{20})["\'`】）]'
        matches = re.findall(quoted_pattern, text, re.IGNORECASE)
        codes.extend([m for m in matches if len(m) == 35])
        
        # 4. 行尾文件码（常见于消息末尾）
        line_end_pattern = r'[:：]\s*(yixiangjiqiren_[a-zA-Z0-9_]{20})$'
        matches = re.findall(line_end_pattern, text, re.IGNORECASE | re.MULTILINE)
        codes.extend([m for m in matches if len(m) == 35])
        
        # 去重并返回
        return list(dict.fromkeys(codes))
    
    def _is_valid_code(self, code):
        """验证是否为有效的文件码"""
        if not code or len(code) != 35:
            return False
        
        # 检查字符集
        valid_chars = set('ABCDEFGHJKLMNPQRSTUVWXYZ23456789_')
        for char in code.upper():
            if char not in valid_chars:
                return False
        
        return True
    
    def extract_and_context(self, text):
        """提取文件码及其上下文"""
        results = []
        
        # 查找所有可能的文件码位置
        pattern = r'(yixiangjiqiren_[a-zA-Z0-9_]{20})'
        for match in re.finditer(pattern, text, re.IGNORECASE):
            if len(match.group(1)) == 35:
                start = max(0, match.start() - 20)  # 前20个字符
                end = min(len(text), match.end() + 20)  # 后20个字符
                context = text[start:end]
                
                results.append({
                    'code': match.group(1),
                    'position': (match.start(), match.end()),
                    'context': context,
                    'is_valid': True
                })
        
        return results

# 替换原来的提取器
code_extractor = EnhancedFileCodeExtractor()

# ==================== 智能批量整合发送器 ====================
class SmartBatchSender:
    """智能批量整合发送器"""
    
    def __init__(self, bot_instance):
        self.bot = bot_instance
        self.user_sessions = {}
        self.lock = threading.Lock()
        print("📦 智能批量整合发送器启动")
    
    def create_merged_session(self, user_id, chat_id, code_list):
        """创建合并发送会话"""
        print(f"🔄 创建合并会话: {len(code_list)} 个文件码")
        
        with self.lock:
            # 获取所有文件
            all_files = []
            code_details = []
            
            for code_info in code_list:
                code = code_info['code']
                pack = get_pack_by_code(code)
                
                if pack:
                    all_files.extend(pack['files'])
                    code_details.append({
                        'code': code,
                        'file_count': pack['file_count'],
                        'file_types': pack['file_types']
                    })
            
            if not all_files:
                return None, "❌ 未找到有效文件"
            
            # 分页（每页10个文件）
            batches = []
            for i in range(0, len(all_files), 10):
                batches.append(all_files[i:i+10])
            
            session_id = f"batch_{user_id}_{int(time.time())}"
            
            self.user_sessions[session_id] = {
                'user_id': user_id,
                'chat_id': chat_id,
                'session_id': session_id,
                'all_files': all_files,
                'batches': batches,
                'current_batch': 0,
                'total_batches': len(batches),
                'total_files': len(all_files),
                'code_details': code_details,
                'codes_count': len(code_list),
                'created_at': time.time(),
                'last_send_time': 0
            }
            
            return session_id, code_details
    
    def get_next_batch(self, session_id):
        """获取下一批文件"""
        with self.lock:
            if session_id not in self.user_sessions:
                return None
            
            session = self.user_sessions[session_id]
            
            if session['current_batch'] >= session['total_batches']:
                return None
            
            batch_files = session['batches'][session['current_batch']]
            batch_info = {
                'files': batch_files,
                'current_batch': session['current_batch'] + 1,
                'total_batches': session['total_batches'],
                'session_id': session_id,
                'total_files': session['total_files'],
                'codes_count': session['codes_count'],
                'is_last': (session['current_batch'] + 1) == session['total_batches']
            }
            
            session['current_batch'] += 1
            session['last_send_time'] = time.time()
            
            return batch_info
    
    def get_session_info(self, session_id):
        """获取会话信息"""
        with self.lock:
            return self.user_sessions.get(session_id)
    
    def clear_session(self, session_id):
        """清理会话"""
        with self.lock:
            if session_id in self.user_sessions:
                del self.user_sessions[session_id]
    
    def create_batch_menu(self, session_id, can_click=True, remaining_seconds=0):
        """创建批量发送菜单"""
        session = self.get_session_info(session_id)
        if not session:
            return None
        
        markup = telebot.types.InlineKeyboardMarkup()
        
        current = session['current_batch']
        total = session['total_batches']
        
        progress_text = f"📦 {current}/{total}"
        
        if current < total:
            if can_click:
                button_text = "➡️ 下一页"
                callback_data = f"batch_next_{session_id}"
            else:
                if remaining_seconds > 0:
                    button_text = f"⏳ {remaining_seconds}s"
                else:
                    button_text = "⏳ 等待中"
                callback_data = f"batch_wait_{session_id}"
            
            markup.row(
                telebot.types.InlineKeyboardButton(progress_text, callback_data=f"batch_info_{session_id}"),
                telebot.types.InlineKeyboardButton(button_text, callback_data=callback_data)
            )
        else:
            markup.row(
                telebot.types.InlineKeyboardButton("✅ 发送完成", callback_data=f"batch_complete_{session_id}")
            )
        
        markup.row(
            telebot.types.InlineKeyboardButton("📋 批量详情", callback_data=f"batch_detail_{session_id}"),
            telebot.types.InlineKeyboardButton("🏠 返回", callback_data="back_to_main")
        )
        
        return markup

# 创建实例
smart_batch_sender = SmartBatchSender(bot)

# ==================== 批量整合处理函数 ====================
def process_merged_batch(user_id, chat_id, code_list, reply_to_message_id=None):
    """处理批量文件码（整合发送）"""
    print(f"🔄 批量整合处理: {len(code_list)} 个文件码")
    
    # 1. 检查用户限制
    total_files = 0
    valid_codes = []
    
    for code_info in code_list:
        code = code_info['code']
        pack = get_pack_by_code(code)
        
        if pack:
            total_files += pack['file_count']
            valid_codes.append(code_info)
    
    if not valid_codes:
        bot.send_message(
            chat_id,
            "❌ 未找到有效文件码",
            reply_to_message_id=reply_to_message_id,
            reply_markup=create_main_menu()
        )
        return
    
    # 2. 检查文件接收限制
    can_receive, error_msg = user_limit_manager.check_file_receive_limit(user_id, total_files)
    if not can_receive:
        bot.send_message(
            chat_id,
            error_msg,
            reply_to_message_id=reply_to_message_id,
            reply_markup=create_main_menu()
        )
        return
    
    # 3. 更新解码计数
    user_limit_manager.increment_decode_count(user_id)
    # 【新增】增加每个文件码的解码次数
    for code_info in valid_codes:
        pack_decode_manager.increment_decode_count_with_cache(code_info['code'])
    # 4. 创建批量发送会话
    session_id, code_details = smart_batch_sender.create_merged_session(user_id, chat_id, valid_codes)
    
    if not session_id:
        bot.send_message(
            chat_id,
            "❌ 创建批量发送失败",
            reply_to_message_id=reply_to_message_id,
            reply_markup=create_main_menu()
        )
        return
    
    # 5. 显示批量信息
    show_batch_info(user_id, chat_id, session_id, code_details, reply_to_message_id)
    
    # 6. 发送第一批文件
    send_first_batch(user_id, chat_id, session_id)

def show_batch_info(user_id, chat_id, session_id, code_details, reply_to_message_id=None):
    """显示批量信息"""
    session = smart_batch_sender.get_session_info(session_id)
    if not session:
        return
    
    # 构建详细信息
    codes_text = "\n".join([
        f"• `{detail['code'][:15]}...` - {detail['file_count']}个文件 ({detail['file_types']})"
        for detail in code_details[:5]  # 最多显示5个
    ])
    
    if len(code_details) > 5:
        codes_text += f"\n• ... 还有 {len(code_details)-5} 个文件码"
    
    total_files = session['total_files']
    total_batches = session['total_batches']
    
    info_text = f"""
📦 **批量文件整合发送**

✅ **成功解析 {len(code_details)} 个文件码**

📋 **文件码详情：**
{codes_text}

📊 **整合统计：**
• 总文件数：{total_files} 个文件
• 总批次：{total_batches} 批（每批最多10个文件）
• 整合方式：智能合并发送

🚀 **开始发送第一批文件...**
"""
    
    bot.send_message(
        chat_id,
        info_text,
        parse_mode='Markdown',
        reply_to_message_id=reply_to_message_id
    )

def send_first_batch(user_id, chat_id, session_id):
    """发送第一批文件"""
    batch_info = smart_batch_sender.get_next_batch(session_id)
    
    if not batch_info:
        return
    
    # 发送文件
    if send_files_compact(chat_id, batch_info['files']):
        # 更新文件接收计数
        for file_info in batch_info['files']:
            user_limit_manager.increment_file_receive_count(user_id, 1, file_info['file_type'])
        
        # 显示分页控制
        show_batch_pagination(user_id, chat_id, session_id, batch_info)
    else:
        bot.send_message(
            chat_id,
            "❌ 文件发送失败，请重试",
            reply_markup=create_main_menu()
        )
        smart_batch_sender.clear_session(session_id)

def show_batch_pagination(user_id, chat_id, session_id, batch_info):
    """显示批量分页控制"""
    session = smart_batch_sender.get_session_info(session_id)
    if not session:
        return
    
    # 检查点击权限
    if user_limit_manager.is_vip(user_id):
        can_click = True
        wait_time = 0
    else:
        can_click = True  # 第一批无延迟
        wait_time = 0
    
    # 显示分页信息
    text = f"""
📦 **批量发送进行中** ({batch_info['current_batch']}/{batch_info['total_batches']})

📊 **批量信息：**
• 文件码数量：{session['codes_count']} 个
• 总文件数：{session['total_files']} 个
• 当前批次：{batch_info['current_batch']}/{batch_info['total_batches']}
• 本批文件：{len(batch_info['files'])} 个

{'⚡ VIP用户：无点击限制' if user_limit_manager.is_vip(user_id) else '⏰ 普通用户：5秒点击间隔'}
"""
    
    menu = smart_batch_sender.create_batch_menu(
        session_id,
        can_click=can_click,
        remaining_seconds=wait_time
    )
    
    if menu:
        msg = bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=menu
        )

# ==================== 增强版文件发送分页器 ====================
class EnhancedFileSendPaginator:
    """增强版文件发送分页器"""
    
    def __init__(self, bot_instance):
        self.bot = bot_instance
        self.user_sessions = {}
        self.click_times = {}
        self.waiting_messages = {}
        self.lock = threading.Lock()
        print("📦 增强版文件发送分页器启动")
        
        self.update_thread = threading.Thread(target=self.update_countdowns, daemon=True)
        self.update_thread.start()
    
    def can_click_next(self, user_id, chat_id):
        """检查是否可以点击下一页"""
        # VIP用户无限制
        if user_limit_manager.is_vip(user_id):
            self.click_times[(user_id, chat_id)] = time.time()
            return True, 0
        
        # 普通用户：5秒限制
        current_time = time.time()
        key = (user_id, chat_id)
        
        if key not in self.click_times:
            self.click_times[key] = current_time
            return True, 0
        
        time_since_last_click = current_time - self.click_times[key]
        
        if time_since_last_click < 5:
            remaining = math.ceil(5 - time_since_last_click)
            return False, remaining
        
        self.click_times[key] = current_time
        return True, 0
    
    def create_send_session(self, user_id, chat_id, files, code):
        """创建文件发送会话"""
        with self.lock:
            batches = []
            for i in range(0, len(files), 10):
                batches.append(files[i:i+10])
            
            self.user_sessions[(user_id, chat_id)] = {
                'batches': batches,
                'current_batch': 0,
                'total_batches': len(batches),
                'code': code,
                'total_files': len(files),
                'last_click': 0,
                'last_message_id': None
            }
            return len(batches)
    
    def get_next_batch(self, user_id, chat_id):
        """获取下一批文件"""
        key = (user_id, chat_id)
        
        with self.lock:
            if key not in self.user_sessions:
                return None
            
            session = self.user_sessions[key]
            current = session['current_batch']
            total = session['total_batches']
            
            if current >= total:
                return None
            
            batch_files = session['batches'][current]
            
            session['current_batch'] += 1
            session['last_click'] = time.time()
            
            return {
                'files': batch_files,
                'current_batch': current + 1,
                'total_batches': total,
                'code': session['code'],
                'total_files': session['total_files'],
                'is_last': (current + 1) == total
            }
    
    def get_current_session(self, user_id, chat_id):
        """获取当前会话信息"""
        key = (user_id, chat_id)
        with self.lock:
            if key in self.user_sessions:
                return self.user_sessions[key].copy()
            return None
    
    def set_last_message_id(self, user_id, chat_id, message_id):
        """设置最后的消息ID"""
        key = (user_id, chat_id)
        with self.lock:
            if key in self.user_sessions:
                self.user_sessions[key]['last_message_id'] = message_id
    
    def register_waiting_message(self, user_id, chat_id, message_id, initial_wait_time):
        """注册等待状态的消息"""
        key = (user_id, chat_id, message_id)
        with self.lock:
            self.waiting_messages[key] = {
                'last_displayed': initial_wait_time + 1,
                'registered_at': time.time(),
                'user_id': user_id,
                'chat_id': chat_id
            }
    
    def unregister_waiting_message(self, user_id, chat_id, message_id):
        """取消注册等待消息"""
        key = (user_id, chat_id, message_id)
        with self.lock:
            if key in self.waiting_messages:
                del self.waiting_messages[key]
    
    def create_dynamic_menu(self, current_batch, total_batches, can_click=True, remaining_seconds=0):
        """创建动态菜单"""
        markup = telebot.types.InlineKeyboardMarkup()
        
        progress_text = f"📦 {current_batch}/{total_batches}"
        
        if current_batch < total_batches:
            if can_click:
                button_text = "➡️ 下一页"
                callback_data = "send_next"
            else:
                if remaining_seconds > 0:
                    button_text = f"⏳ {remaining_seconds}s"
                else:
                    button_text = "⏳ 等待中"
                callback_data = "countdown_info"
            
            markup.row(
                telebot.types.InlineKeyboardButton(progress_text, callback_data="progress_info"),
                telebot.types.InlineKeyboardButton(button_text, callback_data=callback_data)
            )
        else:
            markup.row(
                telebot.types.InlineKeyboardButton("✅ 发送完成", callback_data="send_complete")
            )
        
        markup.row(
            telebot.types.InlineKeyboardButton("🏠 返回主菜单", callback_data="back_to_main_from_send")
        )
        
        return markup
    
    def update_countdowns(self):
        """自动更新倒计时显示"""
        while True:
            try:
                messages_to_update = []
                current_time = time.time()
                
                with self.lock:
                    waiting_messages_copy = self.waiting_messages.copy()
                
                for (user_id, chat_id, message_id), msg_info in waiting_messages_copy.items():
                    try:
                        session_key = (user_id, chat_id)
                        if session_key not in self.user_sessions:
                            self.unregister_waiting_message(user_id, chat_id, message_id)
                            continue
                        
                        click_key = (user_id, chat_id)
                        if click_key in self.click_times:
                            time_since = current_time - self.click_times[click_key]
                            
                            if time_since < 5:
                                remaining = math.ceil(5 - time_since)
                                if remaining != msg_info.get('last_displayed', 6):
                                    messages_to_update.append((
                                        user_id, chat_id, message_id, 
                                        remaining, False
                                    ))
                                    with self.lock:
                                        if (user_id, chat_id, message_id) in self.waiting_messages:
                                            self.waiting_messages[(user_id, chat_id, message_id)]['last_displayed'] = remaining
                            else:
                                messages_to_update.append((
                                    user_id, chat_id, message_id,
                                    0, True
                                ))
                                self.unregister_waiting_message(user_id, chat_id, message_id)
                    
                    except Exception as e:
                        print(f"处理等待消息时出错: {e}")
                        continue
                
                for user_id, chat_id, message_id, remaining, can_click in messages_to_update:
                    try:
                        session = self.get_current_session(user_id, chat_id)
                        if session:
                            current_batch = session['current_batch']
                            total_batches = session['total_batches']
                            
                            self.bot.edit_message_reply_markup(
                                chat_id=chat_id,
                                message_id=message_id,
                                reply_markup=self.create_dynamic_menu(
                                    current_batch=current_batch,
                                    total_batches=total_batches,
                                    can_click=can_click,
                                    remaining_seconds=remaining
                                )
                            )
                    except telebot.apihelper.ApiTelegramException as e:
                        if "message is not modified" not in str(e):
                            print(f"更新消息失败: {e}")
                    except Exception as e:
                        print(f"更新消息时出错: {e}")
                
                time.sleep(1)
                
            except Exception as e:
                print(f"倒计时更新线程出错: {e}")
                time.sleep(5)
    
    def clear_session(self, user_id, chat_id):
        """清理会话"""
        key = (user_id, chat_id)
        with self.lock:
            if key in self.user_sessions:
                session = self.user_sessions[key]
                if session.get('last_message_id'):
                    msg_key = (user_id, chat_id, session['last_message_id'])
                    if msg_key in self.waiting_messages:
                        del self.waiting_messages[msg_key]
                
                del self.user_sessions[key]
            
            if key in self.click_times:
                del self.click_times[key]

file_send_paginator = EnhancedFileSendPaginator(bot)

# ==================== 高并发会话管理 ====================
class ConcurrentSession:
    """支持1200并发的会话管理（修复版）"""
    
    def __init__(self):
        self.user_sessions = {}
        self.processed_messages = set()
        self.lock = threading.Lock()
        self.session_locks = {}
        print("⚡ 高并发会话管理器启动")
        
        threading.Thread(target=self.cleanup_old_sessions, daemon=True).start()
    
    def get_user_lock(self, user_id):
        """获取用户的独立锁"""
        with self.lock:
            if user_id not in self.session_locks:
                self.session_locks[user_id] = threading.Lock()
            return self.session_locks[user_id]
    
    def add_file(self, user_id, chat_id, file_info):
        """添加文件到会话"""
        user_lock = self.get_user_lock(user_id)
        
        with user_lock:
            message_key = f"{user_id}_{chat_id}_{file_info['message_id']}"
            if message_key in self.processed_messages:
                return False
            
            self.processed_messages.add(message_key)
            
            if user_id not in self.user_sessions:
                self.user_sessions[user_id] = {
                    'files': [],
                    'chat_id': chat_id,
                    'last_time': time.time(),
                    'hint_msg_id': None,
                    'processing': False
                }
            
            self.user_sessions[user_id]['files'].append(file_info)
            self.user_sessions[user_id]['last_time'] = time.time()
            return True
    
    def get_session(self, user_id):
        """获取会话"""
        with self.lock:
            return self.user_sessions.get(user_id)
    
    def clear_session(self, user_id):
        """清除会话"""
        user_lock = self.get_user_lock(user_id)
        
        with user_lock:
            if user_id in self.user_sessions:
                session = self.user_sessions[user_id]
                for file_info in session['files']:
                    message_key = f"{user_id}_{session['chat_id']}_{file_info['message_id']}"
                    if message_key in self.processed_messages:
                        self.processed_messages.remove(message_key)
                del self.user_sessions[user_id]
    
    def should_process(self, user_id):
        """检查是否应该处理"""
        user_lock = self.get_user_lock(user_id)
        with user_lock:
            if user_id not in self.user_sessions:
                return False
            
            session = self.user_sessions[user_id]
            if not session['files'] or session.get('processing', False):
                return False
            
            if time.time() - session['last_time'] > 2:
                session['processing'] = True
                return True
            
            return False
    
    def cleanup_old_sessions(self):
        """清理30分钟前的旧会话"""
        while True:
            time.sleep(300)
            with self.lock:
                current_time = time.time()
                expired_users = []
                
                for user_id, session in self.user_sessions.items():
                    if current_time - session['last_time'] > 1800:
                        expired_users.append(user_id)
                
                for user_id in expired_users:
                    if user_id in self.user_sessions:
                        del self.user_sessions[user_id]

session_mgr = ConcurrentSession()

# ==================== 多线程任务处理器 ====================
class TaskProcessor:
    """多线程处理上传任务"""
    
    def __init__(self, max_workers=200):
        self.max_workers = max_workers
        self.workers = []
        self.task_queue = []
        self.queue_lock = threading.Lock()
        self.is_running = True
        
        for i in range(max_workers):
            worker = threading.Thread(target=self.worker_job, args=(i+1,), daemon=True)
            worker.start()
            self.workers.append(worker)
        
        print(f"👷 启动 {max_workers} 个工作线程")
    
    def add_task(self, user_id):
        """添加任务到队列"""
        with self.queue_lock:
            self.task_queue.append(user_id)
    
    def get_task(self):
        """获取任务"""
        with self.queue_lock:
            if self.task_queue:
                return self.task_queue.pop(0)
            return None
    
    def worker_job(self, worker_id):
        """工作线程任务"""
        while self.is_running:
            try:
                user_id = self.get_task()
                if user_id:
                    process_upload_concurrent(user_id, worker_id)
                else:
                    time.sleep(0.001)
            except Exception:
                time.sleep(0.1)

task_processor = TaskProcessor(max_workers=200)

# ==================== 后台检查线程 ====================
def background_processor_concurrent():
    """后台处理上传"""
    print("🔄 并发后台处理器启动")
    
    while True:
        try:
            if session_mgr:
                all_users = list(session_mgr.user_sessions.keys())
                
                for user_id in all_users:
                    if session_mgr.should_process(user_id):
                        task_processor.add_task(user_id)
            
            time.sleep(0.1)
            
        except Exception:
            time.sleep(2)

bg_thread = threading.Thread(target=background_processor_concurrent, daemon=True)
bg_thread.start()

# ==================== 数据库操作函数 ====================
def save_pack(code, user_id, files):
    """保存文件包到数据库"""
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    try:
        file_types = [f['file_type'] for f in files]
        type_count = defaultdict(int)
        type_map = {'photo': 'p', 'video': 'v', 'document': 'd', 'audio': 'a'}
        
        for ftype in file_types:
            if ftype in type_map:
                type_count[type_map[ftype]] += 1
        
        type_text = ""
        for t, c in sorted(type_count.items()):
            type_text += f"{c}{t} "
        type_text = type_text.strip()
        
        cursor.execute('''
        INSERT INTO packs (code, user_id, file_count, file_types)
        VALUES (?, ?, ?, ?)
        ''', (code, user_id, len(files), type_text))
        
        pack_id = cursor.lastrowid
        
        for file_info in files:
            cursor.execute('''
            INSERT INTO files (pack_id, file_id, file_name, file_type)
            VALUES (?, ?, ?, ?)
            ''', (pack_id, file_info['file_id'], file_info['file_name'], file_info['file_type']))
        
        conn.commit()
        # ==================== 【新增】记录到E盘 ====================
        # 这行是新加的，记录到E盘
        e盘日志器.记录文件包(
            用户id=user_id,
            文件码=code,
            文件数量=len(files),
            文件类型=type_text
        )
        print(f"💾 保存文件包: {code} -> user_id: {user_id}, 文件数: {len(files)}")
        return True
        
    except Exception as e:
        print(f"❌ 保存失败: {e}")
        conn.rollback()
        return False

def get_pack_by_code(code):
    """根据代码获取文件包"""
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    cursor.execute('''
    SELECT p.id, p.code, p.user_id, p.file_count, p.file_types, p.created_at,
           f.id, f.file_id, f.file_name, f.file_type
    FROM packs p
    JOIN files f ON p.id = f.pack_id
    WHERE p.code = ?
    ORDER BY f.id
    ''', (code,))
    
    rows = cursor.fetchall()
    
    if not rows:
        return None
    
    pack = {
        'code': rows[0][1],
        'user_id': rows[0][2],
        'file_count': rows[0][3],
        'file_types': rows[0][4],
        'created_at': rows[0][5],
        'files': []
    }
    
    for row in rows:
        pack['files'].append({
            'file_id': row[7],
            'file_name': row[8],
            'file_type': row[9]
        })
    
    return pack

def generate_pack_code(file_count, file_types):
    """生成35位文件包码"""
    type_count = defaultdict(int)
    type_map = {'photo': 'p', 'video': 'v', 'document': 'd', 'audio': 'a'}
    
    for ftype in file_types:
        if ftype in type_map:
            type_count[type_map[ftype]] += 1
    
    middle_parts = []
    for t in sorted(type_count.keys()):
        middle_parts.append(f"{type_count[t]}{t}")
    
    if not middle_parts:
        middle = f"{file_count}f"
    else:
        middle = "_".join(middle_parts)
    
    base = "yixiangjiqiren"
    used_len = len(base) + len(middle) + 2
    need_len = 35 - used_len
    need_len = max(10, min(need_len, 20))
    
    chars = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"
    random_tail = ''.join(random.choice(chars) for _ in range(need_len))
    code = f"{base}_{middle}_{random_tail}"
    
    if len(code) < 35:
        code = code.ljust(35, '0')
    elif len(code) > 35:
        code = code[:35]
    
    return code

# ==================== 文件发送函数 ====================
def send_files_compact(chat_id, files):
    """发送文件组"""
    if not files:
        return False
    
    files = files[:100]
    groups = []
    for i in range(0, len(files), 10):
        groups.append(files[i:i+10])
    
    success = True
    
    for group_idx, file_group in enumerate(groups):
        media_group = []
        
        for idx, file_info in enumerate(file_group):
            if idx == 0 and group_idx == 0:
                caption = f"📦 共 {len(files)} 个文件"
            else:
                caption = None
            
            if file_info['file_type'] == 'photo':
                media_item = telebot.types.InputMediaPhoto(
                    media=file_info['file_id'],
                    caption=caption
                )
            elif file_info['file_type'] == 'video':
                media_item = telebot.types.InputMediaVideo(
                    media=file_info['file_id'],
                    caption=caption
                )
            elif file_info['file_type'] == 'audio':
                media_item = telebot.types.InputMediaAudio(
                    media=file_info['file_id']
                )
            else:
                media_item = telebot.types.InputMediaDocument(
                    media=file_info['file_id'],
                    caption=caption
                )
            
            media_group.append(media_item)
        
        try:
            api_limiter.wait_if_needed()
            bot.send_media_group(chat_id, media_group)
            time.sleep(0.3)
        except Exception as e:
            print(f"发送失败: {e}")
            success = False
    
    return success

def extract_file_info(message):
    """提取文件信息"""
    if message.document:
        file_id = message.document.file_id
        file_name = message.document.file_name or "文档"
        file_type = "document"
    elif message.photo:
        file_id = message.photo[-1].file_id
        file_name = "图片"
        file_type = "photo"
    elif message.video:
        file_id = message.video.file_id
        file_name = "视频"
        file_type = "video"
    elif message.audio:
        file_id = message.audio.file_id
        file_name = "音频"
        file_type = "audio"
    else:
        return None
    
    return {
        'file_id': file_id,  # 🎯 确保有这个字段
        'file_name': file_name,
        'file_type': file_type,
        'message_id': message.message_id,
        'time': datetime.now().isoformat()
    }

# ==================== 并发处理上传 ====================
def process_upload_concurrent(user_id, worker_id):
    """处理用户上传（添加去重功能）"""
    session = session_mgr.get_session(user_id)
    if not session or not session['files']:
        return None
    
    files = session['files']
    chat_id = session['chat_id']
    hint_msg_id = session.get('hint_msg_id')
    
    if hint_msg_id:
        try:
            bot.delete_message(chat_id, hint_msg_id)
        except:
            pass
    
    # 🎯 【新增】文件去重逻辑
    unique_files = []
    duplicate_count = 0
    seen_file_ids = set()
    
    for file_info in files:
        file_id = file_info['file_id']
        if file_id not in seen_file_ids:
            seen_file_ids.add(file_id)
            unique_files.append(file_info)
        else:
            duplicate_count += 1
    
    print(f"🔍 去重统计 - 用户 {user_id}: 原始{len(files)}个, 重复{duplicate_count}个, 保存{len(unique_files)}个")
    
    # 🎯 【修改】使用去重后的文件
    file_types = [f['file_type'] for f in unique_files]
    
    code = None
    for _ in range(10):
        # 🎯 【修改】使用unique_files的长度
        temp_code = generate_pack_code(len(unique_files), file_types)
        
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        cursor.execute('SELECT COUNT(*) FROM packs WHERE code = ?', (temp_code,))
        count = cursor.fetchone()[0]
        
        if count == 0:
            code = temp_code
            break
    
    if not code:
        bot.send_message(chat_id, "❌ 生成唯一代码失败，请重试", 
                        reply_markup=create_main_menu())
        session_mgr.clear_session(user_id)
        return None
    
    # 🎯 【修改】保存去重后的文件
    if not save_pack(code, user_id, unique_files):
        bot.send_message(chat_id, "❌ 保存失败，请重试",
                        reply_markup=create_main_menu())
        session_mgr.clear_session(user_id)
        return None
    
    # 🎯 【修改】简化结果文本
    pack_info = get_pack_by_code(code)
    file_types_text = pack_info['file_types'] if pack_info else ""
    
    result_text = f"""
✅ **文件保存完成！**

📦 **35位文件码：**
`{code}`
🤖 **解码机器人：** [@{BOT_USERNAME}]({BOT_LINK})

📊 **文件统计：**
"""
    
    # 保留原有的去重信息展示
    if duplicate_count > 0:
        result_text += f"• 上传文件：{len(files)} 个\n"
        result_text += f"• 重复文件：{duplicate_count} 个（已去重）\n"
        result_text += f"• 实际保存：{len(unique_files)} 个文件\n"
    else:
        result_text += f"• 文件数量：{len(unique_files)} 个文件\n"
    
    result_text += f"• 文件类型：{file_types_text}\n\n"
    
    # 原有的VIP/普通用户提示
    if user_limit_manager.is_vip(user_id):
        result_text += "⭐ VIP专享：无限解码和下载"
    else:
        result_text += "💡 普通用户每日限制：50次解码，50个文件"
    
    # 原有的去重说明
    if duplicate_count > 0:
        result_text += f"\n\n🔄 **去重说明：**\n系统自动去除了 {duplicate_count} 个重复文件"
    
    # 🎯 【简化】只保留一个分享按钮
    markup = telebot.types.InlineKeyboardMarkup(row_width=1)
    
    # 只保留分享按钮
    import urllib.parse

    share_text = f"📑 文件码：{code}\n🤖 解码机器人：@{BOT_USERNAME}"
# 对文本进行URL编码
    encoded_text = urllib.parse.quote(share_text, safe='')
# 构建完整的分享URL
    share_url = f"https://t.me/share/url?url={urllib.parse.quote(BOT_LINK)}&text={encoded_text}"
    markup.add(
        telebot.types.InlineKeyboardButton("📤 分享文件码", url=share_url)
    )
    
    bot.send_message(
        chat_id, 
        result_text, 
        parse_mode='Markdown',
        reply_markup=markup
    )
    
    print(f"👷 线程{worker_id} 处理完成用户 {user_id}，原始文件:{len(files)}个，去重后:{len(unique_files)}个，重复:{duplicate_count}个")
    session_mgr.clear_session(user_id)
    
    return code

# ==================== 智能文件码处理函数 ====================
def process_file_code_silent(user_id, chat_id, code, original_message_id=None):
    """静默处理文件码"""
    print(f"🔄 智能处理文件码: {code}")
    
    # 检查解码限制
    can_decode, error_msg = user_limit_manager.check_decode_limit(user_id)
    if not can_decode:
        bot.send_message(
            chat_id,
            error_msg,
            reply_to_message_id=original_message_id,
            reply_markup=create_main_menu()
        )
        return
    
    pack = get_pack_by_code(code)
    
    if not pack:
        error_msg = f"❌ 文件码 `{code}` 不存在或已过期"
        bot.send_message(
            chat_id,
            error_msg,
            parse_mode='Markdown',
            reply_to_message_id=original_message_id,
            reply_markup=create_main_menu()
        )
        return
    
    # 检查文件接收限制
    can_receive, error_msg = user_limit_manager.check_file_receive_limit(user_id, pack['file_count'])
    if not can_receive:
        bot.send_message(
            chat_id,
            error_msg,
            reply_to_message_id=original_message_id,
            reply_markup=create_main_menu()
        )
        return
    # 【新增】增加文件码的解码次数
    pack_decode_manager.increment_decode_count_with_cache(code)
    # 更新解码计数
    user_limit_manager.increment_decode_count(user_id)
    
    # 根据文件数量选择处理方式
    if pack['file_count'] <= 10:
        if send_files_compact(chat_id, pack['files']):
            # 更新文件接收计数
            for file_info in pack['files']:
                user_limit_manager.increment_file_receive_count(user_id, 1, file_info['file_type'])
            
            daily_stats = user_limit_manager.get_daily_stats(user_id)
            
            # 🎯 【简化】只添加机器人信息
            success_msg = f"✅ 已发送 {pack['file_count']} 个文件\n`{code}`\n🤖 **解码机器人：** [@{BOT_USERNAME}]({BOT_LINK})\n\n📊 今日已解码：{daily_stats['decode_count']}/50，接收文件：{daily_stats['file_receive_count']}/50"
            
            bot.send_message(
                chat_id,
                success_msg,
                parse_mode='Markdown',
                reply_to_message_id=original_message_id,
                reply_markup=create_main_menu()
            )
        else:
            bot.send_message(
                chat_id,
                "❌ 发送失败，请稍后重试",
                reply_to_message_id=original_message_id,
                reply_markup=create_main_menu()
            )
    else:
        file_send_paginator.clear_session(user_id, chat_id)
        
        total_batches = file_send_paginator.create_send_session(
            user_id, chat_id, pack['files'], code
        )
        
        first_batch = file_send_paginator.get_next_batch(user_id, chat_id)
        if first_batch:
            if send_files_compact(chat_id, first_batch['files']):
                # 更新第一批文件的接收计数
                for file_info in first_batch['files']:
                    user_limit_manager.increment_file_receive_count(user_id, 1, file_info['file_type'])
                
                send_page_info_silent(user_id, chat_id, first_batch, original_message_id)
            else:
                bot.send_message(
                    chat_id,
                    "❌ 发送失败，请重试",
                    reply_markup=create_main_menu()
                )
                file_send_paginator.clear_session(user_id, chat_id)

def send_page_info_silent(user_id, chat_id, batch_info, original_message_id=None):
    """静默发送分页信息"""
    text = f"""
📦 **文件发送中...** ({batch_info['current_batch']}/{batch_info['total_batches']})

🔢 **文件码：** `{batch_info['code']}`
📊 **进度：** {batch_info['current_batch']}/{batch_info['total_batches']} 批
📋 **本批：** {len(batch_info['files'])} 个文件
📁 **总计：** {batch_info['total_files']} 个文件

{'⚡ VIP用户：无点击限制' if user_limit_manager.is_vip(user_id) else '⏰ 普通用户：5秒点击间隔'}
"""
    
    can_click, wait_time = file_send_paginator.can_click_next(user_id, chat_id)
    
    menu = file_send_paginator.create_dynamic_menu(
        current_batch=batch_info['current_batch'],
        total_batches=batch_info['total_batches'],
        can_click=can_click,
        remaining_seconds=wait_time
    )
    
    msg = bot.send_message(
        chat_id,
        text,
        parse_mode='Markdown',
        reply_markup=menu
    )
    
    file_send_paginator.set_last_message_id(user_id, chat_id, msg.message_id)
    
    if not can_click and wait_time > 0:
        file_send_paginator.register_waiting_message(user_id, chat_id, msg.message_id, wait_time)

def show_vip_center(user_id, chat_id):
    """显示VIP中心"""
    vip_info = vip_user_manager.get_vip_info(user_id)
    daily_stats = user_limit_manager.get_daily_stats(user_id)
    
    if vip_info["is_vip"] and not vip_info["is_expired"]:
        text = f"""
⭐ **VIP用户中心**

✅ **VIP状态：** 已激活
📅 **到期时间：** {vip_info['expire_date_str']}
⏰ **剩余天数：** {vip_info['days_left']}天
💰 **累计消费：** {vip_info['total_spent']:.2f}元

🎁 **您的特权**
• 无限解码次数
• 无限接收文件
• 无点击下一页限制
• 最大500个文件/包
• 单个文件最大500MB
• 批量下载功能
• 无广告体验

💡 **您已解锁所有功能！**
"""
    elif vip_info["is_vip"] and vip_info["is_expired"]:
        text = f"""
⚠️ **VIP用户中心**

⏰ **VIP状态：** 已过期
📅 **过期时间：** {vip_info['expire_date_str']}
💰 **累计消费：** {vip_info['total_spent']:.2f}元

🔓 **您的特权已失效**
请续费VIP以恢复所有特权：
"""
    else:
        text = f"""
🔓 **普通用户中心**

📊 **今日使用情况**
• 解码次数：{daily_stats['decode_count']}/50
• 接收文件：{daily_stats['file_receive_count']}/50
• 下一页间隔：5秒

⛔ **普通用户限制**
• 每日最多解码50次
• 每日最多接收50个文件
• 点击下一页需间隔5秒
• 最多50个文件/包
• 单个文件最大20MB
• 有广告显示

💎 **升级VIP解锁所有限制**
"""
    
    markup = telebot.types.InlineKeyboardMarkup(row_width=2)
    
    if not vip_info["is_vip"] or vip_info["is_expired"]:
        markup.add(
            telebot.types.InlineKeyboardButton("💰 购买VIP套餐", callback_data="vip_packages"),
            telebot.types.InlineKeyboardButton("📋 套餐详情", callback_data="vip_compare")
        )
    else:
        markup.add(
            telebot.types.InlineKeyboardButton("🔄 续费VIP", callback_data="vip_packages"),
        )
    
    # 【新增】管理员套餐管理按钮
    if is_admin(user_id):
        markup.add(
            telebot.types.InlineKeyboardButton("⚙️ 套餐管理", callback_data="manage_packages")
        )
    
    markup.add(
        telebot.types.InlineKeyboardButton("❓ 常见问题", callback_data="vip_faq"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)

# ==================== 分页相关功能 ====================
def get_user_packs(user_id):
    """获取用户的所有文件包（包含解码次数）"""
    # 使用新的解码管理器获取数据
    return pack_decode_manager.get_user_packs_with_stats(user_id)

class PageManager:
    """翻页管理器"""
    
    def __init__(self):
        self.user_pages = {}
        self.lock = threading.Lock()
    
    def set_user_packs(self, user_id, packs):
        """设置用户的文件包"""
        with self.lock:
            self.user_pages[user_id] = {
                'packs': packs,
                'page': 1,
                'last_view': time.time()
            }
    
    def get_page(self, user_id, page_num=None):
        """获取指定页"""
        with self.lock:
            if user_id not in self.user_pages:
                return None, None
            
            data = self.user_pages[user_id]
            packs = data['packs']
            
            if page_num is not None:
                data['page'] = page_num
                data['last_view'] = time.time()
            
            page = data['page']
            items_per_page = 10
            total_pages = (len(packs) + items_per_page - 1) // items_per_page
            
            if page < 1 or page > total_pages:
                return None, None
            
            start_idx = (page - 1) * items_per_page
            end_idx = min(start_idx + items_per_page, len(packs))
            
            return packs[start_idx:end_idx], {
                'current_page': page,
                'total_pages': total_pages,
                'total_items': len(packs),
                'start_idx': start_idx + 1,
                'end_idx': end_idx
            }

page_manager = PageManager()

def create_page_menu(current_page, total_pages, packs_on_page):
    """创建翻页菜单（带文件码详情按钮）"""
    markup = telebot.types.InlineKeyboardMarkup()
    
    if total_pages > 1:
        row = []
        
        if current_page > 1:
            row.append(telebot.types.InlineKeyboardButton("⬅️ 上一页", callback_data=f"page_{current_page-1}"))
        
        row.append(telebot.types.InlineKeyboardButton(f"{current_page}/{total_pages}", callback_data="page_info"))
        
        if current_page < total_pages:
            row.append(telebot.types.InlineKeyboardButton("下一页 ➡️", callback_data=f"page_{current_page+1}"))
        
        markup.row(*row)
    
    # 添加文件码详情按钮（当前页的每个文件码）
    for i, pack in enumerate(packs_on_page[:10], 1):  # 最多显示5个按钮
        btn_text = f"{i}. {pack['code'][:8]}..."
        # 检查是否有备注
        remark_info = pack_remark_manager.get_remark(pack['user_id'], pack['code'])
        if remark_info:
            btn_text += " 📝"
        
        markup.row(
            telebot.types.InlineKeyboardButton(
                btn_text,
                callback_data=f"code_detail_{pack['code']}"
            )
        )
    
    # 添加功能按钮
    markup.row(
        telebot.types.InlineKeyboardButton("🔍 搜索备注", callback_data="search_remarks"),
        telebot.types.InlineKeyboardButton("📁 我的代码", callback_data="my_codes")
    )
    
    markup.row(telebot.types.InlineKeyboardButton("🏠 返回主菜单", callback_data="back_to_main"))
    
    return markup

def show_user_codes_page(user_id, chat_id, page_num=1):
    """显示用户的文件码页面（带备注和解码次数）"""
    packs = get_user_packs(user_id)  # 现在这个函数会返回包含decode_count的数据
    
    if not packs:
        bot.send_message(
            chat_id,
            "📭 你还没有上传过文件\n\n💡 转发文件给我开始使用",
            reply_markup=create_main_menu()
        )
        return
    
    # 获取每个文件码的备注
    for pack in packs:
        remark_info = pack_remark_manager.get_remark(user_id, pack['code'])
        if remark_info:
            pack['remark'] = remark_info['remark']
            pack['has_remark'] = True
        else:
            pack['remark'] = None
            pack['has_remark'] = False
    
    page_manager.set_user_packs(user_id, packs)
    page_packs, page_info = page_manager.get_page(user_id, page_num)
    
    if not page_packs:
        bot.send_message(chat_id, "❌ 页码错误", reply_markup=create_main_menu())
        return
    
    # 【新增】计算本页解码统计
    total_decodes = sum(pack.get('decode_count', 0) for pack in page_packs)
    avg_decodes = total_decodes / len(page_packs) if page_packs else 0
    
    page_text = f"""
📁 **我的文件码** 📝

📊 **统计信息：**
• 文件包总数：{page_info['total_items']} 个
• 当前显示：{page_info['start_idx']}-{page_info['end_idx']} 项
• 总页数：{page_info['total_pages']} 页
• 当前页：{page_info['current_page']}/{page_info['total_pages']}
• 本页解码次数：{total_decodes} 次
• 平均解码：{avg_decodes:.1f} 次/文件包

📋 **文件码列表：**
"""
    
    for idx, pack in enumerate(page_packs):
        item_num = (page_info['current_page'] - 1) * 10 + idx + 1
        
        created_at = pack['created_at']
        if isinstance(created_at, str):
            try:
                time_str = created_at[:16].replace('T', ' ')
            except:
                time_str = created_at
        else:
            time_str = str(created_at)
        
        # 显示文件码和备注
        page_text += f"\n**{item_num}. ** `{pack['code']}`\n"
        
        if pack.get('remark'):
            remark_preview = pack['remark'][:30] + ("..." if len(pack['remark']) > 30 else "")
            page_text += f"├─ 📝 备注：{remark_preview}\n"
        
        # 【新增】显示解码次数
        decode_count = pack_decode_manager.get_decode_count_with_cache(pack['code'])
        # 选择图标
        if decode_count == 0:
            decode_icon = "📭"
        elif decode_count <= 10:
            decode_icon = "📊"
        else:
            decode_icon = "🔥"
        
        page_text += f"├─ {decode_icon} 解码：{decode_count} 次\n"
        page_text += f"├─ 文件：{pack['file_count']}个\n"
        page_text += f"├─ 类型：{pack['file_types']}\n"
        page_text += f"└─ 时间：{time_str}\n"
    
    page_text += f"""
💡 **图标说明：**
📭 - 0次下载  📊 - 1-10次  🔥 - 10+次

点击文件码查看详情和管理备注
"""
    
    page_menu = create_page_menu(page_info['current_page'], page_info['total_pages'], page_packs)
    
    bot.send_message(
        chat_id,
        page_text,
        parse_mode='Markdown',
        reply_markup=page_menu
    )

# ==================== 消息处理器 ====================
@bot.message_handler(commands=['start', 'help'])
def handle_start(message):
    """处理/start命令"""
    welcome_text = f"""
🤖 **文件码机器人**

🤖 **解码机器人：** @{BOT_USERNAME}

🆕 **核心功能：**
• 上传文件生成唯一码
• 发送文件码获取文件
• 自动去重处理
• 文件永久保存

⭐ **VIP特权：**
• 无限解码次数
• 无限接收文件
• 无点击限制

📱 **底部菜单已启用**
"""
    
    bot.send_message(
        message.chat.id,
        welcome_text,
        parse_mode='Markdown',
        reply_markup=create_main_menu()
    )
@bot.message_handler(commands=['menu', 'mycodes', 'vip', 'stats', 'admin', 'search'])
def handle_other_commands(message):
    """处理其他斜杠命令（转发到对应的功能）"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    command = message.text.split()[0].lower()
    
    if command == '/menu':
        # 显示使用说明
        handle_menu_buttons(type('Message', (), {'text': '📖 使用说明', 'chat': type('Chat', (), {'id': chat_id})(), 'from_user': type('User', (), {'id': user_id})()})())
        
    elif command == '/mycodes':
        # 显示我的代码
        handle_menu_buttons(type('Message', (), {'text': '📁 我的代码', 'chat': type('Chat', (), {'id': chat_id})(), 'from_user': type('User', (), {'id': user_id})()})())
        
    elif command == '/vip':
        # 显示VIP服务
        handle_menu_buttons(type('Message', (), {'text': '⭐ VIP服务', 'chat': type('Chat', (), {'id': chat_id})(), 'from_user': type('User', (), {'id': user_id})()})())
        
    elif command == '/stats':
        # 统计信息（仅管理员）
        handle_stats_command(message)
        
    elif command == '/admin':
        # 管理员面板
        handle_admin_command(message)
        
    elif command == '/search':
        # 搜索备注
        handle_remark_search_command(message)
@bot.message_handler(func=lambda m: m.text in ["📖 使用说明", "📜 用户协议", "📊 当前状态", "📁 我的代码", "⭐ VIP服务", "👤 我的账户"])
def handle_menu_buttons(message):
    """处理菜单按钮（移除斜杠命令）"""
    text = message.text
    
    # 移除斜杠命令的处理，只处理按钮文本
    command_map = {
        "/menu": "📖 使用说明",
        "/mycodes": "📁 我的代码",
        "/vip": "⭐ VIP服务",
        "/stats": "stats_command",
        "/admin": "admin_panel"
    }
    
    # 如果text在command_map中，转换为对应的按钮文本
    if text in command_map:
        if command_map[text] == "stats_command":
            handle_stats_command(message)
            return
        elif command_map[text] == "admin_panel":
            handle_admin_command(message)
            return
        text = command_map[text]
    
    if text == "📖 使用说明":
        content = """📖 云盘解码器-介绍

1️⃣ 直接批量上传或转发给我可生成代码
2️⃣ 生成的代码发给我，我给你文件
3️⃣ 解码yixiangjiqiren类型直接发给我获得媒体

🆕 **智能识别功能：**
  直接发送包含文件码的消息：
• 直接复制文件码发送
• 转发包含文件码的消息
• 消息中提及文件码
• 自动处理

🆕 **备注功能：**
• 为文件码添加备注方便查找
• 通过备注搜索文件码
• 管理员可搜索所有备注

⭐ **VIP特权：**
• 无限解码次数
• 无限接收文件
• 无点击限制

注意事项：
1. 请适度使用
2. 禁止发送违规内容"""
        
        bot.send_message(
            message.chat.id,
            content,
            parse_mode='Markdown',
            reply_markup=create_main_menu()
        )
        return  # ✅ 添加return
        
    elif text == "📜 用户协议":
        content = """用户协议"云盘解码器"

欢迎您使用本服务。请在使用本服务之前仔细阅读以下条款。

1. 服务简介
本服务基于Telegram机器人API服务，提供用户上传、下载和分享服务。

2. 用户责任
用户在使用本服务时，承诺不上传任何违反法律法规的内容。

3. 内容审核与免责
我方不具备对用户上传内容进行全面审查的能力。

4. 用户隐私
我方承诺保护用户隐私。

5. 服务中断
我方有权在不提前通知的情况下暂停或终止服务。

6. 协议变更
本协议条款可能随时更新。

使用本服务即表示您已接受上述条款。"""
        
        bot.send_message(
            message.chat.id,
            content,
            parse_mode='Markdown',
            reply_markup=create_main_menu()
        )
        return  # ✅ 添加return
        
    elif text == "📊 当前状态":
        user_id = message.from_user.id
        
        if is_admin(user_id):
            # 管理员显示带广播按钮的菜单
            show_admin_status_menu(message)
            return
        else:
            content = f"""
📊 **当前状态**

🟢 运行状态：正常
📅 当前时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
⏰ 服务时间：24/7

🎯 **系统特性：**
• 智能文件码识别
• 支持批量文件处理
• 文件备注管理
• 备注搜索功能
• VIP专属特权
• 安全稳定运行

💡 **温馨提示：**
如需查看详细统计，请联系管理员。
"""
        
            bot.send_message(
                message.chat.id,
                content,
                parse_mode='Markdown',
                reply_markup=create_main_menu()
            )
            return  # ✅ 添加return，注意缩进在else分支内
        
    elif text == "📁 我的代码":
        user_id = message.from_user.id
        chat_id = message.chat.id
        show_user_codes_page(user_id, chat_id, 1)
        return  # ✅ 已经有return
        
    elif text == "⭐ VIP服务":
        user_id = message.from_user.id
        chat_id = message.chat.id
        show_vip_center(user_id, chat_id)
        return  # ✅ 已经有return
        
    elif text == "👤 我的账户":
        user_id = message.from_user.id
        chat_id = message.chat.id
        show_vip_center(user_id, chat_id)
        return  # ✅ 已经有return
        
    else:
        content = "未知命令，请使用菜单按钮"
        # 只有未知命令才会执行到这里
        bot.send_message(
            message.chat.id,
            content,
            parse_mode='Markdown' if text in ["📖 使用说明", "📜 用户协议", "📊 当前状态"] else None,
            reply_markup=create_main_menu()
        )
        return  # ✅ 添加return
def show_admin_status_menu(message):
    """显示管理员状态菜单（带广播按钮）"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    
    if not is_admin(user_id):
        bot.send_message(chat_id, "❌ 权限不足", reply_markup=create_main_menu())
        return
    
    text = f"""
👑 **管理员状态面板**

📊 **系统状态**
🟢 运行状态：正常
📅 当前时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
⏰ 服务时间：24/7

📈 **管理员功能**
1. 查看系统统计
2. 用户管理
3. 订单管理
4. 发送广播

💡 **操作指南**
点击下方按钮选择功能
"""
    
    # 创建带广播按钮的菜单
    markup = telebot.types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        telebot.types.InlineKeyboardButton("📊 系统统计", callback_data="back_to_stats"),
        telebot.types.InlineKeyboardButton("📢 发送广播", callback_data="admin_broadcast")
    )
# 【新增】支付方式管理按钮
    markup.add(
        telebot.types.InlineKeyboardButton("💳 支付方式管理", callback_data="manage_payment_methods")
    )

    markup.add(
        telebot.types.InlineKeyboardButton("👥 用户管理", callback_data="admin_users"),
        telebot.types.InlineKeyboardButton("💰 订单管理", callback_data="admin_pending_orders")
    )
    markup.add(
        telebot.types.InlineKeyboardButton("🏠 返回主菜单", callback_data="back_to_main")
    )
    
    bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
@bot.message_handler(content_types=['photo', 'document', 'video', 'audio'])
def handle_file_upload(message):
    """处理文件上传"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    
    file_info = extract_file_info(message)
    if not file_info:
        return
    
    added = session_mgr.add_file(user_id, chat_id, file_info)
    if not added:
        return
    
    session = session_mgr.get_session(user_id)
    if not session:
        return
    
    if len(session['files']) == 1:
        hint_msg = bot.send_message(chat_id, "📥 开始接收文件...")
        session['hint_msg_id'] = hint_msg.message_id
    
    if session['hint_msg_id']:
        try:
            bot.edit_message_text(
                f"📥 已收到 {len(session['files'])} 个文件...",
                chat_id=chat_id,
                message_id=session['hint_msg_id']
            )
        except:
            pass
# ==================== 【新增】套餐管理消息处理器 ====================
@bot.message_handler(func=lambda m: check_if_package_management(m))
def handle_package_management_messages(message):
    """专门处理套餐管理相关的消息（优先级最高）"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    text = message.text.strip() if message.text else ""
    
    print(f"🎯 [套餐管理处理器] 捕获到套餐管理消息: {text}")
    
    # 调用 handle_all_messages 来处理
    handle_all_messages(message)
# ==================== 智能文本消息处理器 ====================
@bot.message_handler(content_types=['text'])
def handle_text_messages(message):
    """处理文本消息（支持批量整合）"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    text = message.text.strip() if message.text else ""
    
    print(f"📨 收到消息: {text[:50]}...")
    # 🔴【重要】先检查是否是套餐管理消息
    if check_if_package_management(message):
        print(f"⚠️ 这是套餐管理消息，跳过文本处理器")
        return  # 直接返回，不处理
    
    # 🔴【重要】再检查是否是命令
    if text and text.startswith('/'):
        print(f"⚠️ 这是命令，跳过文本处理器")
        return  # 让命令处理器处理
    
    # 🔴【重要】再检查是否是菜单按钮
    menu_buttons = ["📖 使用说明", "📜 用户协议", "📊 当前状态", "📁 我的代码", "⭐ VIP服务", "👤 我的账户"]
    if text in menu_buttons:
        print(f"⚠️ 这是菜单按钮，跳过文本处理器")
        return  # 让菜单按钮处理器处理
    
    # 使用增强版提取器提取所有文件码
    all_codes = code_extractor.extract_codes_from_text(text)
    
    if all_codes:
        print(f"🔍 识别到 {len(all_codes)} 个文件码: {all_codes}")
        
        # 转换为code_info格式
        code_list = []
        for i, code in enumerate(all_codes):
            code_list.append({
                'code': code,
                'source': 'text',
                'message_id': message.message_id,
                'index': i + 1
            })
        
        # 启动批量整合处理
        process_merged_batch(user_id, chat_id, code_list, message.message_id)
        return
    
    # 检查是否是备注搜索
    if text.startswith("/search"):
        handle_remark_search_command(message)
        return
    
    # 如果没有提取到文件码，提示用户
    bot.send_message(
        chat_id,
        "💡 请发送文件码或转发包含文件码的消息\n\n"
        "✅ 支持格式：\n"
        "• 纯文件码: `yixiangjiqiren_xxxxxxxxxxxxxxx`\n"
        "• 文字+文件码: `我的文件码是 yixiangjiqiren_xxxxxxxxxxxxxxx`\n"
        "• 多个文件码: 可以在一行发送多个文件码\n\n"
        "🔍 搜索备注: 使用 /search 关键词",
        parse_mode='Markdown',
        reply_to_message_id=message.message_id,
        reply_markup=create_main_menu()
    )

# ==================== 【新增】备注搜索命令 ====================
@bot.message_handler(commands=['search'])
def handle_remark_search_command(message):
    """处理备注搜索命令"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    
    # 提取搜索关键词
    command_parts = message.text.split()
    if len(command_parts) < 2:
        # 显示搜索提示
        text = """
🔍 **文件码备注搜索**

请输入搜索关键词：
• 可以搜索备注内容
• 可以搜索标签
• 支持模糊搜索

📌 **使用方法：**
`/search 关键词`

例如：
`/search 工作报告`
`/search 重要`
`/search 图片`

💡 **提示：**
• 搜索自己的备注内容
• 快速找到已备注的文件码
• 管理员可以搜索所有用户备注
"""
        
        markup = telebot.types.InlineKeyboardMarkup()
        markup.add(
            telebot.types.InlineKeyboardButton("📁 查看我的文件码", callback_data="my_codes"),
            telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
        )
        
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
        return
    
    keyword = " ".join(command_parts[1:]).strip()
    
    if len(keyword) < 1:
        bot.send_message(chat_id, "❌ 请输入有效的搜索关键词", reply_markup=create_main_menu())
        return
    
    if len(keyword) > 50:
        bot.send_message(chat_id, "❌ 搜索关键词太长（最多50字）", reply_markup=create_main_menu())
        return
    
    # 执行搜索
    if is_admin(user_id):
        # 管理员搜索所有用户备注
        results = pack_remark_manager.search_all_remarks_admin(keyword)
    else:
        # 普通用户搜索自己的备注
        results = pack_remark_manager.search_user_remarks(user_id, keyword)
    
    if not results:
        text = f"""
🔍 **搜索结果：** "{keyword}"

❌ 未找到匹配的文件码

💡 提示：
1. 检查关键词是否正确
2. 尝试更短的关键词
3. 确保已为文件码添加备注
"""
    else:
        text = f"""
🔍 **搜索结果：** "{keyword}"

✅ 找到 {len(results)} 个匹配的文件码：

"""
        
        for i, result in enumerate(results[:10], 1):  # 最多显示10个
            # 截断长备注
            remark_display = result['remark']
            if len(remark_display) > 40:
                remark_display = remark_display[:37] + "..."
            
            text += f"\n**{i}. ** `{result['code']}`\n"
            text += f"├─ 📝 {remark_display}\n"
            if result.get('tags'):
                text += f"├─ 🏷️ 标签：{result['tags']}\n"
            text += f"├─ 文件：{result['file_count']}个\n"
            text += f"└─ 类型：{result['file_types']}\n"
        
        if len(results) > 10:
            text += f"\n... 还有 {len(results)-10} 个结果未显示"
        
        text += "\n💡 发送上面的文件码即可取回文件"
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("🔄 重新搜索", callback_data="search_remarks"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    bot.send_message(
        chat_id,
        text,
        parse_mode='Markdown',
        reply_markup=markup
    )

# ==================== 转发消息处理器 ====================
@bot.message_handler(content_types=['text'], func=lambda m: m.forward_date is not None)
def handle_forwarded_text_messages(message):
    """处理转发的文本消息"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    text = message.text.strip() if message.text else ""
    
    print(f"🔄 处理转发消息: {text[:50]}...")
    
    if text.startswith('yixiangjiqiren_') and len(text) == 35:
        process_file_code_silent(user_id, chat_id, text, message.message_id)
        return
    
    extracted_code = code_extractor.extract_code_from_text(text)
    
    if extracted_code:
        print(f"🔍 从转发消息中识别到文件码: {extracted_code}")
        process_file_code_silent(user_id, chat_id, extracted_code, message.message_id)
        return
    
    bot.send_message(
        chat_id,
        "💡 未在转发的消息中识别到有效文件码",
        reply_to_message_id=message.message_id,
        reply_markup=create_main_menu()
    )

# ==================== 管理员功能 ====================
def is_admin(user_id):
    """检查是否是管理员"""
    return user_id in ADMIN_USER_IDS
def check_if_package_management(message):
    """检查是否是套餐管理相关的消息"""
    if not message.text:
        return False
    
    user_id = message.from_user.id
    user_sessions = getattr(bot, 'user_sessions', {})
    session = user_sessions.get(user_id, {})
    
    if session and session.get('action'):
        action = session.get('action')
        package_actions = ['add_new_package', 'edit_package_price', 'edit_package_name', 
                          'edit_package_months', 'edit_package_description', 'edit_package_order']
        
        if action in package_actions:
            print(f"✅ [套餐检查] 用户 {user_id} 正在进行 {action} 操作")
            return True
    
    return False
@bot.message_handler(commands=['admin'])
def handle_admin_command(message):
    """管理员面板"""
    user_id = message.from_user.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足")
        return
    
    admin_text = """
👑 **管理员面板**

📊 **统计命令:**
/admin - 显示此面板
/stats - 系统统计信息
/users - 用户管理
/pending - 待处理订单

🔍 **用户管理:**
/userinfo <用户ID> - 查看用户详细信息
/searchuser <关键词> - 搜索用户
/userorders <用户ID> - 查看用户订单
/setvip <用户ID> <月数> - 设置VIP时长
/removevip <用户ID> - 移除VIP

🔍 **备注搜索:**
/searchremarks - 搜索所有用户备注

💰 **VIP套餐管理:**
/packages - 查看套餐
/setprice <套餐ID> <价格> - 修改价格

📢 **系统管理:**
/clearcache - 清除缓存
/restart - 重启机器人
"""
    
    bot.send_message(message.chat.id, admin_text, parse_mode='Markdown')

@bot.message_handler(commands=['stats'])
def handle_stats_command(message):
    """查看系统统计"""
    user_id = message.from_user.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足，此功能仅管理员可用")
        return
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    cursor.execute('SELECT COUNT(*) FROM packs')
    total_packs = cursor.fetchone()[0] or 0
    
    cursor.execute('SELECT SUM(file_count) FROM packs')
    total_files = cursor.fetchone()[0] or 0
    
    cursor.execute('SELECT COUNT(DISTINCT user_id) FROM packs')
    total_users = cursor.fetchone()[0] or 0
    
    cursor.execute('SELECT COUNT(*) FROM vip_users WHERE is_vip = 1')
    vip_users = cursor.fetchone()[0] or 0
    
    today = datetime.now().strftime('%Y-%m-%d')
    cursor.execute('SELECT SUM(decode_count), SUM(file_receive_count) FROM daily_stats WHERE date = ?', (today,))
    today_data = cursor.fetchone()
    today_decodes = today_data[0] or 0 if today_data else 0
    today_files = today_data[1] or 0 if today_data else 0
    
    cursor.execute('SELECT COUNT(*) FROM pack_remarks')
    total_remarks = cursor.fetchone()[0] or 0
    
    cursor.execute('''
    SELECT COUNT(*) FROM payment_orders 
    WHERE status = 'paid' AND activated = 0
    ''')
    pending_orders = cursor.fetchone()[0] or 0
    
    text = f"""
📊 **系统统计信息（仅管理员可见）**

📦 **文件统计：**
• 文件包总数：{total_packs} 个
• 总文件数量：{total_files} 个
• 总用户数：{total_users} 人
• VIP用户数：{vip_users} 人
• 备注总数：{total_remarks} 个

📈 **今日数据：**
• 解码次数：{today_decodes} 次
• 接收文件：{today_files} 个

💰 **订单状态：**
• 待处理订单：{pending_orders} 个

🌐 **支付配置：**
• 支付页面：{PAYMENT_BASE_URL}
• 状态：{'已配置' if 'your-domain.com' not in PAYMENT_BASE_URL else '需配置'}

⚙️ **系统状态：**
• 数据库连接：正常
• 工作线程：{task_processor.max_workers} 个
• 当前时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
"""
    
    markup = telebot.types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        telebot.types.InlineKeyboardButton("📋 查看待处理订单", callback_data="admin_pending_orders"),
        telebot.types.InlineKeyboardButton("👥 用户管理", callback_data="admin_users")
    )
    markup.add(
        telebot.types.InlineKeyboardButton("🔍 搜索所有备注", callback_data="admin_search_remarks"),
        telebot.types.InlineKeyboardButton("📊 系统监控", callback_data="admin_monitor")
    )
    
    bot.send_message(message.chat.id, text, parse_mode='Markdown', reply_markup=markup)

# ==================== 【新增】管理员备注搜索 ====================
@bot.message_handler(commands=['searchremarks'])
def handle_admin_search_remarks_command(message):
    """管理员搜索所有备注"""
    user_id = message.from_user.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足")
        return
    
    # 提取搜索关键词
    command_parts = message.text.split()
    if len(command_parts) < 2:
        text = """
🔍 **管理员备注搜索**

请输入搜索关键词：
• 搜索所有用户的备注和标签
• 支持模糊搜索
• 显示文件码、用户信息和备注

📌 **使用方法：**
`/searchremarks 关键词`

例如：
`/searchremarks 工作报告`
`/searchremarks 重要`
`/searchremarks 图片`

💡 **提示：**
• 搜索范围：所有用户
• 显示用户ID和备注信息
"""
        
        markup = telebot.types.InlineKeyboardMarkup()
        markup.add(
            telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
        )
        
        bot.send_message(message.chat.id, text, parse_mode='Markdown', reply_markup=markup)
        return
    
    keyword = " ".join(command_parts[1:]).strip()
    
    if len(keyword) < 1:
        bot.reply_to(message, "❌ 请输入有效的搜索关键词")
        return
    
    if len(keyword) > 50:
        bot.reply_to(message, "❌ 搜索关键词太长（最多50字）")
        return
    
    # 执行管理员搜索
    results = pack_remark_manager.search_all_remarks_admin(keyword)
    
    if not results:
        text = f"""
🔍 **管理员搜索结果：** "{keyword}"

❌ 未找到匹配的文件码

💡 提示：
1. 检查关键词是否正确
2. 尝试更短的关键词
3. 可能还没有用户添加备注
"""
    else:
        text = f"""
🔍 **管理员搜索结果：** "{keyword}"

✅ 找到 {len(results)} 个匹配的文件码：

"""
        
        for i, result in enumerate(results[:15], 1):  # 最多显示15个
            # 截断长备注
            remark_display = result['remark']
            if len(remark_display) > 30:
                remark_display = remark_display[:27] + "..."
            
            text += f"\n**{i}. ** `{result['code']}`\n"
            text += f"├─ 📝 {remark_display}\n"
            if result.get('tags'):
                text += f"├─ 🏷️ 标签：{result['tags']}\n"
            text += f"├─ 用户ID：`{result['user_id']}`\n"
            text += f"├─ 文件：{result['file_count']}个\n"
            text += f"└─ 类型：{result['file_types']}\n"
        
        if len(results) > 15:
            text += f"\n... 还有 {len(results)-15} 个结果未显示"
        
        text += "\n💡 使用 `/userinfo 用户ID` 查看用户详情"
    
    bot.send_message(message.chat.id, text, parse_mode='Markdown')

# ==================== 用户管理命令 ====================

@bot.message_handler(commands=['userinfo'])
def handle_userinfo_command(message):
    """查看用户详细信息"""
    user_id = message.from_user.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足")
        return
    
    command_parts = message.text.split()
    if len(command_parts) < 2:
        bot.reply_to(message, "📌 使用方法: /userinfo <用户ID>\n例如: /userinfo 123456789")
        return
    
    try:
        target_user_id = int(command_parts[1])
    except ValueError:
        bot.reply_to(message, "❌ 用户ID格式错误")
        return
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 1. 获取用户基本信息
    cursor.execute('SELECT * FROM vip_users WHERE user_id = ?', (target_user_id,))
    vip_info = cursor.fetchone()
    
    # 2. 获取文件统计
    cursor.execute('''
    SELECT COUNT(*) as pack_count, 
           SUM(file_count) as total_files,
           MIN(created_at) as first_upload,
           MAX(created_at) as last_upload
    FROM packs 
    WHERE user_id = ?
    ''', (target_user_id,))
    
    file_stats = cursor.fetchone()
    
    # 3. 获取使用统计
    cursor.execute('''
    SELECT SUM(decode_count) as total_decodes,
           SUM(file_receive_count) as total_receives
    FROM daily_stats 
    WHERE user_id = ?
    ''', (target_user_id,))
    
    usage_stats = cursor.fetchone()
    
    # 4. 获取订单记录
    cursor.execute('''
    SELECT COUNT(*) as order_count,
           SUM(amount) as total_spent
    FROM payment_orders 
    WHERE user_id = ? AND status = 'activated'
    ''', (target_user_id,))
    
    order_stats = cursor.fetchone()
    
    # 5. 获取备注数量
    cursor.execute('SELECT COUNT(*) FROM pack_remarks WHERE user_id = ?', (target_user_id,))
    remark_count = cursor.fetchone()[0] or 0
    
    # 构建用户信息文本
    text = f"👤 **用户信息**\n\n"
    
    # 尝试获取用户名
    username = "未知用户"
    try:
        user = bot.get_chat(target_user_id)
        username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
    except:
        pass
    
    text += f"**用户名:** {username}\n"
    text += f"**用户ID:** `{target_user_id}`\n\n"
    
    # VIP信息
    if vip_info:
        is_vip = vip_info[1]  # is_vip字段
        vip_expire = vip_info[2]  # vip_expire字段
        total_spent = vip_info[3] or 0  # total_spent字段
        
        if is_vip and vip_expire:
            expire_date = datetime.fromisoformat(vip_expire)
            days_left = (expire_date - datetime.now()).days
            
            text += f"⭐ **VIP状态:** 已激活\n"
            text += f"📅 **到期时间:** {expire_date.strftime('%Y-%m-%d %H:%M:%S')}\n"
            text += f"⏰ **剩余天数:** {days_left}天\n"
        elif is_vip and not vip_expire:
            text += f"⭐ **VIP状态:** 永久VIP\n"
        else:
            text += f"🔓 **VIP状态:** 普通用户\n"
        
        text += f"💰 **累计消费:** {total_spent:.2f}元\n\n"
    else:
        text += f"🔓 **VIP状态:** 普通用户\n\n"
    
    # 文件统计
    if file_stats and file_stats[0]:
        pack_count = file_stats[0] or 0
        total_files = file_stats[1] or 0
        first_upload = file_stats[2]
        last_upload = file_stats[3]
        
        text += f"📦 **文件统计:**\n"
        text += f"├ 文件包数: {pack_count}个\n"
        text += f"├ 总文件数: {total_files}个\n"
        
        if first_upload:
            try:
                if isinstance(first_upload, str):
                    first_str = first_upload[:16].replace('T', ' ')
                else:
                    first_str = first_upload.strftime('%Y-%m-%d %H:%M')
                text += f"├ 首次上传: {first_str}\n"
            except:
                pass
        
        if last_upload:
            try:
                if isinstance(last_upload, str):
                    last_str = last_upload[:16].replace('T', ' ')
                else:
                    last_str = last_upload.strftime('%Y-%m-%d %H:%M')
                text += f"└ 最后上传: {last_str}\n"
            except:
                pass
        
        text += "\n"
    
    # 使用统计
    if usage_stats:
        total_decodes = usage_stats[0] or 0
        total_receives = usage_stats[1] or 0
        
        if total_decodes > 0 or total_receives > 0:
            text += f"📊 **使用统计:**\n"
            text += f"├ 总解码次数: {total_decodes}次\n"
            text += f"└ 总接收文件: {total_receives}个\n\n"
    
    # 订单统计
    if order_stats:
        order_count = order_stats[0] or 0
        total_spent = order_stats[1] or 0
        
        if order_count > 0:
            text += f"💰 **订单统计:**\n"
            text += f"├ 订单数量: {order_count}个\n"
            text += f"└ 总消费额: {total_spent:.2f}元\n\n"
    
    # 备注统计
    if remark_count > 0:
        text += f"📝 **备注统计:**\n"
        text += f"└ 备注数量: {remark_count}个\n\n"
    
    # 查询时间
    text += f"⏰ **查询时间:** {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}"
    
    bot.reply_to(message, text, parse_mode='Markdown')

@bot.message_handler(commands=['searchuser'])
def handle_searchuser_command(message):
    """搜索用户"""
    user_id = message.from_user.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足")
        return
    
    command_parts = message.text.split()
    if len(command_parts) < 2:
        bot.reply_to(message, "📌 使用方法: /searchuser <关键词>\n可以搜索用户名或用户ID的部分内容")
        return
    
    keyword = " ".join(command_parts[1:]).strip()
    
    if len(keyword) < 2:
        bot.reply_to(message, "❌ 关键词太短，至少2个字符")
        return
    
    # 搜索用户（通过获取的用户信息）
    # 注意：Telegram API 不提供搜索用户的功能
    # 我们只能搜索我们数据库中的用户
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 搜索用户ID中包含关键词的
    try:
        if keyword.isdigit():
            # 如果是纯数字，尝试按用户ID搜索
            cursor.execute('''
            SELECT DISTINCT p.user_id, 
                   COUNT(p.id) as pack_count,
                   SUM(p.file_count) as total_files
            FROM packs p
            WHERE p.user_id LIKE ?
            GROUP BY p.user_id
            ORDER BY total_files DESC
            LIMIT 10
            ''', (f'%{keyword}%',))
            
            users = cursor.fetchall()
        else:
            # 如果是文本，尝试搜索用户（需要先获取用户信息）
            users = []
            bot.reply_to(message, "🔍 正在搜索用户...")
    except Exception as e:
        bot.reply_to(message, f"❌ 搜索失败: {e}")
        return
    
    if not users:
        text = f"🔍 未找到匹配的用户\n\n关键词: `{keyword}`\n\n💡 提示:\n1. 用户需要上传过文件才能被搜索到\n2. 只能搜索用户ID\n3. 尝试使用 /userinfo <用户ID> 查看特定用户"
    else:
        text = f"🔍 **用户搜索结果** (关键词: `{keyword}`)\n\n"
        
        for i, user_data in enumerate(users, 1):
            user_id_db = user_data[0]
            pack_count = user_data[1] or 0
            total_files = user_data[2] or 0
            
            # 获取用户名
            username = "未知用户"
            try:
                user = bot.get_chat(user_id_db)
                username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
            except:
                pass
            
            # 检查VIP状态
            cursor.execute('SELECT is_vip FROM vip_users WHERE user_id = ?', (user_id_db,))
            vip_info = cursor.fetchone()
            vip_status = "⭐" if vip_info and vip_info[0] else "👤"
            
            text += f"**{i}. {vip_status} {username}**\n"
            text += f"├ ID: `{user_id_db}`\n"
            text += f"├ 文件包: {pack_count}个\n"
            text += f"└ 总文件: {total_files}个\n\n"
        
        text += "💡 使用 /userinfo <用户ID> 查看详细信息"
    
    bot.reply_to(message, text, parse_mode='Markdown')

@bot.message_handler(commands=['userorders'])
def handle_userorders_command(message):
    """查看用户订单"""
    user_id = message.from_user.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足")
        return
    
    command_parts = message.text.split()
    if len(command_parts) < 2:
        bot.reply_to(message, "📌 使用方法: /userorders <用户ID>\n例如: /userorders 123456789")
        return
    
    try:
        target_user_id = int(command_parts[1])
    except ValueError:
        bot.reply_to(message, "❌ 用户ID格式错误")
        return
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 获取用户订单
    cursor.execute('''
    SELECT po.order_id, po.amount, po.payment_method, 
           po.status, po.created_at, po.payment_time, po.activated_time,
           vp.name, vp.months
    FROM payment_orders po
    JOIN vip_packages vp ON po.package_id = vp.id
    WHERE po.user_id = ?
    ORDER BY po.created_at DESC
    LIMIT 20
    ''', (target_user_id,))
    
    orders = cursor.fetchall()
    
    # 获取用户名
    username = "未知用户"
    try:
        user = bot.get_chat(target_user_id)
        username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
    except:
        pass
    
    if not orders:
        text = f"📭 **用户订单记录**\n\n👤 用户: {username} (ID: `{target_user_id}`)\n\n❌ 暂无订单记录"
    else:
        text = f"📋 **用户订单记录**\n\n👤 用户: {username} (ID: `{target_user_id}`)\n\n"
        
        for i, order in enumerate(orders, 1):
            order_id = order[0]
            amount = order[1]
            payment_method = order[2]
            status = order[3]
            created_at = order[4]
            payment_time = order[5]
            activated_time = order[6]
            package_name = order[7]
            months = order[8]
            
            # 支付方式名称
            method_name = VIPPackageConfig.PAYMENT_METHODS.get(payment_method, {}).get("name", payment_method)
            
            # 状态文本
            status_text = {
                'pending': '⏳ 待支付',
                'paid': '✅ 已支付',
                'activated': '⭐ 已激活',
                'cancelled': '❌ 已取消'
            }.get(status, status)
            
            text += f"**{i}. {status_text} - {package_name}**\n"
            text += f"├ 订单号: `{order_id}`\n"
            text += f"├ 金额: {amount}元\n"
            text += f"├ 时长: {months}个月\n"
            text += f"├ 支付方式: {method_name}\n"
            
            # 格式化时间
            if created_at:
                try:
                    if isinstance(created_at, str):
                        create_str = created_at[:16].replace('T', ' ')
                    else:
                        create_str = created_at.strftime('%Y-%m-%d %H:%M')
                    text += f"├ 创建: {create_str}\n"
                except:
                    pass
            
            if payment_time:
                try:
                    if isinstance(payment_time, str):
                        pay_str = payment_time[:16].replace('T', ' ')
                    else:
                        pay_str = payment_time.strftime('%Y-%m-%d %H:%M')
                    text += f"├ 支付: {pay_str}\n"
                except:
                    pass
            
            if activated_time:
                try:
                    if isinstance(activated_time, str):
                        activate_str = activated_time[:16].replace('T', ' ')
                    else:
                        activate_str = activated_time.strftime('%Y-%m-%d %H:%M')
                    text += f"└ 激活: {activate_str}\n"
                except:
                    pass
            
            text += "\n"
    
    bot.reply_to(message, text, parse_mode='Markdown')

@bot.message_handler(commands=['setvip'])
def handle_setvip_command(message):
    """设置用户VIP"""
    user_id = message.from_user.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足")
        return
    
    command_parts = message.text.split()
    if len(command_parts) < 3:
        bot.reply_to(message, "📌 使用方法: /setvip <用户ID> <月数>\n例如: /setvip 123456789 6")
        return
    
    try:
        target_user_id = int(command_parts[1])
        months = int(command_parts[2])
    except ValueError:
        bot.reply_to(message, "❌ 参数格式错误，请使用数字")
        return
    
    # 月份限制
    if months <= 0:
        months = 1
    elif months > 36:
        months = 36
    
    try:
        # 调用VIP激活函数
        expire_time = vip_user_manager.activate_vip(target_user_id, months)
        days_left = (expire_time - datetime.now()).days
        
        text = f"""
✅ **VIP设置成功！**

👤 **用户ID:** `{target_user_id}`
📅 **时长设置:** {months}个月
⏰ **到期时间:** {expire_time.strftime('%Y-%m-%d %H:%M:%S')}
📊 **剩余天数:** {days_left}天
⚡ **操作时间:** {datetime.now().strftime('%H:%M:%S')}

💡 VIP特权已立即生效！
"""
        
        # 通知用户
        notify_user_vip_activated(target_user_id, months)
        
    except Exception as e:
        text = f"❌ VIP设置失败: {e}"
    
    bot.reply_to(message, text, parse_mode='Markdown')

@bot.message_handler(commands=['removevip'])
def handle_removevip_command(message):
    """移除用户VIP"""
    user_id = message.from_user.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足")
        return
    
    command_parts = message.text.split()
    if len(command_parts) < 2:
        bot.reply_to(message, "📌 使用方法: /removevip <用户ID>\n例如: /removevip 123456789")
        return
    
    try:
        target_user_id = int(command_parts[1])
    except ValueError:
        bot.reply_to(message, "❌ 用户ID格式错误")
        return
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    try:
        # 检查用户是否存在
        cursor.execute('SELECT is_vip FROM vip_users WHERE user_id = ?', (target_user_id,))
        vip_info = cursor.fetchone()
        
        if not vip_info:
            text = f"❌ 用户 `{target_user_id}` 不是VIP用户"
        else:
            # 移除VIP
            cursor.execute('''
            UPDATE vip_users 
            SET is_vip = 0, vip_expire = NULL, updated_at = ?
            WHERE user_id = ?
            ''', (datetime.now().isoformat(), target_user_id))
            
            conn.commit()
            
            # 清除缓存
            cache_key = f"vip_{target_user_id}"
            if cache_key in user_limit_manager.cache:
                del user_limit_manager.cache[cache_key]
            
            text = f"""
✅ **VIP已移除！**

👤 **用户ID:** `{target_user_id}`
⚡ **操作时间:** {datetime.now().strftime('%H:%M:%S')}

💡 用户已恢复为普通用户权限
"""
            
            # 获取用户名
            username = "未知用户"
            try:
                user = bot.get_chat(target_user_id)
                username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
            except:
                pass
            
            # 通知用户
            try:
                bot.send_message(
                    target_user_id,
                    f"⚠️ **VIP状态变更通知**\n\n您的VIP特权已被管理员移除。\n如有疑问，请联系客服。",
                    parse_mode='@vdhfjd'
                )
            except:
                pass
    
    except Exception as e:
        text = f"❌ VIP移除失败: {e}"
        if conn:
            conn.rollback()
    
    bot.reply_to(message, text, parse_mode='Markdown')
@bot.message_handler(commands=['flushcache'])
def handle_flushcache_command(message):
    """手动刷新缓存到数据库"""
    user_id = message.from_user.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足")
        return
    
    if hasattr(decode_cache, 'flush'):
        count = decode_cache.flush()
        bot.reply_to(message, f"✅ 缓存已刷新到数据库\n刷新了 {count} 个文件码的解码次数")
    else:
        bot.reply_to(message, "❌ 缓存系统未启用")    
@bot.message_handler(commands=['packages'])
def handle_packages_command(message):
    """管理VIP套餐"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足")
        return
    
    vip_payment_system.show_package_management(user_id, chat_id)

@bot.message_handler(commands=['setprice'])
def handle_setprice_command(message):
    """快速修改套餐价格"""
    user_id = message.from_user.id
    
    if not is_admin(user_id):
        bot.reply_to(message, "❌ 权限不足")
        return
    
    command_parts = message.text.split()
    if len(command_parts) < 3:
        bot.reply_to(
            message,
            "📌 使用方法: /setprice <套餐ID> <新价格>\n"
            "例如: /setprice 1 25.00\n\n"
            "查看套餐ID使用: /packages"
        )
        return
    
    try:
        package_id = int(command_parts[1])
        new_price = float(command_parts[2])
        
        if new_price <= 0:
            bot.reply_to(message, "❌ 价格必须大于0")
            return
        
        success, msg = vip_payment_system.package_manager.update_package_price(package_id, new_price)
        bot.reply_to(message, msg)
        
    except ValueError:
        bot.reply_to(message, "❌ 参数格式错误！套餐ID必须是整数，价格必须是数字")
    except Exception as e:
        bot.reply_to(message, f"❌ 修改失败: {e}")
# ==================== 批量相关回调处理 ====================
@bot.callback_query_handler(func=lambda call: call.data.startswith('batch_'))
def handle_batch_callbacks(call):
    """处理批量相关回调"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    if call.data.startswith('batch_next_'):
        session_id = call.data.replace('batch_next_', '')
        handle_batch_next_page(call, session_id)
        
    elif call.data.startswith('batch_info_'):
        session_id = call.data.replace('batch_info_', '')
        show_batch_detail_info(call, session_id)
        
    elif call.data.startswith('batch_detail_'):
        session_id = call.data.replace('batch_detail_', '')
        show_batch_full_detail(call, session_id)
        
    elif call.data.startswith('batch_complete_'):
        session_id = call.data.replace('batch_complete_', '')
        complete_batch_session(call, session_id)
    
    bot.answer_callback_query(call.id)

def show_batch_detail_info(call, session_id):
    """显示批次详情信息"""
    try:
        user_id = call.from_user.id
        chat_id = call.message.chat.id
        message_id = call.message.message_id
        
        session = smart_batch_sender.get_session_info(session_id)
        if not session:
            bot.answer_callback_query(call.id, "❌ 会话已过期", show_alert=True)
            return
        
        # 构建详情信息
        text = f"""
📋 **批量详情信息**

📦 **批量会话ID:** `{session_id}`
👤 **用户ID:** `{session['user_id']}`
📊 **总计:** {session['total_files']} 个文件
📦 **总批次:** {session['total_batches']} 批
🔢 **文件码数量:** {session['codes_count']} 个

📈 **当前进度:**
• 已发送: {session['current_batch']}/{session['total_batches']} 批
• 剩余文件: {sum(len(b) for b in session['batches'][session['current_batch']:])} 个

🗂️ **包含的文件码:**
"""
        
        # 显示前5个文件码
        for i, code_detail in enumerate(session['code_details'][:5], 1):
            text += f"{i}. `{code_detail['code']}` - {code_detail['file_count']}个文件 ({code_detail['file_types']})\n"
        
        if len(session['code_details']) > 5:
            text += f"... 还有 {len(session['code_details']) - 5} 个文件码\n"
        
        # 添加时间信息
        created_time = datetime.fromtimestamp(session['created_at'])
        text += f"\n⏰ **创建时间:** {created_time.strftime('%Y-%m-%d %H:%M:%S')}"
        
        markup = telebot.types.InlineKeyboardMarkup()
        markup.add(
            telebot.types.InlineKeyboardButton("⬅️ 返回", callback_data=f"batch_next_{session_id}"),
            telebot.types.InlineKeyboardButton("🏠 首页", callback_data="back_to_main")
        )
        
        # 尝试编辑消息，如果失败则发送新消息
        try:
            bot.edit_message_text(
                text,
                chat_id=chat_id,
                message_id=message_id,
                parse_mode='Markdown',
                reply_markup=markup
            )
        except:
            bot.send_message(
                chat_id,
                text,
                parse_mode='Markdown',
                reply_markup=markup
            )
        
        bot.answer_callback_query(call.id)
        
    except Exception as e:
        print(f"显示批次详情出错: {e}")
        bot.answer_callback_query(
            call.id,
            f"❌ 获取详情失败: {str(e)}",
            show_alert=True
        )

def show_batch_full_detail(call, session_id):
    """显示批次完整详情"""
    show_batch_detail_info(call, session_id)

def complete_batch_session(call, session_id):
    """完成批量会话"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    session = smart_batch_sender.get_session_info(session_id)
    if session:
        smart_batch_sender.clear_session(session_id)
    
    bot.answer_callback_query(call.id, "✅ 批量会话已结束")
    
    try:
        bot.delete_message(chat_id, call.message.message_id)
    except:
        pass
    
    bot.send_message(
        chat_id,
        "🎉 批量处理完成！已返回主菜单",
        reply_markup=create_main_menu()
    )

def handle_batch_next_page(call, session_id):
    """处理批量下一页"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    # 检查权限
    if user_limit_manager.is_vip(user_id):
        can_click = True
        wait_time = 0
    else:
        session = smart_batch_sender.get_session_info(session_id)
        if session:
            current_time = time.time()
            time_since_last = current_time - session.get('last_send_time', 0)
            
            if time_since_last < 5:
                wait_time = math.ceil(5 - time_since_last)
                bot.answer_callback_query(
                    call.id,
                    f"⏳ 请等待 {wait_time} 秒",
                    show_alert=True
                )
                return
            else:
                can_click = True
                wait_time = 0
    
    # 获取下一批文件
    batch_info = smart_batch_sender.get_next_batch(session_id)
    
    if not batch_info:
        bot.answer_callback_query(call.id, "✅ 所有文件已发送完成")
        return
    
    # 删除原消息
    try:
        bot.delete_message(chat_id, call.message.message_id)
    except:
        pass
    
    # 发送文件
    if send_files_compact(chat_id, batch_info['files']):
        # 更新文件接收计数
        for file_info in batch_info['files']:
            user_limit_manager.increment_file_receive_count(user_id, 1, file_info['file_type'])
        
        # 显示新的分页控制
        if batch_info['is_last']:
            # 最后一批
            completion_text = f"""
🎉 **批量发送完成！**

✅ **批量处理统计：**
• 文件码数量：{batch_info['codes_count']} 个
• 总文件数：{batch_info['total_files']} 个
• 发送批次：{batch_info['total_batches']} 批

📊 **今日统计：**
• 解码次数：+{batch_info['codes_count']}
• 接收文件：+{batch_info['total_files']}
"""
            
            bot.send_message(
                chat_id,
                completion_text,
                parse_mode='Markdown',
                reply_markup=create_main_menu()
            )
            
            smart_batch_sender.clear_session(session_id)
        else:
            # 还有更多批次
            show_batch_pagination(user_id, chat_id, session_id, batch_info)
    else:
        bot.send_message(
            chat_id,
            "❌ 文件发送失败",
            reply_markup=create_main_menu()
        )
        smart_batch_sender.clear_session(session_id)

# ==================== 用户管理回调函数 ====================
def handle_admin_monitor_callback(call):
    """处理系统监控回调"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    # 获取系统信息
    import platform
    
    try:
        # CPU使用率
        cpu_percent = psutil.cpu_percent(interval=1)
        
        # 内存使用情况
        memory = psutil.virtual_memory()
        memory_total_gb = memory.total / (1024**3)
        memory_used_gb = memory.used / (1024**3)
        memory_percent = memory.percent
        
        # 磁盘使用情况
        disk = psutil.disk_usage('/')
        disk_total_gb = disk.total / (1024**3)
        disk_used_gb = disk.used / (1024**3)
        disk_percent = disk.percent
        
        # 进程信息
        process = psutil.Process()
        process_memory_mb = process.memory_info().rss / (1024**2)
        process_cpu_percent = process.cpu_percent(interval=0.1)
        
        # 数据库统计
        conn = db_pool.get_connection()
        cursor = conn.cursor()
        
        # 用户统计
        cursor.execute('SELECT COUNT(DISTINCT user_id) FROM packs')
        total_users = cursor.fetchone()[0] or 0
        
        cursor.execute('SELECT COUNT(*) FROM vip_users WHERE is_vip = 1 AND vip_expire > datetime("now")')
        active_vip = cursor.fetchone()[0] or 0
        
        # 文件统计
        cursor.execute('SELECT COUNT(*) FROM packs')
        total_packs = cursor.fetchone()[0] or 0
        
        cursor.execute('SELECT SUM(file_count) FROM packs')
        total_files = cursor.fetchone()[0] or 0
        
        # 订单统计
        cursor.execute("SELECT COUNT(*) FROM payment_orders WHERE status = 'pending'")
        pending_orders = cursor.fetchone()[0] or 0
        
        cursor.execute("SELECT COUNT(*) FROM payment_orders WHERE status = 'paid' AND activated = 0")
        paid_not_activated = cursor.fetchone()[0] or 0
        
        # 会话统计
        active_sessions = len(session_mgr.user_sessions) if hasattr(session_mgr, 'user_sessions') else 0
        batch_sessions = len(smart_batch_sender.user_sessions) if hasattr(smart_batch_sender, 'user_sessions') else 0
        
        # 线程信息
        active_threads = threading.active_count()
        
        # 构建监控信息
        text = f"""
📊 **系统实时监控**

🖥️ **服务器信息**
• 系统：{platform.system()} {platform.release()}
• Python：{platform.python_version()}
• 运行时间：{get_uptime()}

⚙️ **硬件资源**
• CPU使用率：{cpu_percent:.1f}%
• 内存：{memory_used_gb:.1f}GB / {memory_total_gb:.1f}GB ({memory_percent:.1f}%)
• 磁盘：{disk_used_gb:.1f}GB / {disk_total_gb:.1f}GB ({disk_percent:.1f}%)

🤖 **机器人进程**
• 进程内存：{process_memory_mb:.1f} MB
• 进程CPU：{process_cpu_percent:.1f}%
• 活跃线程：{active_threads} 个

👥 **用户统计**
• 总用户数：{total_users} 人
• 活跃VIP：{active_vip} 人
• 文件包数：{total_packs} 个
• 总文件数：{total_files} 个

💰 **订单状态**
• 待支付订单：{pending_orders} 个
• 待激活订单：{paid_not_activated} 个

🔄 **会话状态**
• 上传会话：{active_sessions} 个
• 批量会话：{batch_sessions} 个

⏰ **查询时间**：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

💡 **提示**：点击刷新按钮更新实时数据
"""
        
    except ImportError:
        # 如果没有psutil，显示简化的信息
        text = """
📊 **系统监控（基础版）**

⚠️ **缺少依赖**
请安装 psutil 获取完整系统监控：
`pip install psutil`

📈 **当前可用统计：**
• 数据库连接池：正常
• 工作线程：200 个
• 当前时间：{}

🔄 **手动刷新查看最新数据**
""".format(datetime.now().strftime('%Y-%m-%d %H:%M:%S'))
    
    except Exception as e:
        text = f"""
❌ **监控数据获取失败**

错误信息：{str(e)}

💡 **建议：**
1. 检查服务器状态
2. 检查数据库连接
3. 重启机器人尝试
"""
    
    # 创建按钮
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("🔄 刷新监控", callback_data="admin_monitor"),
        telebot.types.InlineKeyboardButton("📊 系统统计", callback_data="back_to_stats")
    )
    markup.row(
        telebot.types.InlineKeyboardButton("📋 待处理订单", callback_data="admin_pending_orders"),
        telebot.types.InlineKeyboardButton("👥 用户管理", callback_data="admin_users")
    )
    markup.row(
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    # 编辑或发送消息
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    bot.answer_callback_query(call.id)

def get_uptime():
    """获取机器人运行时间"""
    try:
        import time as ttime
        uptime_seconds = ttime.time() - psutil.boot_time()
        
        days = int(uptime_seconds // (24 * 3600))
        hours = int((uptime_seconds % (24 * 3600)) // 3600)
        minutes = int((uptime_seconds % 3600) // 60)
        
        if days > 0:
            return f"{days}天{hours}小时{minutes}分钟"
        elif hours > 0:
            return f"{hours}小时{minutes}分钟"
        else:
            return f"{minutes}分钟"
    except:
        return "未知"
    
@bot.callback_query_handler(func=lambda call: call.data == "admin_users")
def handle_admin_users_callback(call):
    """显示用户管理菜单"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    # 获取用户统计信息
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 1. 总用户数（上传过文件的用户）
    cursor.execute('SELECT COUNT(DISTINCT user_id) FROM packs')
    total_upload_users = cursor.fetchone()[0] or 0
    
    # 2. 活跃VIP用户（未过期）
    cursor.execute('SELECT COUNT(*) FROM vip_users WHERE is_vip = 1 AND vip_expire > datetime("now")')
    active_vip = cursor.fetchone()[0] or 0
    
    # 3. 已过期VIP用户
    cursor.execute('SELECT COUNT(*) FROM vip_users WHERE is_vip = 1 AND vip_expire <= datetime("now")')
    expired_vip = cursor.fetchone()[0] or 0
    
    # 4. 普通用户（非VIP）
    regular_users = max(0, total_upload_users - (active_vip + expired_vip))
    
    # 5. 今日活跃用户
    today = datetime.now().strftime('%Y-%m-%d')
    cursor.execute('SELECT COUNT(DISTINCT user_id) FROM daily_stats WHERE date = ?', (today,))
    today_active = cursor.fetchone()[0] or 0
    
    # 6. 总文件包数
    cursor.execute('SELECT COUNT(*) FROM packs')
    total_packs = cursor.fetchone()[0] or 0
    
    # 7. 总文件数
    cursor.execute('SELECT SUM(file_count) FROM packs')
    total_files = cursor.fetchone()[0] or 0
    
    # 8. 今日解码次数
    cursor.execute('SELECT SUM(decode_count) FROM daily_stats WHERE date = ?', (today,))
    today_decodes = cursor.fetchone()[0] or 0
    
    # 9. 今日接收文件数
    cursor.execute('SELECT SUM(file_receive_count) FROM daily_stats WHERE date = ?', (today,))
    today_files = cursor.fetchone()[0] or 0
    
    # 10. 备注总数
    cursor.execute('SELECT COUNT(*) FROM pack_remarks')
    total_remarks = cursor.fetchone()[0] or 0
    
    text = f"""
👥 **用户管理面板**

📊 **用户统计：**
├ 总用户数：{total_upload_users} 人
├ VIP会员：{active_vip} 人
├ 过期VIP：{expired_vip} 人
├ 普通用户：{regular_users} 人
└ 今日活跃：{today_active} 人

📦 **文件统计：**
├ 总文件包：{total_packs} 个
├ 总文件数：{total_files} 个
├ 今日解码：{today_decodes} 次
└ 今日接收：{today_files} 个

📝 **备注统计：**
└ 总备注数：{total_remarks} 个

⏰ **查询时间：** {datetime.now().strftime("%Y-%m-%d %H:%M:%S")}

🔍 **用户管理命令：**
• `/userinfo <用户ID>` - 查看用户详细信息
• `/searchuser <关键词>` - 搜索用户名或ID
• `/userorders <用户ID>` - 查看用户订单
• `/setvip <用户ID> <月数>` - 设置VIP时长
• `/removevip <用户ID>` - 移除VIP
"""

    markup = telebot.types.InlineKeyboardMarkup(row_width=2)
    
    # 添加功能按钮
    markup.add(
        telebot.types.InlineKeyboardButton("👑 VIP用户列表", callback_data="admin_vip_list"),
        telebot.types.InlineKeyboardButton("👤 普通用户列表", callback_data="admin_normal_list")
    )
    markup.add(
        telebot.types.InlineKeyboardButton("📈 活跃用户榜", callback_data="admin_active_users"),
        telebot.types.InlineKeyboardButton("📦 文件排行", callback_data="admin_file_ranking")
    )
    markup.add(
        telebot.types.InlineKeyboardButton("🔄 刷新统计", callback_data="admin_users"),
        telebot.types.InlineKeyboardButton("⬅️ 返回", callback_data="back_to_stats")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except Exception as e:
        print(f"更新用户管理面板失败: {e}")
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    bot.answer_callback_query(call.id)

# ==================== 【修改点1：VIP用户列表添加移除按钮】====================
@bot.callback_query_handler(func=lambda call: call.data == "admin_vip_list")
def handle_admin_vip_list(call):
    """显示VIP用户列表"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 获取VIP用户列表（按到期时间排序）
    cursor.execute('''
    SELECT vu.user_id, vu.vip_expire, vu.total_spent, 
           vu.created_at, vu.updated_at,
           (SELECT COUNT(*) FROM packs WHERE user_id = vu.user_id) as pack_count,
           (SELECT SUM(file_count) FROM packs WHERE user_id = vu.user_id) as total_files
    FROM vip_users vu
    WHERE vu.is_vip = 1
    ORDER BY vu.vip_expire DESC
    LIMIT 30
    ''')
    
    vip_users = cursor.fetchall()
    
    if not vip_users:
        text = "📭 暂无VIP用户"
    else:
        text = f"👑 **VIP用户列表** (共{len(vip_users)}人)\n\n"
        
        for i, vip_user in enumerate(vip_users, 1):
            user_id_db = vip_user[0]
            vip_expire = vip_user[1]
            total_spent = vip_user[2] or 0
            created_at = vip_user[3]
            updated_at = vip_user[4]
            pack_count = vip_user[5] or 0
            total_files = vip_user[6] or 0
            
            # 获取用户名
            username = "未知用户"
            try:
                user = bot.get_chat(user_id_db)
                username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
            except:
                pass
            
            # 计算剩余天数
            days_left = "未知"
            if vip_expire:
                try:
                    expire_date = datetime.fromisoformat(vip_expire)
                    days = (expire_date - datetime.now()).days
                    days_left = f"{days}天" if days > 0 else "已过期"
                except:
                    days_left = "永久"
            
            text += f"**{i}. {username}**\n"
            text += f"├ ID: `{user_id_db}`\n"
            text += f"├ 文件包: {pack_count}个\n"
            text += f"├ 总文件: {total_files}个\n"
            text += f"├ 消费: {total_spent:.2f}元\n"
            text += f"├ 到期: {vip_expire[:10] if vip_expire else '无'}\n"
            text += f"└ 剩余: {days_left}\n\n"
    
    markup = telebot.types.InlineKeyboardMarkup(row_width=2)
    
    # 添加移除VIP按钮
    if vip_users:
        for i, vip_user in enumerate(vip_users, 1):
            user_id_db = vip_user[0]
            markup.add(
                telebot.types.InlineKeyboardButton(
                    f"🗑️ 移除{i}", 
                    callback_data=f"admin_remove_vip_{user_id_db}"
                )
            )
    
    markup.add(
        telebot.types.InlineKeyboardButton("⬅️ 返回用户管理", callback_data="admin_users"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    bot.answer_callback_query(call.id)

@bot.callback_query_handler(func=lambda call: call.data == "admin_normal_list")
def handle_admin_normal_list(call):
    """显示普通用户列表"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 获取普通用户列表（按文件数量排序）
    cursor.execute('''
    SELECT DISTINCT p.user_id, 
           COUNT(p.id) as pack_count,
           SUM(p.file_count) as total_files,
           MAX(p.created_at) as last_upload
    FROM packs p
    WHERE p.user_id NOT IN (
        SELECT user_id FROM vip_users WHERE is_vip = 1 AND vip_expire > datetime("now")
    )
    GROUP BY p.user_id
    ORDER BY total_files DESC
    LIMIT 30
    ''')
    
    normal_users = cursor.fetchall()
    
    if not normal_users:
        text = "📭 暂无普通用户数据"
    else:
        text = f"👤 **普通用户列表** (共{len(normal_users)}人)\n\n"
        
        for i, normal_user in enumerate(normal_users, 1):
            user_id_db = normal_user[0]
            pack_count = normal_user[1] or 0
            total_files = normal_user[2] or 0
            last_upload = normal_user[3]
            
            # 获取用户名
            username = "未知用户"
            try:
                user = bot.get_chat(user_id_db)
                username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
            except:
                pass
            
            # 格式化最后上传时间
            last_upload_str = "无记录"
            if last_upload:
                try:
                    if isinstance(last_upload, str):
                        last_upload_str = last_upload[:16].replace('T', ' ')
                    else:
                        last_upload_str = last_upload.strftime('%Y-%m-%d %H:%M')
                except:
                    last_upload_str = str(last_upload)[:16]
            
            text += f"**{i}. {username}**\n"
            text += f"├ ID: `{user_id_db}`\n"
            text += f"├ 文件包: {pack_count}个\n"
            text += f"├ 总文件: {total_files}个\n"
            text += f"└ 最后上传: {last_upload_str}\n\n"
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("⬅️ 返回用户管理", callback_data="admin_users"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    bot.answer_callback_query(call.id)

@bot.callback_query_handler(func=lambda call: call.data == "admin_active_users")
def handle_admin_active_users(call):
    """显示活跃用户榜"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 获取最近30天活跃用户（按解码次数排序）
    thirty_days_ago = (datetime.now() - timedelta(days=30)).strftime('%Y-%m-%d')
    
    cursor.execute('''
    SELECT user_id, 
           SUM(decode_count) as total_decodes,
           SUM(file_receive_count) as total_receives,
           MAX(date) as last_active
    FROM daily_stats
    WHERE date >= ?
    GROUP BY user_id
    ORDER BY total_decodes DESC
    LIMIT 20
    ''', (thirty_days_ago,))
    
    active_users = cursor.fetchall()
    
    if not active_users:
        text = "📭 暂无活跃用户数据"
    else:
        text = "🏆 **活跃用户榜** (最近30天)\n\n"
        
        for i, active_user in enumerate(active_users, 1):
            user_id_db = active_user[0]
            total_decodes = active_user[1] or 0
            total_receives = active_user[2] or 0
            last_active = active_user[3]
            
            # 获取用户名
            username = "未知用户"
            try:
                user = bot.get_chat(user_id_db)
                username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
                if len(username) > 15:
                    username = username[:12] + "..."
            except:
                pass
            
            # 检查是否为VIP
            cursor.execute('SELECT is_vip, vip_expire FROM vip_users WHERE user_id = ?', (user_id_db,))
            vip_info = cursor.fetchone()
            is_vip = "⭐" if vip_info and vip_info[0] else ""
            
            text += f"**{i}. {is_vip}{username}**\n"
            text += f"├ ID: `{user_id_db}`\n"
            text += f"├ 解码次数: {total_decodes}次\n"
            text += f"└ 接收文件: {total_receives}个\n\n"
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("⬅️ 返回用户管理", callback_data="admin_users"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    bot.answer_callback_query(call.id)

@bot.callback_query_handler(func=lambda call: call.data == "admin_file_ranking")
def handle_admin_file_ranking(call):
    """显示文件排行"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 获取文件上传排行
    cursor.execute('''
    SELECT user_id, 
           COUNT(*) as pack_count,
           SUM(file_count) as total_files,
           MAX(created_at) as last_upload
    FROM packs
    GROUP BY user_id
    ORDER BY total_files DESC
    LIMIT 20
    ''')
    
    file_ranking = cursor.fetchall()
    
    if not file_ranking:
        text = "📭 暂无文件上传数据"
    else:
        text = "📊 **文件上传排行**\n\n"
        
        for i, user_stats in enumerate(file_ranking, 1):
            user_id_db = user_stats[0]
            pack_count = user_stats[1] or 0
            total_files = user_stats[2] or 0
            last_upload = user_stats[3]
            
            # 获取用户名
            username = "未知用户"
            try:
                user = bot.get_chat(user_id_db)
                username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
                if len(username) > 15:
                    username = username[:12] + "..."
            except:
                pass
            
            # 检查是否为VIP
            cursor.execute('SELECT is_vip, vip_expire FROM vip_users WHERE user_id = ?', (user_id_db,))
            vip_info = cursor.fetchone()
            is_vip = "⭐" if vip_info and vip_info[0] else ""
            
            text += f"**{i}. {is_vip}{username}**\n"
            text += f"├ ID: `{user_id_db}`\n"
            text += f"├ 文件包: {pack_count}个\n"
            text += f"└ 总文件: {total_files}个\n\n"
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("⬅️ 返回用户管理", callback_data="admin_users"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    bot.answer_callback_query(call.id)
    
# ==================== 回调查询处理器 ====================
@bot.callback_query_handler(func=lambda call: True)
def handle_callback(call):
    """处理回调查询"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    # 【新增】套餐管理相关的回调处理
    if call.data.startswith("manage_package_"):
        package_id = int(call.data.replace("manage_package_", ""))
        vip_payment_system.show_package_detail_management(
            user_id, chat_id, package_id, message_id
        )
        bot.answer_callback_query(call.id)
        return
    
    elif call.data == "manage_packages":
        vip_payment_system.show_package_management(
            user_id, chat_id, message_id
        )
        bot.answer_callback_query(call.id)
        return
    
    elif call.data == "add_new_package":
        vip_payment_system.add_new_package(user_id, chat_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data == "package_stats":
        show_package_statistics(user_id, chat_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("edit_price_"):
        package_id = int(call.data.replace("edit_price_", ""))
        ask_for_new_price(user_id, chat_id, package_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("edit_name_"):
        package_id = int(call.data.replace("edit_name_", ""))
        ask_for_new_name(user_id, chat_id, package_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("edit_months_"):
        package_id = int(call.data.replace("edit_months_", ""))
        ask_for_new_months(user_id, chat_id, package_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("edit_desc_"):
        package_id = int(call.data.replace("edit_desc_", ""))
        ask_for_new_description(user_id, chat_id, package_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("edit_order_"):
        package_id = int(call.data.replace("edit_order_", ""))
        ask_for_new_order(user_id, chat_id, package_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("toggle_status_"):
        package_id = int(call.data.replace("toggle_status_", ""))
        toggle_package_status(user_id, chat_id, package_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("delete_package_"):
        package_id = int(call.data.replace("delete_package_", ""))
        confirm_delete_package(user_id, chat_id, package_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("confirm_delete_pkg_"):
        handle_confirm_delete_package(call)
        return
    # 【直接在这里处理广播回调】
    if call.data == "confirm_broadcast":
        handle_confirm_broadcast(call)
        return
    elif call.data == "cancel_broadcast":
        handle_cancel_broadcast(call)
        return
    elif call.data == "admin_broadcast":
        handle_admin_broadcast_callback(call)
        return
    if call.data.startswith("send_") or call.data in ["countdown_info", "progress_info", "back_to_main_from_send"]:
        handle_send_callback(call)
        return
    
    elif call.data.startswith("vip_"):
        handle_vip_callback(call)
        return
    
    elif call.data.startswith("page_"):
        handle_page_callback(call)
        return
    elif call.data == "manage_payment_methods":
        show_payment_methods_management(user_id, chat_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("manage_payment_method_"):
        method_id = call.data.replace("manage_payment_method_", "")
        show_payment_method_detail_management(user_id, chat_id, method_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("toggle_payment_status_"):
        method_id = call.data.replace("toggle_payment_status_", "")
        success, msg = payment_method_manager.toggle_method_status(method_id)
        bot.answer_callback_query(call.id, msg, show_alert=True)
        show_payment_method_detail_management(user_id, chat_id, method_id, message_id)
        return
    
    elif call.data.startswith("edit_payment_name_"):
        method_id = call.data.replace("edit_payment_name_", "")
        ask_for_new_payment_name(user_id, chat_id, method_id, message_id)
        bot.answer_callback_query(call.id)
        return
    
    elif call.data.startswith("edit_payment_order_"):
        method_id = call.data.replace("edit_payment_order_", "")
        ask_for_new_payment_order(user_id, chat_id, method_id, message_id)
        bot.answer_callback_query(call.id)
        return    

    elif call.data.startswith("admin_"):
        handle_admin_callback(call)
        return
    
    elif call.data.startswith("code_detail_"):
        handle_code_detail(call)
        return
    
    elif call.data.startswith("add_remark_"):
        handle_add_remark(call)
        return
    
    elif call.data.startswith("edit_remark_"):
        handle_edit_remark(call)
        return
    
    elif call.data.startswith("delete_remark_"):
        handle_delete_remark(call)
        return
    
    elif call.data == "search_remarks":
        handle_search_remarks(call)
        return
    
    elif call.data == "my_codes":
        handle_my_codes_callback(call)
        return
    
    elif call.data == "back_to_stats":
        if is_admin(user_id):
            handle_stats_command_by_message_id(chat_id, message_id)
        else:
            bot.answer_callback_query(call.id, "❌ 权限不足")
        return
    elif call.data == "admin_broadcast":
        handle_admin_broadcast_callback(call)
        return
    elif call.data == "back_to_main":
        try:
            bot.delete_message(chat_id, message_id)
        except:
            pass
def handle_admin_broadcast_callback(call):
    """处理管理员广播回调"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    # 询问广播内容
    text = """
📢 **发送广播消息**

请输入要广播的内容：
• 支持Markdown格式
• 最大长度：4000字
• 所有用户都会收到这条消息

⚠️ **注意事项：**
1. 请勿发送违规内容
2. 广播无法撤回
3. 用户可能会被打扰

💡 **广播示例：**
【系统通知】
亲爱的用户：
系统将于今晚23:00-24:00进行维护。
在此期间服务将暂停，请提前安排。

感谢您的理解与支持！
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("❌ 取消", callback_data="back_to_stats")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    # 注册下一步处理器
    msg = bot.send_message(chat_id, "请直接发送广播内容（输入 /cancel 取消）：")
    bot.register_next_step_handler(msg, handle_broadcast_content_input)
    
    bot.answer_callback_query(call.id)

def handle_broadcast_content_input(message):
    """处理广播内容输入"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    
    if not is_admin(user_id):
        bot.send_message(chat_id, "❌ 权限不足", reply_markup=create_main_menu())
        return
    
    # 检查是否取消
    if message.text and message.text.strip() == "/cancel":
        bot.send_message(chat_id, "❌ 广播已取消", reply_markup=create_main_menu())
        return
    
    broadcast_content = message.text
    if not broadcast_content:
        bot.send_message(chat_id, "❌ 广播内容不能为空", reply_markup=create_main_menu())
        return
    
    if len(broadcast_content) > 4000:
        bot.send_message(chat_id, "❌ 广播内容太长（最多4000字）", reply_markup=create_main_menu())
        return
    
    # 确认广播
    confirm_text = f"""
📢 **确认广播内容**

**广播内容预览：**
{broadcast_content[:500]}{'...' if len(broadcast_content) > 500 else ''}

📊 **发送信息：**
• 内容长度：{len(broadcast_content)} 字
• 发送时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
• 发送者：管理员 (ID: `{user_id}`)

⚠️ **确认要发送给所有用户吗？**
此操作无法撤销！
"""
    
    # 保存广播内容到临时存储
    user_sessions = getattr(bot, 'user_sessions', {})
    user_sessions[user_id] = {
        'action': 'broadcast',
        'content': broadcast_content,
        'time': datetime.now().isoformat()
    }
    bot.user_sessions = user_sessions
    
    markup = telebot.types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        telebot.types.InlineKeyboardButton("✅ 确认发送", callback_data="confirm_broadcast"),
        telebot.types.InlineKeyboardButton("❌ 取消发送", callback_data="cancel_broadcast")
    )
    
    bot.send_message(
        chat_id,
        confirm_text,
        parse_mode='Markdown',
        reply_markup=markup
    )

@bot.callback_query_handler(func=lambda call: call.data == "confirm_broadcast")
def handle_confirm_broadcast(call):
    """确认发送广播"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    # 获取广播内容
    user_sessions = getattr(bot, 'user_sessions', {})
    session = user_sessions.get(user_id, {})
    
    if not session or session.get('action') != 'broadcast':
        bot.answer_callback_query(call.id, "❌ 广播内容已过期", show_alert=True)
        return
    
    broadcast_content = session.get('content', '')
    broadcast_time = session.get('time', datetime.now().isoformat())
    
    # 开始发送广播
    bot.answer_callback_query(call.id, "📢 开始发送广播...", show_alert=True)
    
    try:
        # 更新消息状态
        bot.edit_message_text(
            "📢 **正在发送广播...**\n\n⏳ 请稍候，这可能需要一些时间...",
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown'
        )
    except:
        pass
    
    # 发送广播给所有用户
    success_count, fail_count = send_broadcast_to_all_users(broadcast_content, user_id)
    
    # 发送完成报告
    report_text = f"""
✅ **广播发送完成！**

📊 **发送统计：**
• 成功发送：{success_count} 人
• 发送失败：{fail_count} 人
• 总接收用户：{success_count + fail_count} 人
• 发送时间：{datetime.fromisoformat(broadcast_time).strftime('%Y-%m-%d %H:%M:%S')}

💡 **广播内容已发送给所有用户。**
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("⬅️ 返回管理面板", callback_data="back_to_stats")
    )
    
    bot.send_message(
        chat_id,
        report_text,
        parse_mode='Markdown',
        reply_markup=markup
    )
    
    # 清理session
    if user_id in user_sessions:
        del user_sessions[user_id]
        bot.user_sessions = user_sessions

@bot.callback_query_handler(func=lambda call: call.data == "cancel_broadcast")
def handle_cancel_broadcast(call):
    """取消广播"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    # 清理session
    user_sessions = getattr(bot, 'user_sessions', {})
    if user_id in user_sessions:
        del user_sessions[user_id]
        bot.user_sessions = user_sessions
    
    bot.answer_callback_query(call.id, "❌ 广播已取消", show_alert=True)
    
    try:
        bot.delete_message(chat_id, call.message.message_id)
    except:
        pass
    
    bot.send_message(
        chat_id,
        "📢 广播发送已取消",
        reply_markup=create_main_menu()
    )

def send_broadcast_to_all_users(content, admin_id):
    """发送广播给所有用户"""
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 获取所有用户（上传过文件的用户）
    cursor.execute('SELECT DISTINCT user_id FROM packs')
    all_users = [row[0] for row in cursor.fetchall()]
    
    # 获取VIP用户
    cursor.execute('SELECT DISTINCT user_id FROM vip_users')
    vip_users = [row[0] for row in cursor.fetchall()]
    
    # 合并所有用户（去重）
    all_user_ids = list(set(all_users + vip_users))
    
    success_count = 0
    fail_count = 0
    
    total_users = len(all_user_ids)
    
    # 发送进度消息
    progress_msg = bot.send_message(
        admin_id,
        f"📢 **广播发送进度**\n\n总用户数：{total_users} 人\n已发送：0/{total_users}\n成功率：0%"
    )
    
    for i, user_id in enumerate(all_user_ids, 1):
        try:
            # 构建广播消息
            broadcast_message = f"""
📢 **【机器人公告】**

{content}

---
⏰ 发送时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
💡 这是系统广播消息，请勿回复。
"""
            
            # 发送消息
            bot.send_message(
                user_id,
                broadcast_message,
                parse_mode='Markdown'
            )
            
            success_count += 1
            
            # 每发送20个用户更新一次进度
            if i % 20 == 0 or i == total_users:
                success_rate = (success_count / i) * 100
                try:
                    bot.edit_message_text(
                        f"📢 **广播发送进度**\n\n"
                        f"总用户数：{total_users} 人\n"
                        f"已发送：{i}/{total_users}\n"
                        f"成功：{success_count}\n"
                        f"失败：{i - success_count}\n"
                        f"成功率：{success_rate:.1f}%",
                        chat_id=admin_id,
                        message_id=progress_msg.message_id,
                        parse_mode='Markdown'
                    )
                except:
                    pass
            
            # 避免发送过快
            time.sleep(0.1)
            
        except Exception as e:
            fail_count += 1
            print(f"发送广播给用户 {user_id} 失败: {e}")
            continue
    
    # 删除进度消息
    try:
        bot.delete_message(admin_id, progress_msg.message_id)
    except:
        pass
    
    return success_count, fail_count        

# ==================== 【新增】备注相关回调处理 ====================
def handle_code_detail(call):
    """处理文件码详情"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    code = call.data.replace('code_detail_', '')
    
    # 获取文件包信息
    pack = get_pack_by_code(code)
    if not pack:
        bot.answer_callback_query(call.id, "❌ 文件码不存在")
        return
    
    # 获取备注信息
    remark_info = pack_remark_manager.get_remark(user_id, code)
    
    # 【新增】获取解码次数
    decode_count = pack_decode_manager.get_decode_count_with_cache(code)
    # 选择图标
    if decode_count == 0:
        decode_icon = "📭"
    elif decode_count <= 10:
        decode_icon = "📊"
    else:
        decode_icon = "🔥"
    
    text = f"""
📋 **文件码详情**

🔢 **代码：** `{code}`
📊 **文件数：** {pack['file_count']} 个
📁 **文件类型：** {pack['file_types']}
🕐 **创建时间：** {pack['created_at'][:16].replace('T', ' ') if isinstance(pack['created_at'], str) else pack['created_at']}
{decode_icon} **解码次数：** {decode_count} 次

"""
    
    if remark_info:
        text += f"""
📝 **备注信息：**
• 备注：{remark_info['remark']}
{f"• 标签：{remark_info['tags']}" if remark_info['tags'] else ""}
• 更新：{remark_info['updated_at'][:16].replace('T', ' ') if isinstance(remark_info['updated_at'], str) else remark_info['updated_at']}
"""
    else:
        text += "\n📝 **备注：** 未添加备注"
    
    markup = telebot.types.InlineKeyboardMarkup(row_width=2)
    
    if remark_info:
        markup.row(
            telebot.types.InlineKeyboardButton("✏️ 修改备注", callback_data=f"edit_remark_{code}"),
            telebot.types.InlineKeyboardButton("🗑️ 删除备注", callback_data=f"delete_remark_{code}")
        )
    else:
        markup.row(
            telebot.types.InlineKeyboardButton("📝 添加备注", callback_data=f"add_remark_{code}")
        )
    
    markup.row(
        telebot.types.InlineKeyboardButton("📤 发送文件", callback_data=f"send_code_{code}"),
        telebot.types.InlineKeyboardButton("📋 复制代码", callback_data=f"copy_code_{code}")
    )
    markup.row(
        telebot.types.InlineKeyboardButton("⬅️ 返回列表", callback_data="my_codes"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=call.message.message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    bot.answer_callback_query(call.id)

def handle_add_remark(call):
    """处理添加备注"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    code = call.data.replace('add_remark_', '')
    
    # 保存当前code到session
    user_sessions = getattr(bot, 'user_sessions', {})
    user_sessions[user_id] = {'action': 'add_remark', 'code': code}
    bot.user_sessions = user_sessions
    
    text = f"""
📝 **为文件码添加备注**

文件码：`{code}`

请输入备注内容（最多200字）：
• 可以添加描述
• 可以使用标签，用逗号分隔
• 例如："2024工作报告,重要"

💡 标签示例：工作,重要,文档,报告
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("⬅️ 取消", callback_data=f"code_detail_{code}")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=call.message.message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    # 注册下一步处理器
    msg = bot.send_message(chat_id, "请直接发送备注内容：")
    bot.register_next_step_handler(msg, handle_remark_input, code, 'add')
    
    bot.answer_callback_query(call.id)

def handle_edit_remark(call):
    """处理修改备注"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    code = call.data.replace('edit_remark_', '')
    
    # 获取当前备注
    remark_info = pack_remark_manager.get_remark(user_id, code)
    
    if not remark_info:
        bot.answer_callback_query(call.id, "❌ 未找到备注", show_alert=True)
        return
    
    # 保存当前code到session
    user_sessions = getattr(bot, 'user_sessions', {})
    user_sessions[user_id] = {'action': 'edit_remark', 'code': code, 'old_remark': remark_info}
    bot.user_sessions = user_sessions
    
    text = f"""
✏️ **修改文件码备注**

文件码：`{code}`
当前备注：{remark_info['remark']}
{f"当前标签：{remark_info['tags']}" if remark_info['tags'] else ""}

请输入新的备注内容（最多200字）：
• 可以修改描述
• 可以修改标签
• 例如："2024工作报告更新版,重要,已完成"
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("⬅️ 取消", callback_data=f"code_detail_{code}")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=call.message.message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    # 注册下一步处理器
    msg = bot.send_message(chat_id, "请直接发送新的备注内容：")
    bot.register_next_step_handler(msg, handle_remark_input, code, 'edit')
    
    bot.answer_callback_query(call.id)

def handle_delete_remark(call):
    """处理删除备注"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    code = call.data.replace('delete_remark_', '')
    
    # 确认删除
    text = f"""
🗑️ **确认删除备注**

文件码：`{code}`

⚠️ **您确定要删除这个备注吗？**
删除后将无法恢复！

请确认操作：
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("✅ 确认删除", callback_data=f"confirm_delete_remark_{code}"),
        telebot.types.InlineKeyboardButton("❌ 取消", callback_data=f"code_detail_{code}")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=call.message.message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    bot.answer_callback_query(call.id)

@bot.callback_query_handler(func=lambda call: call.data.startswith('confirm_delete_remark_'))
def handle_confirm_delete_remark(call):
    """确认删除备注"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    code = call.data.replace('confirm_delete_remark_', '')
    
    # 删除备注
    success = pack_remark_manager.delete_remark(user_id, code)
    
    if success:
        bot.answer_callback_query(call.id, "✅ 备注已删除", show_alert=True)
        
        # 返回文件码详情
        handle_code_detail(call)
    else:
        bot.answer_callback_query(call.id, "❌ 删除失败", show_alert=True)

def handle_remark_input(message, code, action):
    """处理备注输入"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    
    remark_text = message.text.strip()
    
    if not remark_text:
        bot.send_message(
            chat_id,
            "❌ 备注内容不能为空",
            reply_markup=create_main_menu()
        )
        return
    
    # 添加备注
    success, result_msg = pack_remark_manager.add_remark(user_id, code, remark_text)
    
    if success:
        if action == 'add':
            action_text = "添加"
        else:
            action_text = "修改"
        
        text = f"""
✅ **备注{action_text}成功！**

文件码：`{code}`
备注：{remark_text}

💡 备注已保存，可以使用搜索功能快速查找
"""
    else:
        text = result_msg
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("📋 查看详情", callback_data=f"code_detail_{code}"),
        telebot.types.InlineKeyboardButton("🔍 搜索备注", callback_data="search_remarks")
    )
    markup.row(
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    bot.send_message(
        chat_id,
        text,
        parse_mode='Markdown',
        reply_markup=markup
    )

def handle_search_remarks(call):
    """处理搜索备注"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    text = """
🔍 **文件码备注搜索**

请输入搜索关键词：
• 可以搜索备注内容
• 可以搜索标签
• 支持模糊搜索

📌 **使用方法：**
直接发送搜索关键词

例如：
工作报告
重要
图片

💡 **提示：**
• 搜索自己的备注内容
• 快速找到已备注的文件码
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("⬅️ 返回", callback_data="my_codes"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=call.message.message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    # 注册下一步处理器
    msg = bot.send_message(chat_id, "请直接发送搜索关键词：")
    bot.register_next_step_handler(msg, handle_remark_search_input)
    
    bot.answer_callback_query(call.id)

def handle_remark_search_input(message):
    """处理备注搜索输入"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    keyword = message.text.strip()
    
    if not keyword or len(keyword) < 1:
        bot.send_message(
            chat_id,
            "❌ 请输入有效的搜索关键词（至少1个字符）",
            reply_markup=create_main_menu()
        )
        return
    
    if len(keyword) > 50:
        bot.send_message(
            chat_id,
            "❌ 搜索关键词太长（最多50字）",
            reply_markup=create_main_menu()
        )
        return
    
    # 执行搜索
    if is_admin(user_id):
        # 管理员搜索所有用户备注
        results = pack_remark_manager.search_all_remarks_admin(keyword)
    else:
        # 普通用户搜索自己的备注
        results = pack_remark_manager.search_user_remarks(user_id, keyword)
    
    if not results:
        text = f"""
🔍 **搜索结果：** "{keyword}"

❌ 未找到匹配的文件码

💡 提示：
1. 检查关键词是否正确
2. 尝试更短的关键词
3. 确保已为文件码添加备注
"""
    else:
        text = f"""
🔍 **搜索结果：** "{keyword}"

✅ 找到 {len(results)} 个匹配的文件码：

"""
        
        for i, result in enumerate(results[:10], 1):  # 最多显示10个
            # 截断长备注
            remark_display = result['remark']
            if len(remark_display) > 40:
                remark_display = remark_display[:37] + "..."
            
            text += f"\n**{i}. ** `{result['code']}`\n"
            text += f"├─ 📝 {remark_display}\n"
            if result.get('tags'):
                text += f"├─ 🏷️ 标签：{result['tags']}\n"
            text += f"├─ 文件：{result['file_count']}个\n"
            text += f"└─ 类型：{result['file_types']}\n"
        
        if len(results) > 10:
            text += f"\n... 还有 {len(results)-10} 个结果未显示"
        
        text += "\n💡 发送上面的文件码即可取回文件"
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("🔄 重新搜索", callback_data="search_remarks"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    bot.send_message(
        chat_id,
        text,
        parse_mode='Markdown',
        reply_markup=markup
    )

def handle_my_codes_callback(call):
    """处理我的代码回调"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    try:
        bot.delete_message(chat_id, call.message.message_id)
    except:
        pass
    
    show_user_codes_page(user_id, chat_id, 1)
    bot.answer_callback_query(call.id)

# ==================== 其他原有的回调处理函数 ====================
def handle_stats_command_by_message_id(chat_id, message_id):
    """通过消息ID显示统计信息"""
    class MockMessage:
        def __init__(self, chat_id):
            self.chat = type('Chat', (), {'id': chat_id})()
            self.from_user = type('User', (), {'id': ADMIN_USER_IDS[0] if ADMIN_USER_IDS else 0})()
    
    mock_message = MockMessage(chat_id)
    handle_stats_command(mock_message)
    
    try:
        bot.delete_message(chat_id, message_id)
    except:
        pass

def handle_send_callback(call):
    """处理文件发送的回调"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if call.data == "send_next":
        can_click, wait_time = file_send_paginator.can_click_next(user_id, chat_id)
        
        if not can_click:
            bot.answer_callback_query(
                call.id,
                f"⏳ 请等待 {wait_time} 秒（按钮会自动更新）",
                show_alert=True
            )
            
            file_send_paginator.register_waiting_message(user_id, chat_id, message_id, wait_time)
            return
        
        next_batch = file_send_paginator.get_next_batch(user_id, chat_id)
        
        if next_batch:
            try:
                bot.delete_message(chat_id, message_id)
            except:
                pass
            
            file_send_paginator.unregister_waiting_message(user_id, chat_id, message_id)
            
            if send_files_compact(chat_id, next_batch['files']):
                for file_info in next_batch['files']:
                    user_limit_manager.increment_file_receive_count(user_id, 1, file_info['file_type'])
                
                if next_batch['is_last']:
                    bot.send_message(
                        chat_id,
                        f"🎉 **所有文件发送完成！**\n\n代码：`{next_batch['code']}`",
                        parse_mode='Markdown',
                        reply_markup=create_main_menu()
                    )
                    file_send_paginator.clear_session(user_id, chat_id)
                else:
                    send_page_info(user_id, chat_id, next_batch)
            else:
                bot.send_message(
                    chat_id, 
                    "❌ 发送失败，请重试",
                    reply_markup=create_main_menu()
                )
                file_send_paginator.clear_session(user_id, chat_id)
        else:
            bot.answer_callback_query(call.id, "✅ 所有文件已发送完成")
            bot.send_message(
                chat_id,
                "🎉 所有文件已发送完成！",
                reply_markup=create_main_menu()
            )
            file_send_paginator.clear_session(user_id, chat_id)
    
    elif call.data == "send_complete":
        bot.answer_callback_query(call.id, "✅ 文件发送已完成")
        try:
            bot.delete_message(chat_id, message_id)
        except:
            pass
        bot.send_message(
            chat_id, 
            "🏠 返回主菜单",
            reply_markup=create_main_menu()
        )
        file_send_paginator.clear_session(user_id, chat_id)
    
    elif call.data == "back_to_main_from_send":
        file_send_paginator.clear_session(user_id, chat_id)
        try:
            bot.delete_message(chat_id, message_id)
        except:
            pass
        bot.send_message(
            chat_id,
            "🏠 返回主菜单",
            reply_markup=create_main_menu()
        )
        bot.answer_callback_query(call.id, "已返回主菜单")
    
    elif call.data == "countdown_info":
        can_click, wait_time = file_send_paginator.can_click_next(user_id, chat_id)
        if can_click:
            bot.answer_callback_query(call.id, "✅ 可以点击下一页了！", show_alert=True)
        else:
            bot.answer_callback_query(call.id, f"⏳ 还需等待 {wait_time} 秒", show_alert=True)
    
    elif call.data == "progress_info":
        session = file_send_paginator.get_current_session(user_id, chat_id)
        if session:
            current = session['current_batch']
            total = session['total_batches']
            bot.answer_callback_query(
                call.id,
                f"当前进度：第 {current} 批 / 共 {total} 批\n总计文件：{session['total_files']} 个",
                show_alert=True
            )
        else:
            bot.answer_callback_query(call.id, "当前发送进度")
    
    elif call.data.startswith("send_code_"):
        code = call.data.replace("send_code_", "")
        process_file_code_silent(user_id, chat_id, code)
        bot.answer_callback_query(call.id, "🚀 开始发送文件...")
    
    elif call.data.startswith("copy_code_"):
        code = call.data.replace("copy_code_", "")
        bot.answer_callback_query(call.id, f"📋 已复制代码：{code}", show_alert=True)

def handle_vip_callback(call):
    """处理VIP回调"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    print(f"🔄 VIP回调: {call.data} from {user_id}")
    
    try:
        if call.data == "vip_center":
            show_vip_center(user_id, chat_id)
            bot.answer_callback_query(call.id)
            
        elif call.data == "vip_packages":
            print(f"📦 显示VIP套餐 for {user_id}")
            vip_payment_system.show_vip_packages(user_id, chat_id)
            bot.answer_callback_query(call.id)
            
        elif call.data.startswith("vip_package_"):
            try:
                package_id = int(call.data.replace("vip_package_", ""))
                print(f"💰 选择套餐 {package_id}")
                vip_payment_system.show_payment_methods(user_id, chat_id, package_id, call)
            except ValueError as e:
                print(f"❌ 套餐ID解析失败: {e}")
                bot.answer_callback_query(
                    call.id, 
                    "❌ 套餐选择失败，请重试", 
                    show_alert=True
                )
                
        elif call.data.startswith("vip_pay_"):
            parts = call.data.replace("vip_pay_", "").split("_")
            if len(parts) == 2:
                package_id = int(parts[0])
                method_id = parts[1]
                vip_payment_system.create_payment_order(user_id, chat_id, package_id, method_id)
                bot.answer_callback_query(call.id)
            
        elif call.data.startswith("vip_confirm_"):
            order_id = call.data.replace("vip_confirm_", "")
            vip_payment_system.confirm_payment(call, order_id)
            
        elif call.data == "vip_compare":
            text = """
📋 **套餐对比**

🔓 **普通用户**
• 每日解码：50次
• 每日接收文件：50个
• 下一页间隔：5秒
• 最大文件：20MB/个
• 最大包大小：50个文件
• 广告：有

⭐ **VIP用户**
• 每日解码：无限
• 每日接收文件：无限
• 下一页间隔：无限制
• 最大文件：500MB/个
• 最大包大小：500个文件
• 广告：无

💰 **价格**
• 20元/月
• 60元/季（省10%）
• 240元/年（省20%）
"""
            
            markup = telebot.types.InlineKeyboardMarkup(row_width=2)
            markup.add(
                telebot.types.InlineKeyboardButton("💰 VIP套餐", callback_data="vip_packages"),
                telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
            )
            
            try:
                bot.edit_message_text(
                    text,
                    chat_id=chat_id,
                    message_id=message_id,
                    parse_mode='Markdown',
                    reply_markup=markup
                )
            except:
                bot.send_message(
                    chat_id,
                    text,
                    parse_mode='Markdown',
                    reply_markup=markup
                )
            
            bot.answer_callback_query(call.id)
            
        elif call.data == "vip_faq":
            text = f"""
❓ **常见问题**

Q: VIP如何购买？
A: 点击"购买VIP套餐"，选择套餐和支付方式，跳转到支付页面扫码支付。

Q: 支付后多久激活？
A: 支付后点击"我已支付"，管理员会在5分钟内激活。

Q: 可以退款吗？
A: 虚拟商品，一经激活不支持退款。

Q: VIP有哪些特权？
A: 无限解码、无限接收文件、无点击限制等。

Q: 如何联系客服？
A: 📞 联系客服：@kfjdfkjdd_bot

🤖 **客服机器人：** @kfjdfkjdd_bot
⏰ **服务时间：** 全天24小时
💡 **联系时请提供用户ID和问题描述**
"""
            
            markup = telebot.types.InlineKeyboardMarkup(row_width=2)
            markup.add(
                telebot.types.InlineKeyboardButton("📞 联系客服", url="https://t.me/kfjdfkjdd_bot"),
                telebot.types.InlineKeyboardButton("💰 VIP套餐", callback_data="vip_packages")
            )
            markup.add(
                telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
            )
            
            try:
                bot.edit_message_text(
                    text,
                    chat_id=chat_id,
                    message_id=message_id,
                    parse_mode='Markdown',
                    reply_markup=markup
                )
            except:
                bot.send_message(
                    chat_id,
                    text,
                    parse_mode='Markdown',
                    reply_markup=markup
                )
            
            bot.answer_callback_query(call.id)
            
        else:
            # 处理未知回调
            bot.answer_callback_query(
                call.id,
                "❌ 未知操作",
                show_alert=True
            )
            
    except Exception as e:
        print(f"❌ VIP回调处理错误: {e}")
        traceback.print_exc()
        
        bot.answer_callback_query(
            call.id,
            f"❌ 操作失败: {str(e)[:50]}",
            show_alert=True
        )

def handle_page_callback(call):
    """处理翻页回调"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if call.data == "page_info":
        bot.answer_callback_query(call.id, "当前页面信息")
        return
    
    try:
        page_num = int(call.data.split("_")[1])
        show_user_codes_page(user_id, chat_id, page_num)
        
        try:
            bot.delete_message(chat_id, message_id)
        except:
            pass
        
        bot.answer_callback_query(call.id, f"跳转到第 {page_num} 页")
        
    except Exception as e:
        print(f"❌ 翻页错误: {e}")
        bot.answer_callback_query(call.id, "❌ 翻页失败")

def handle_admin_callback(call):
    """处理管理员回调"""
    user_id = call.from_user.id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    # 处理删除订单和移除VIP（已经添加的）
    if call.data.startswith("admin_delete_order_"):
        handle_admin_delete_order_callback(call)
        return
    
    elif call.data.startswith("admin_confirm_delete_order_"):
        handle_admin_confirm_delete_order_callback(call)
        return
    
    elif call.data.startswith("admin_remove_vip_"):
        handle_admin_remove_vip_callback(call)
        return
    
    elif call.data.startswith("admin_confirm_remove_vip_"):
        handle_admin_confirm_remove_vip_callback(call)
        return
    
    # 处理系统监控
    elif call.data == "admin_monitor":
        handle_admin_monitor_callback(call)
        return
    
    # 处理广播功能
    elif call.data == "admin_broadcast":
        handle_admin_broadcast_callback(call)
        return
    
    elif call.data == "confirm_broadcast":
        handle_confirm_broadcast(call)
        return
    
    elif call.data == "cancel_broadcast":
        handle_cancel_broadcast(call)
        return
    
    # 原有代码继续...
    if call.data.startswith("admin_activate_"):
        handle_admin_activate_callback(call)
    elif call.data == "admin_pending_orders":
        handle_admin_pending_orders_callback(call)
    elif call.data == "admin_search_remarks":
        handle_admin_search_remarks_callback(call)
    elif call.data == "admin_users":
        handle_admin_users_callback(call)
    elif call.data == "admin_vip_list":
        handle_admin_vip_list(call)
    elif call.data == "admin_normal_list":
        handle_admin_normal_list(call)
    elif call.data == "admin_active_users":
        handle_admin_active_users(call)
    elif call.data == "admin_file_ranking":
        handle_admin_file_ranking(call)
    else:
        bot.answer_callback_query(call.id, "管理员功能暂未实现")

# ==================== 【修改点2：待处理订单添加删除按钮】====================
def handle_admin_pending_orders_callback(call):
    """显示待处理订单"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    cursor.execute('''
    SELECT po.order_id, po.user_id, po.amount, po.payment_method, 
           vp.name, vp.months, po.created_at, po.status
    FROM payment_orders po
    JOIN vip_packages vp ON po.package_id = vp.id
    WHERE po.status IN ('pending', 'paid') AND po.activated = 0
    ORDER BY po.created_at DESC
    LIMIT 20
    ''')
    
    orders = cursor.fetchall()
    
    if not orders:
        text = "📭 没有待处理的订单"
        markup = telebot.types.InlineKeyboardMarkup()
        markup.add(
            telebot.types.InlineKeyboardButton("⬅️ 返回", callback_data="back_to_stats")
        )
        
        try:
            bot.edit_message_text(
                text,
                chat_id=chat_id,
                message_id=message_id,
                parse_mode='Markdown',
                reply_markup=markup
            )
        except:
            bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
        
        bot.answer_callback_query(call.id)
        return
    
    text = "💰 **待处理订单列表**\n\n"
    markup = telebot.types.InlineKeyboardMarkup(row_width=2)
    
    for i, order in enumerate(orders, 1):
        order_id, order_user_id, amount, method_id, name, months, created_at, status = order
        
        # 获取用户名
        username = "未知用户"
        try:
            user = bot.get_chat(order_user_id)
            username = user.username or f"{user.first_name or ''} {user.last_name or ''}".strip()
        except:
            pass
        
        method_name = VIPPackageConfig.PAYMENT_METHODS.get(method_id, {}).get("name", method_id)
        
        # 状态文本
        status_text = {
            'pending': '⏳ 待支付',
            'paid': '✅ 已支付',
            'activated': '⭐ 已激活',
            'cancelled': '❌ 已取消'
        }.get(status, status)
        
        text += f"**{i}. {status_text} - {name}**\n"
        text += f"├─ 用户：{username} (ID: `{order_user_id}`)\n"
        text += f"├─ 套餐：{name} ({months}个月)\n"
        text += f"├─ 金额：{amount}元\n"
        text += f"├─ 支付：{method_name}\n"
        
        # 格式化时间
        try:
            if isinstance(created_at, str):
                time_str = created_at[:16].replace('T', ' ')
            else:
                time_str = created_at.strftime('%Y-%m-%d %H:%M')
            text += f"└─ 时间：{time_str}\n\n"
        except:
            text += "└─ 时间：未知\n\n"
        
        # 按钮行：激活按钮 + 删除按钮
        button_row = []
        
        # 只有已支付的订单才显示激活按钮
        if status == 'paid':
            button_row.append(
                telebot.types.InlineKeyboardButton(
                    f"✅ 激活{i}", 
                    callback_data=f"admin_activate_{order_user_id}_{order_id}_{months}"
                )
            )
        
        # 所有订单都显示删除按钮
        button_row.append(
            telebot.types.InlineKeyboardButton(
                f"🗑️ 删除{i}", 
                callback_data=f"admin_delete_order_{order_id}"
            )
        )
        
        markup.row(*button_row)
    
    markup.add(
        telebot.types.InlineKeyboardButton("🔄 刷新列表", callback_data="admin_pending_orders"),
        telebot.types.InlineKeyboardButton("⬅️ 返回", callback_data="back_to_stats")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except Exception as e:
        print(f"更新消息失败: {e}")
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    bot.answer_callback_query(call.id)

# ==================== 【新增：删除订单回调函数】====================
@bot.callback_query_handler(func=lambda call: call.data.startswith('admin_delete_order_'))
def handle_admin_delete_order_callback(call):
    """处理删除订单"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    order_id = call.data.replace('admin_delete_order_', '')
    
    # 确认删除
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("✅ 确认删除", callback_data=f"admin_confirm_delete_order_{order_id}"),
        telebot.types.InlineKeyboardButton("❌ 取消", callback_data="admin_pending_orders")
    )
    
    try:
        bot.edit_message_text(
            f"🗑️ **确认删除订单**\n\n订单号：`{order_id}`\n\n⚠️ **您确定要删除这个订单吗？**\n删除后将无法恢复！",
            chat_id=chat_id,
            message_id=call.message.message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            f"🗑️ **确认删除订单**\n\n订单号：`{order_id}`\n\n⚠️ **您确定要删除这个订单吗？**\n删除后将无法恢复！",
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    bot.answer_callback_query(call.id)

@bot.callback_query_handler(func=lambda call: call.data.startswith('admin_confirm_delete_order_'))
def handle_admin_confirm_delete_order_callback(call):
    """确认删除订单"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    order_id = call.data.replace('admin_confirm_delete_order_', '')
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    try:
        # 获取订单信息（用于显示）
        cursor.execute('''
        SELECT user_id, amount, status FROM payment_orders WHERE order_id = ?
        ''', (order_id,))
        
        order_info = cursor.fetchone()
        
        if not order_info:
            bot.answer_callback_query(call.id, "❌ 订单不存在", show_alert=True)
            return
        
        order_user_id, amount, status = order_info
        
        # 删除订单
        cursor.execute('DELETE FROM payment_orders WHERE order_id = ?', (order_id,))
        conn.commit()
        
        bot.answer_callback_query(
            call.id,
            f"✅ 订单已删除\n订单号：{order_id}\n金额：{amount}元\n状态：{status}",
            show_alert=True
        )
        
        # 刷新订单列表
        handle_admin_pending_orders_callback(call)
        
    except Exception as e:
        print(f"删除订单失败: {e}")
        bot.answer_callback_query(call.id, f"❌ 删除失败: {e}", show_alert=True)
        if conn:
            conn.rollback()

# ==================== 【新增：移除VIP回调函数】====================
@bot.callback_query_handler(func=lambda call: call.data.startswith('admin_remove_vip_'))
def handle_admin_remove_vip_callback(call):
    """处理移除VIP"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    target_user_id = int(call.data.replace('admin_remove_vip_', ''))
    
    # 确认移除
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("✅ 确认移除", callback_data=f"admin_confirm_remove_vip_{target_user_id}"),
        telebot.types.InlineKeyboardButton("❌ 取消", callback_data="admin_vip_list")
    )
    
    try:
        bot.edit_message_text(
            f"🗑️ **确认移除VIP**\n\n用户ID：`{target_user_id}`\n\n⚠️ **您确定要移除这个用户的VIP吗？**\n移除后用户将恢复为普通用户！",
            chat_id=chat_id,
            message_id=call.message.message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            f"🗑️ **确认移除VIP**\n\n用户ID：`{target_user_id}`\n\n⚠️ **您确定要移除这个用户的VIP吗？**\n移除后用户将恢复为普通用户！",
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    bot.answer_callback_query(call.id)

@bot.callback_query_handler(func=lambda call: call.data.startswith('admin_confirm_remove_vip_'))
def handle_admin_confirm_remove_vip_callback(call):
    """确认移除VIP"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    target_user_id = int(call.data.replace('admin_confirm_remove_vip_', ''))
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    try:
        # 检查用户是否存在
        cursor.execute('SELECT is_vip FROM vip_users WHERE user_id = ?', (target_user_id,))
        vip_info = cursor.fetchone()
        
        if not vip_info:
            bot.answer_callback_query(call.id, "❌ 用户不是VIP", show_alert=True)
            return
        
        # 移除VIP
        cursor.execute('''
        UPDATE vip_users 
        SET is_vip = 0, vip_expire = NULL, updated_at = ?
        WHERE user_id = ?
        ''', (datetime.now().isoformat(), target_user_id))
        
        conn.commit()
        
        # 清除缓存
        cache_key = f"vip_{target_user_id}"
        if cache_key in user_limit_manager.cache:
            del user_limit_manager.cache[cache_key]
        
        bot.answer_callback_query(
            call.id,
            f"✅ VIP已移除\n用户ID：{target_user_id}",
            show_alert=True
        )
        
        # 刷新VIP列表
        handle_admin_vip_list(call)
        
        # 通知用户
        try:
            bot.send_message(
                target_user_id,
                f"⚠️ **VIP状态变更通知**\n\n您的VIP特权已被管理员移除。\n如有疑问，请联系客服。",
                parse_mode='Markdown'
            )
        except:
            pass
        
    except Exception as e:
        print(f"移除VIP失败: {e}")
        bot.answer_callback_query(call.id, f"❌ 移除失败: {e}", show_alert=True)
        if conn:
            conn.rollback()

def handle_admin_search_remarks_callback(call):
    """管理员搜索备注"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    text = """
🔍 **管理员备注搜索**

请输入搜索关键词：
• 搜索所有用户的备注和标签
• 支持模糊搜索
• 显示文件码、用户信息和备注

📌 **使用方法：**
直接发送搜索关键词

例如：
工作报告
重要
图片

💡 **提示：**
• 搜索范围：所有用户
• 显示用户ID和备注信息
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.row(
        telebot.types.InlineKeyboardButton("⬅️ 返回", callback_data="back_to_stats"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(
            chat_id,
            text,
            parse_mode='Markdown',
            reply_markup=markup
        )
    
    # 注册下一步处理器
    msg = bot.send_message(chat_id, "请直接发送搜索关键词：")
    bot.register_next_step_handler(msg, handle_admin_remark_search_input)
    
    bot.answer_callback_query(call.id)

def handle_admin_remark_search_input(message):
    """处理管理员备注搜索输入"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    keyword = message.text.strip()
    
    if not is_admin(user_id):
        bot.send_message(chat_id, "❌ 权限不足", reply_markup=create_main_menu())
        return
    
    if not keyword or len(keyword) < 1:
        bot.send_message(
            chat_id,
            "❌ 请输入有效的搜索关键词（至少1个字符）",
            reply_markup=create_main_menu()
        )
        return
    
    if len(keyword) > 50:
        bot.send_message(
            chat_id,
            "❌ 搜索关键词太长（最多50字）",
            reply_markup=create_main_menu()
        )
        return
    
    # 执行管理员搜索
    results = pack_remark_manager.search_all_remarks_admin(keyword)
    
    if not results:
        text = f"""
🔍 **管理员搜索结果：** "{keyword}"

❌ 未找到匹配的文件码

💡 提示：
1. 检查关键词是否正确
2. 尝试更短的关键词
3. 可能还没有用户添加备注
"""
    else:
        text = f"""
🔍 **管理员搜索结果：** "{keyword}"

✅ 找到 {len(results)} 个匹配的文件码：

"""
        
        for i, result in enumerate(results[:15], 1):  # 最多显示15个
            # 截断长备注
            remark_display = result['remark']
            if len(remark_display) > 30:
                remark_display = remark_display[:27] + "..."
            
            text += f"\n**{i}. ** `{result['code']}`\n"
            text += f"├─ 📝 {remark_display}\n"
            if result.get('tags'):
                text += f"├─ 🏷️ 标签：{result['tags']}\n"
            text += f"├─ 用户ID：`{result['user_id']}`\n"
            text += f"├─ 文件：{result['file_count']}个\n"
            text += f"└─ 类型：{result['file_types']}\n"
        
        if len(results) > 15:
            text += f"\n... 还有 {len(results)-15} 个结果未显示"
        
        text += "\n💡 使用 `/userinfo 用户ID` 查看用户详情"
    
    bot.send_message(
        chat_id,
        text,
        parse_mode='Markdown'
    )

def handle_admin_activate_callback(call):
    """管理员激活VIP"""
    print(f"🎯 [回调处理] 收到回调: {call.data}")
    
    user_id = call.from_user.id
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    try:
        data_part = call.data.replace("admin_activate_", "")
        parts = data_part.split("_")
        
        print(f"📝 [回调处理] 分割后parts: {parts}")
        
        if len(parts) >= 3:
            target_user_id = int(parts[0])
            print(f"✅ [回调处理] 用户ID: {target_user_id}")
            
            try:
                months = int(parts[-1])
                print(f"✅ [回调处理] 月份参数: {months}")
            except:
                months = 1
                print(f"⚠️ [回调处理] 月份解析失败，使用默认值1")
            
            if len(parts) > 2:
                order_id = "_".join(parts[1:-1])
            else:
                order_id = parts[1] if len(parts) > 1 else "unknown"
            
            print(f"✅ [回调处理] 订单ID: {order_id}")
            print(f"✅ [回调处理] 最终月份: {months}")
            
        else:
            target_user_id = int(parts[0]) if len(parts) > 0 else 0
            order_id = parts[1] if len(parts) > 1 else "unknown"
            months = int(parts[2]) if len(parts) > 2 else 1
        
        if target_user_id == 0:
            bot.answer_callback_query(call.id, "❌ 用户ID解析失败", show_alert=True)
            return
        
        print(f"🔍 [回调处理] 开始从数据库获取套餐月份...")
        
        conn_local = None
        cursor_local = None
        try:
            conn_local = db_pool.get_connection()
            cursor_local = conn_local.cursor()
            
            cursor_local.execute('''
            SELECT vp.months 
            FROM payment_orders po
            JOIN vip_packages vp ON po.package_id = vp.id
            WHERE po.order_id = ?
            ''', (order_id,))
            
            row = cursor_local.fetchone()
            if row:
                actual_months = row[0]
                print(f"✅ [回调处理] 从数据库获取套餐月份: {actual_months}个月")
                months = actual_months
            else:
                print(f"⚠️ [回调处理] 未找到订单 {order_id}，使用回调中的月份: {months}")
                
                cursor_local.execute('''
                SELECT months FROM vip_packages WHERE id = ?
                ''', (months,))
                
                pkg_row = cursor_local.fetchone()
                if pkg_row:
                    months = pkg_row[0]
                    print(f"✅ [回调处理] 从套餐表获取月份: {months}个月")
                else:
                    print(f"⚠️ [回调处理] 也未找到套餐，使用默认值1")
                    months = 1
                    
        except Exception as db_error:
            print(f"❌ [回调处理] 数据库查询错误: {db_error}")
            months = 1
        
        if months <= 0:
            months = 1
            print(f"⚠️ [回调处理] 月份<=0，调整为1")
        elif months > 36:
            months = 36
            print(f"⚠️ [回调处理] 月份>36，限制为36")
        
        print(f"🎯 [回调处理] 最终确定月份: {months}个月")
        
        print(f"🚀 [回调处理] 开始激活VIP...")
        
        try:
            expire_time = vip_user_manager.activate_vip(target_user_id, months)
            
            conn_update = db_pool.get_connection()
            cursor_update = conn_update.cursor()
            try:
                cursor_update.execute('''
                UPDATE payment_orders 
                SET status = 'activated', activated = 1, activated_time = ?
                WHERE order_id = ?
                ''', (datetime.now().isoformat(), order_id))
                conn_update.commit()
                print(f"✅ [回调处理] 订单状态更新成功: {order_id}")
            except Exception as update_error:
                print(f"❌ [回调处理] 更新订单状态失败: {update_error}")
                conn_update.rollback()
            finally:
                cursor_update.close()
            
            days_left = (expire_time - datetime.now()).days
            
            bot.answer_callback_query(
                call.id,
                f"✅ VIP激活成功！\n用户: {target_user_id}\n时长: {months}个月\n到期: {expire_time.strftime('%Y-%m-%d')}\n剩余: {days_left}天",
                show_alert=True
            )
            
            try:
                bot.edit_message_text(
                    f"✅ **VIP已激活**\n\n"
                    f"• 用户ID: `{target_user_id}`\n"
                    f"• 订单号: `{order_id}`\n"
                    f"• 套餐时长: {months}个月\n"
                    f"• 到期时间: {expire_time.strftime('%Y-%m-%d %H:%M:%S')}\n"
                    f"• 剩余天数: {days_left}天\n"
                    f"• 激活时间: {datetime.now().strftime('%H:%M:%S')}",
                    chat_id=call.message.chat.id,
                    message_id=call.message.message_id,
                    parse_mode='Markdown'
                )
            except Exception as e:
                print(f"⚠️ [回调处理] 更新消息失败: {e}")
            
            notify_user_vip_activated(target_user_id, months, order_id)
            
        except Exception as activate_error:
            print(f"❌ [回调处理] VIP激活失败: {activate_error}")
            traceback.print_exc()
            bot.answer_callback_query(call.id, f"❌ VIP激活失败: {activate_error}", show_alert=True)
            
    except ValueError as e:
        print(f"❌ [回调处理] 数值转换错误: {e}")
        bot.answer_callback_query(call.id, f"❌ 数据格式错误: {e}", show_alert=True)
    except Exception as e:
        print(f"❌ [回调处理] 激活失败: {e}")
        traceback.print_exc()
        bot.answer_callback_query(call.id, f"❌ 激活失败: {e}", show_alert=True)

def send_page_info(user_id, chat_id, batch_info):
    """发送分页信息"""
    text = f"""
📦 **文件发送中...** ({batch_info['current_batch']}/{batch_info['total_batches']})

🔢 **文件码：** `{batch_info['code']}`
📊 **进度：** {batch_info['current_batch']}/{batch_info['total_batches']} 批
📋 **本批：** {len(batch_info['files'])} 个文件
📁 **总计：** {batch_info['total_files']} 个文件

{'⚡ VIP用户：无点击限制' if user_limit_manager.is_vip(user_id) else '⏰ 普通用户：5秒点击间隔'}
"""
    
    can_click, wait_time = file_send_paginator.can_click_next(user_id, chat_id)
    
    menu = file_send_paginator.create_dynamic_menu(
        current_batch=batch_info['current_batch'],
        total_batches=batch_info['total_batches'],
        can_click=can_click,
        remaining_seconds=wait_time
    )
    
    msg = bot.send_message(
        chat_id,
        text,
        parse_mode='Markdown',
        reply_markup=menu
    )
    
    file_send_paginator.set_last_message_id(user_id, chat_id, msg.message_id)
    
    if not can_click and wait_time > 0:
        file_send_paginator.register_waiting_message(user_id, chat_id, msg.message_id, wait_time)

def notify_user_vip_activated(user_id, months, order_id=None):
    """通知用户VIP已激活"""
    vip_info = vip_user_manager.get_vip_info(user_id)
    
    print(f"📢 [用户通知] 通知用户VIP激活: 用户ID={user_id}, 月份={months}")
    print(f"📊 [用户通知] VIP信息: 到期={vip_info['expire_date_str']}, 剩余天数={vip_info['days_left']}")
    
    text = f"""
🎉 **VIP激活成功！**

✅ **激活信息**
• 套餐时长：{months}个月
• 激活时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
• 到期时间：{vip_info['expire_date_str']}
• 剩余天数：{vip_info['days_left']}天
{f"• 订单号：`{order_id}`" if order_id else ""}

🎁 **已解锁特权**
• 无限解码次数
• 无限接收文件
• 无点击下一页限制
• 最多500个文件/包
• 单个文件最大500MB
• 批量下载功能
• 无广告体验

💝 **感谢您的支持！**
如有任何问题，请联系客服。
"""
    
    try:
        bot.send_message(
            user_id,
            text,
            parse_mode='Markdown',
            reply_markup=create_main_menu()
        )
        print(f"✅ [用户通知] 通知发送成功")
    except Exception as e:
        print(f"❌ [用户通知] 通知用户失败: {e}")

# ==================== 每日重置任务 ====================
def daily_reset_task():
    """每日重置任务"""
    while True:
        now = datetime.now()
        
        tomorrow = now.replace(hour=0, minute=0, second=0, microsecond=0) + timedelta(days=1)
        sleep_seconds = (tomorrow - now).total_seconds()
        
        print(f"⏰ 每日重置任务将在 {sleep_seconds/3600:.1f} 小时后执行")
        time.sleep(sleep_seconds)
        
        print("🔄 正在重置每日统计...")
        print("✅ 每日重置完成")

# 启动每日重置任务
reset_thread = threading.Thread(target=daily_reset_task, daemon=True)
reset_thread.start()
# ==================== 【新增】定期统计显示 ====================
def show_cache_stats():
    """显示缓存统计信息"""
    if hasattr(decode_cache, 'cache'):
        cache_size = len(decode_cache.cache)
        total_cached = decode_cache.total_cached
        print(f"📊 缓存统计: {cache_size}个文件码待刷新，累计{total_cached}次解码")
    return True

# 启动定期统计线程
def start_stats_thread():
    """启动统计显示线程"""
    def stats_worker():
        while True:
            time.sleep(60)  # 每分钟显示一次
            try:
                show_cache_stats()
            except Exception as e:
                print(f"统计显示错误: {e}")
    
    thread = threading.Thread(target=stats_worker, daemon=True)
    thread.start()
    print("📈 缓存统计线程已启动")

# 启动统计线程
start_stats_thread()
# ==================== 主程序 ====================
def main():
    try:
        print("=" * 60)
        print("🤖 文件码机器人 - 性能优化版")
        print("=" * 60)
        print("✅ 系统初始化完成")
        print(f"👷 工作线程: {task_processor.max_workers} 个")
        print(f"💾 数据库连接池: {db_pool.pool_size} 个连接")
        print(f"⚡ SQLite缓存: 256MB 内存 + 1GB 内存映射")
        print(f"🔄 解码缓存: 内存缓存 + 30秒自动刷新")
        print(f"📊 最大并发: 300+ 用户（优化后）")
        print("=" * 60)
        print(f"📦 文件分页发送: 已启用")
        print(f"🔍 智能文件码识别: 已启用")
        print(f"📝 文件备注功能: 已启用")
        print(f"🔍 备注搜索功能: 已启用")
        print(f"⭐ VIP用户限制系统: 已启用")
        print(f"🌐 跳转页面支付: 已启用")
        print(f"💰 支付方式: 微信/支付宝/USDT")
        
        # 检查支付配置
        if "your-domain.com" in PAYMENT_BASE_URL:
            print(f"⚠️ 支付页面配置: 需修改PAYMENT_BASE_URL")
            print(f"💡 请修改第28行的PAYMENT_BASE_URL为你的HTML页面地址")
        else:
            print(f"✅ 支付页面: {PAYMENT_BASE_URL}")
        
        print(f"👑 管理员: {len(ADMIN_USER_IDS)} 人")
        print(f"🗃️  数据库连接池: {db_pool.pool_size} 个连接")
        print(f"⏱️  API限流: {api_limiter.max_rps} 次/秒")
        
        print("=" * 60)
        print("🚀 机器人开始运行...")
        print("📱 使用 /start 开始")
        print("💡 普通用户限制：50解码/天，50文件/天")
        print("📝 备注功能：为文件码添加备注，方便搜索")
        print("🔍 搜索功能：/search 关键词 搜索备注")
        print("⭐ VIP特权：无限解码，无限文件")
        print("🌐 支付方式：点击按钮跳转到支付页面")
        print("=" * 60)
        
        # 设置机器人命令
        commands = [
            telebot.types.BotCommand("start", "开始使用机器人"),
            telebot.types.BotCommand("help", "获取帮助信息"),
]
        
        try:
            bot.set_my_commands(commands)
            print("✅ 机器人命令设置成功（仅保留start和help）")
        except Exception as e:
            print(f"⚠️ 设置命令失败: {e}")
        
        print("🤖 机器人正在启动...")
        bot.polling(none_stop=True, interval=0.1, timeout=30)
        
    except KeyboardInterrupt:
        print("\n🛑 用户中断")
        print("✅ 机器人停止")
        if db_pool:
            db_pool.close_all()
    except Exception as e:
        print(f"❌ 运行错误: {e}")
        traceback.print_exc()
        print("🔄 5秒后重启...")
        time.sleep(5)
        main()
# ==================== 【新增】套餐管理功能 ====================

def show_package_statistics(user_id, chat_id, message_id):
    """显示套餐统计"""
    if not is_admin(user_id):
        return
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    # 修复的SQL查询 - 使用正确的列名
    try:
        cursor.execute('''
        SELECT 
            vp.id,
            vp.name,
            vp.price_cny,
            vp.months,
            COUNT(po.order_id) as total_orders,
            SUM(CASE WHEN po.status = 'activated' THEN 1 ELSE 0 END) as activated_orders,
            SUM(CASE WHEN po.status = 'activated' THEN po.amount ELSE 0 END) as total_revenue
        FROM vip_packages vp
        LEFT JOIN payment_orders po ON vp.id = po.package_id
        GROUP BY vp.id
        ORDER BY total_revenue DESC
        ''')
    except sqlite3.OperationalError as e:
        print(f"❌ SQL查询错误: {e}")
        # 使用备用查询
        cursor.execute('''
        SELECT 
            vp.id,
            vp.name,
            vp.price_cny,
            vp.months
        FROM vip_packages vp
        ORDER BY vp.display_order
        ''')
        packages = cursor.fetchall()
        
        # 分别查询统计数据
        packages_stats = []
        for pkg in packages:
            pkg_id, name, price, months = pkg
            
            cursor.execute('''
            SELECT COUNT(order_id), SUM(amount) 
            FROM payment_orders 
            WHERE package_id = ? AND status = 'activated'
            ''', (pkg_id,))
            
            stats = cursor.fetchone()
            total_orders = stats[0] or 0
            revenue = stats[1] or 0
            
            packages_stats.append((pkg_id, name, price, months, total_orders, total_orders, revenue))
    else:
        packages_stats = cursor.fetchall()
    
    # 获取总销售额
    try:
        cursor.execute('SELECT SUM(amount) FROM payment_orders WHERE status = "activated"')
        total_revenue_result = cursor.fetchone()
        total_revenue = total_revenue_result[0] if total_revenue_result and total_revenue_result[0] else 0
    except:
        total_revenue = 0
    
    text = "📊 **VIP套餐销售统计**\n\n"
    text += f"💰 **总销售额：** {total_revenue:.2f}元\n\n"
    
    if not packages_stats:
        text += "📭 暂无销售数据"
    else:
        text += "**套餐销售排行：**\n"
        for i, stats in enumerate(packages_stats, 1):
            if len(stats) >= 7:
                pkg_id, name, price, months, total_orders, activated_orders, revenue = stats
            elif len(stats) >= 4:
                pkg_id, name, price, months = stats[:4]
                total_orders = stats[4] if len(stats) > 4 else 0
                activated_orders = total_orders
                revenue = stats[5] if len(stats) > 5 else 0
            
            text += f"\n**{i}. {name}**\n"
            text += f"├─ 价格：{price}元 ({months}个月)\n"
            text += f"├─ 总订单：{total_orders}个\n"
            text += f"├─ 已激活：{activated_orders}个\n"
            text += f"└─ 销售额：{revenue or 0:.2f}元\n"
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("⬅️ 返回套餐管理", callback_data="manage_packages"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)

def ask_for_new_price(user_id, chat_id, package_id, message_id):
    """询问新的套餐价格"""
    if not is_admin(user_id):
        return
    
    packages = vip_payment_system.package_manager.get_all_packages()
    package = next((p for p in packages if p['id'] == package_id), None)
    
    if not package:
        return
    
    text = f"""
💰 **修改套餐价格**

套餐：{package['name']}
当前价格：{package['price_cny']}元

请输入新的价格（元）：
例如：25.00、50、100.50

⚠️ 必须是数字，最多保留2位小数
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("❌ 取消", callback_data=f"manage_package_{package_id}")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    user_sessions = getattr(bot, 'user_sessions', {})
    user_sessions[user_id] = {
        'action': 'edit_package_price',
        'package_id': package_id,
        'package_name': package['name']
    }
    bot.user_sessions = user_sessions

def ask_for_new_name(user_id, chat_id, package_id, message_id):
    """询问新的套餐名称"""
    if not is_admin(user_id):
        return
    
    packages = vip_payment_system.package_manager.get_all_packages()
    package = next((p for p in packages if p['id'] == package_id), None)
    
    if not package:
        return
    
    text = f"""
✏️ **修改套餐名称**

当前套餐：{package['name']}

请输入新的套餐名称：
例如："25元包月VIP"、"90元包季VIP"

⚠️ 最多50个字符
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("❌ 取消", callback_data=f"manage_package_{package_id}")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    user_sessions = getattr(bot, 'user_sessions', {})
    user_sessions[user_id] = {
        'action': 'edit_package_name',
        'package_id': package_id,
        'current_name': package['name']
    }
    bot.user_sessions = user_sessions

def ask_for_new_months(user_id, chat_id, package_id, message_id):
    """询问新的套餐时长"""
    if not is_admin(user_id):
        return
    
    packages = vip_payment_system.package_manager.get_all_packages()
    package = next((p for p in packages if p['id'] == package_id), None)
    
    if not package:
        return
    
    text = f"""
📅 **修改套餐时长**

套餐：{package['name']}
当前时长：{package['months']}个月

请输入新的时长（月数）：
例如：1、3、6、12

⚠️ 必须是整数，建议1-36个月
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("❌ 取消", callback_data=f"manage_package_{package_id}")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    user_sessions = getattr(bot, 'user_sessions', {})
    user_sessions[user_id] = {
        'action': 'edit_package_months',
        'package_id': package_id,
        'package_name': package['name']
    }
    bot.user_sessions = user_sessions

def ask_for_new_description(user_id, chat_id, package_id, message_id):
    """询问新的套餐描述"""
    if not is_admin(user_id):
        return
    
    packages = vip_payment_system.package_manager.get_all_packages()
    package = next((p for p in packages if p['id'] == package_id), None)
    
    if not package:
        return
    
    text = f"""
📝 **修改套餐描述**

套餐：{package['name']}
当前描述：{package['description']}

请输入新的套餐描述：
例如："适合短期使用，性价比最高"

⚠️ 最多200个字符
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("❌ 取消", callback_data=f"manage_package_{package_id}")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    user_sessions = getattr(bot, 'user_sessions', {})
    user_sessions[user_id] = {
        'action': 'edit_package_description',
        'package_id': package_id,
        'package_name': package['name']
    }
    bot.user_sessions = user_sessions

def ask_for_new_order(user_id, chat_id, package_id, message_id):
    """询问新的排序号"""
    if not is_admin(user_id):
        return
    
    packages = vip_payment_system.package_manager.get_all_packages()
    package = next((p for p in packages if p['id'] == package_id), None)
    
    if not package:
        return
    
    text = f"""
🔢 **修改套餐排序**

套餐：{package['name']}
当前排序：{package['display_order']}

请输入新的排序号：
数字越小，显示越靠前

⚠️ 必须是整数，建议1-100
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("❌ 取消", callback_data=f"manage_package_{package_id}")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    user_sessions = getattr(bot, 'user_sessions', {})
    user_sessions[user_id] = {
        'action': 'edit_package_order',
        'package_id': package_id,
        'package_name': package['name']
    }
    bot.user_sessions = user_sessions

def toggle_package_status(user_id, chat_id, package_id, message_id):
    """切换套餐状态"""
    if not is_admin(user_id):
        return
    
    success, message = vip_payment_system.package_manager.toggle_package_status(package_id)
    
    if success:
        vip_payment_system.show_package_detail_management(user_id, chat_id, package_id, message_id)
    else:
        bot.send_message(chat_id, message)

def confirm_delete_package(user_id, chat_id, package_id, message_id):
    """确认删除套餐"""
    if not is_admin(user_id):
        return
    
    packages = vip_payment_system.package_manager.get_all_packages()
    package = next((p for p in packages if p['id'] == package_id), None)
    
    if not package:
        return
    
    text = f"""
🗑️ **确认删除套餐**

套餐：{package['name']}
价格：{package['price_cny']}元
时长：{package['months']}个月

⚠️ **警告：**
删除套餐将：
1. 无法恢复
2. 不影响已购买用户
3. 新用户无法购买此套餐

确认要删除吗？
"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(
        telebot.types.InlineKeyboardButton("✅ 确认删除", callback_data=f"confirm_delete_pkg_{package_id}"),
        telebot.types.InlineKeyboardButton("❌ 取消", callback_data=f"manage_package_{package_id}")
    )
    
    try:
        bot.edit_message_text(
            text,
            chat_id=chat_id,
            message_id=message_id,
            parse_mode='Markdown',
            reply_markup=markup
        )
    except:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data.startswith('confirm_delete_pkg_'))
def handle_confirm_delete_package(call):
    """确认删除套餐"""
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    message_id = call.message.message_id
    
    if not is_admin(user_id):
        bot.answer_callback_query(call.id, "❌ 权限不足", show_alert=True)
        return
    
    package_id = int(call.data.replace("confirm_delete_pkg_", ""))
    
    conn = db_pool.get_connection()
    cursor = conn.cursor()
    
    try:
        cursor.execute('''
        SELECT COUNT(*) FROM payment_orders 
        WHERE package_id = ? AND status IN ('pending', 'paid')
        ''', (package_id,))
        
        pending_orders = cursor.fetchone()[0] or 0
        
        if pending_orders > 0:
            bot.answer_callback_query(
                call.id,
                f"❌ 此套餐还有{pending_orders}个待处理订单，请先处理订单再删除套餐",
                show_alert=True
            )
            return
        
        cursor.execute('DELETE FROM vip_packages WHERE id = ?', (package_id,))
        conn.commit()
        
        bot.answer_callback_query(call.id, "✅ 套餐已删除", show_alert=True)
        vip_payment_system.show_package_management(user_id, chat_id, message_id)
        
    except Exception as e:
        print(f"删除套餐失败: {e}")
        bot.answer_callback_query(call.id, f"❌ 删除失败: {e}", show_alert=True)
        if conn:
            conn.rollback()

# 添加消息处理器来处理套餐管理输入
@bot.message_handler(func=lambda m: True)
def handle_all_messages(message):
    """处理所有未被其他处理器处理的消息"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    text = message.text.strip() if message.text else ""
    
    print(f"🔧 [通用处理器] 处理消息: {text[:30]}...")
    
    # 检查用户是否有套餐管理session
    user_sessions = getattr(bot, 'user_sessions', {})
    session = user_sessions.get(user_id, {})
    
    if not session:
        print(f"⏭️ [通用处理器] 用户 {user_id} 没有session，跳过")
        return
    
    action = session.get('action')
    
    if action == 'add_new_package':
        print(f"🎯 [通用处理器] 处理添加套餐: {text}")
        # 现在只需要传入message
        handle_add_package_input(message)  # 只传1个参数
        
    elif action == 'edit_package_price':
        print(f"🎯 [通用处理器] 处理修改价格: {text}")
        handle_price_input(user_id, chat_id, text, session)
        # 清理session
        if user_id in user_sessions:
            del user_sessions[user_id]
            bot.user_sessions = user_sessions
            
    elif action == 'edit_package_name':
        print(f"🎯 [通用处理器] 处理修改名称: {text}")
        handle_name_input(user_id, chat_id, text, session)
        if user_id in user_sessions:
            del user_sessions[user_id]
            bot.user_sessions = user_sessions
            
    elif action == 'edit_package_months':
        print(f"🎯 [通用处理器] 处理修改时长: {text}")
        handle_months_input(user_id, chat_id, text, session)
        if user_id in user_sessions:
            del user_sessions[user_id]
            bot.user_sessions = user_sessions
            
    elif action == 'edit_package_description':
        print(f"🎯 [通用处理器] 处理修改描述: {text}")
        handle_description_input(user_id, chat_id, text, session)
        if user_id in user_sessions:
            del user_sessions[user_id]
            bot.user_sessions = user_sessions
            
    elif action == 'edit_package_order':
        print(f"🎯 [通用处理器] 处理修改排序: {text}")
        handle_order_input(user_id, chat_id, text, session)
        if user_id in user_sessions:
            del user_sessions[user_id]
            bot.user_sessions = user_sessions
            
    else:
        print(f"❓ [通用处理器] 未知action: {action}")

def handle_price_input(user_id, chat_id, text, session):
    """处理价格输入"""
    package_id = session.get('package_id')
    package_name = session.get('package_name')
    
    try:
        new_price = float(text)
        if new_price <= 0:
            raise ValueError("价格必须大于0")
        
        success, message = vip_payment_system.package_manager.update_package_price(package_id, new_price)
        
        if success:
            markup = telebot.types.InlineKeyboardMarkup()
            markup.add(
                telebot.types.InlineKeyboardButton("⬅️ 返回管理", callback_data=f"manage_package_{package_id}")
            )
            
            bot.send_message(
                chat_id,
                f"✅ 套餐价格修改成功！\n\n"
                f"**{package_name}**\n"
                f"新价格：{new_price}元",
                parse_mode='Markdown',
                reply_markup=markup
            )
        else:
            bot.send_message(chat_id, message)
            
    except ValueError:
        bot.send_message(
            chat_id,
            "❌ 价格格式错误！请输入有效的数字（例如：25.00）"
        )

def handle_name_input(user_id, chat_id, text, session):
    """处理名称输入"""
    package_id = session.get('package_id')
    
    if not text or len(text) > 50:
        bot.send_message(
            chat_id,
            "❌ 套餐名称不能为空且不能超过50个字符"
        )
        return
    
    success, message = vip_payment_system.package_manager.update_package_name(package_id, text)
    
    if success:
        markup = telebot.types.InlineKeyboardMarkup()
        markup.add(
            telebot.types.InlineKeyboardButton("⬅️ 返回管理", callback_data=f"manage_package_{package_id}")
        )
        
        bot.send_message(
            chat_id,
            f"✅ 套餐名称修改成功！\n\n新名称：{text}",
            parse_mode='Markdown',
            reply_markup=markup
        )
    else:
        bot.send_message(chat_id, message)

def handle_months_input(user_id, chat_id, text, session):
    """处理时长输入"""
    package_id = session.get('package_id')
    package_name = session.get('package_name')
    
    try:
        new_months = int(text)
        if new_months <= 0 or new_months > 120:
            raise ValueError("时长必须在1-120个月之间")
        
        success, message = vip_payment_system.package_manager.update_package_months(package_id, new_months)
        
        if success:
            markup = telebot.types.InlineKeyboardMarkup()
            markup.add(
                telebot.types.InlineKeyboardButton("⬅️ 返回管理", callback_data=f"manage_package_{package_id}")
            )
            
            bot.send_message(
                chat_id,
                f"✅ 套餐时长修改成功！\n\n"
                f"**{package_name}**\n"
                f"新时长：{new_months}个月",
                parse_mode='Markdown',
                reply_markup=markup
            )
        else:
            bot.send_message(chat_id, message)
            
    except ValueError:
        bot.send_message(
            chat_id,
            "❌ 时长格式错误！请输入1-120之间的整数"
        )

def handle_description_input(user_id, chat_id, text, session):
    """处理描述输入"""
    package_id = session.get('package_id')
    package_name = session.get('package_name')
    
    if not text or len(text) > 200:
        bot.send_message(
            chat_id,
            "❌ 描述不能为空且不能超过200个字符"
        )
        return
    
    success, message = vip_payment_system.package_manager.update_package_description(package_id, text)
    
    if success:
        markup = telebot.types.InlineKeyboardMarkup()
        markup.add(
            telebot.types.InlineKeyboardButton("⬅️ 返回管理", callback_data=f"manage_package_{package_id}")
        )
        
        bot.send_message(
            chat_id,
            f"✅ 套餐描述修改成功！\n\n"
            f"**{package_name}**\n"
            f"新描述：{text}",
            parse_mode='Markdown',
            reply_markup=markup
        )
    else:
        bot.send_message(chat_id, message)

def handle_order_input(user_id, chat_id, text, session):
    """处理排序输入"""
    package_id = session.get('package_id')
    package_name = session.get('package_name')
    
    try:
        new_order = int(text)
        if new_order < 0 or new_order > 999:
            raise ValueError("排序号必须在0-999之间")
        
        success, message = vip_payment_system.package_manager.update_package_order(package_id, new_order)
        
        if success:
            markup = telebot.types.InlineKeyboardMarkup()
            markup.add(
                telebot.types.InlineKeyboardButton("⬅️ 返回管理", callback_data=f"manage_package_{package_id}")
            )
            
            bot.send_message(
                chat_id,
                f"✅ 套餐排序修改成功！\n\n"
                f"**{package_name}**\n"
                f"新排序：{new_order}",
                parse_mode='Markdown',
                reply_markup=markup
            )
        else:
            bot.send_message(chat_id, message)
            
    except ValueError:
        bot.send_message(
            chat_id,
            "❌ 排序号格式错误！请输入0-999之间的整数"
        )

def handle_add_package_input(message):
    """处理添加套餐的输入"""
    user_id = message.from_user.id
    chat_id = message.chat.id
    text = message.text.strip() if message.text else ""
    
    print(f"🎯 [添加套餐] 开始处理 - 用户: {user_id}, 文本: {text}")
    
    # 获取用户session
    user_sessions = getattr(bot, 'user_sessions', {})
    
    # 确保user_sessions存在
    if not hasattr(bot, 'user_sessions'):
        bot.user_sessions = {}
        user_sessions = {}
    
    if user_id not in user_sessions:
        print(f"❌ [添加套餐] 错误：用户 {user_id} 没有session")
        bot.send_message(chat_id, "❌ 会话已过期，请重新开始添加套餐", reply_markup=create_main_menu())
        return
    
    session = user_sessions[user_id]
    
    if not session or session.get('action') != 'add_new_package':
        print(f"❌ [添加套餐] 错误：无效的session或action")
        bot.send_message(chat_id, "❌ 无效的操作，请重新开始", reply_markup=create_main_menu())
        return
    
    step = session.get('step', 1)
    package_data = session.get('package_data', {})
    
    print(f"[添加套餐输入] 步骤 {step}, 当前数据: {package_data}")
    
    # 第一步：获取套餐名称
    if step == 1:
        if not text or len(text) > 50:
            bot.send_message(chat_id, "❌ 套餐名称不能为空且不能超过50个字符，请重新输入：")
            return
        
        package_data['name'] = text
        session['step'] = 2
        session['package_data'] = package_data
        user_sessions[user_id] = session
        bot.user_sessions = user_sessions
        
        bot.send_message(
            chat_id,
            f"✅ **步骤 {step}/5 完成**\n\n"
            f"• 套餐名称：{text}\n\n"
            f"**第2步：价格**\n"
            f"请输入价格（元），例如：30.00：",
            parse_mode='Markdown'
        )
        
    elif step == 2:
        # 第二步：获取价格
        try:
            # 移除可能的"元"字和其他非数字字符
            price_text = text.replace('元', '').replace('￥', '').replace('$', '').strip()
            price = float(price_text)
            
            if price <= 0:
                bot.send_message(chat_id, "❌ 价格必须大于0，请重新输入：")
                return
            
            package_data['price'] = price
            session['step'] = 3
            session['package_data'] = package_data
            user_sessions[user_id] = session
            bot.user_sessions = user_sessions
            
            bot.send_message(
                chat_id,
                f"✅ **步骤 {step}/5 完成**\n\n"
                f"• 价格：{price}元\n\n"
                f"**第3步：时长**\n"
                f"请输入时长（月数），例如：1：",
                parse_mode='Markdown'
            )
            
        except ValueError:
            bot.send_message(chat_id, "❌ 价格格式错误！请输入有效的数字（例如：25.00），请重新输入：")
    
    elif step == 3:
        # 第三步：获取时长
        try:
            months = int(text)
            if months <= 0 or months > 120:
                bot.send_message(chat_id, "❌ 时长必须在1-120个月之间，请重新输入：")
                return
            
            package_data['months'] = months
            session['step'] = 4
            session['package_data'] = package_data
            user_sessions[user_id] = session
            bot.user_sessions = user_sessions
            
            bot.send_message(
                chat_id,
                f"✅ **步骤 {step}/5 完成**\n\n"
                f"• 时长：{months}个月\n\n"
                f"**第4步：描述**\n"
                f"请输入套餐描述：",
                parse_mode='Markdown'
            )
            
        except ValueError:
            bot.send_message(chat_id, "❌ 时长格式错误！请输入1-120之间的整数，请重新输入：")
            
    elif step == 4:
        # 第四步：获取描述
        if not text or len(text) > 200:
            bot.send_message(chat_id, "❌ 描述不能为空且不能超过200个字符，请重新输入：")
            return
        
        package_data['description'] = text
        session['step'] = 5
        session['package_data'] = package_data
        user_sessions[user_id] = session
        bot.user_sessions = user_sessions
        
        bot.send_message(
            chat_id,
            f"✅ **步骤 {step}/5 完成**\n\n"
            f"• 描述：{text}\n\n"
            f"**第5步：排序号**\n"
            f"请输入排序号（数字越小越靠前），例如：5：",
            parse_mode='Markdown'
        )
        
    elif step == 5:
        # 第五步：获取排序号并保存
        try:
            display_order = int(text)
            if display_order < 0 or display_order > 999:
                bot.send_message(chat_id, "❌ 排序号必须在0-999之间，请重新输入：")
                return
            
            package_data['display_order'] = display_order
            
            # 保存到数据库
            conn = db_pool.get_connection()
            cursor = conn.cursor()
            
            try:
                cursor.execute('''
                INSERT INTO vip_packages 
                (name, price_cny, months, description, display_order, is_active, created_at, updated_at)
                VALUES (?, ?, ?, ?, ?, 1, ?, ?)
                ''', (
                    package_data['name'],
                    package_data['price'],
                    package_data['months'],
                    package_data['description'],
                    package_data['display_order'],
                    datetime.now().isoformat(),
                    datetime.now().isoformat()
                ))
                
                conn.commit()
                
                # 清理session
                if user_id in user_sessions:
                    del user_sessions[user_id]
                    bot.user_sessions = user_sessions
                
                # 显示成功消息
                markup = telebot.types.InlineKeyboardMarkup()
                markup.add(
                    telebot.types.InlineKeyboardButton("📋 查看套餐", callback_data="manage_packages"),
                    telebot.types.InlineKeyboardButton("➕ 继续添加", callback_data="add_new_package")
                )
                
                success_text = f"""
✅ **新套餐添加成功！**

📋 **套餐详情：**
• **名称：** {package_data['name']}
• **价格：** {package_data['price']}元
• **时长：** {package_data['months']}个月
• **描述：** {package_data['description']}
• **排序：** {package_data['display_order']}

🎉 套餐已自动激活，用户现在可以购买此套餐。
"""
                
                bot.send_message(
                    chat_id,
                    success_text,
                    parse_mode='Markdown',
                    reply_markup=markup
                )
                
            except Exception as e:
                conn.rollback()
                print(f"[错误] 保存套餐失败: {e}")
                bot.send_message(chat_id, f"❌ 保存失败: {str(e)}")
                
        except ValueError:
            bot.send_message(chat_id, "❌ 排序号格式错误！请输入0-999之间的整数，请重新输入：")
# ==================== 支付方式管理相关函数 ====================

def show_payment_methods_management(user_id, chat_id, message_id=None):
    """显示支付方式管理菜单"""
    if not is_admin(user_id):
        bot.send_message(chat_id, "❌ 权限不足", reply_markup=create_main_menu())
        return
    
    methods = payment_method_manager.get_all_methods()
    
    if not methods:
        text = "📭 暂无支付方式配置"
    else:
        text = "💰 **支付方式管理**\n\n"
        for method in methods:
            status = "🟢" if method['is_enabled'] else "🔴"
            text += f"{status} **{method['method_name']}**\n"
            text += f"├─ ID：{method['method_id']}\n"
            text += f"└─ 排序：{method['display_order']}\n\n"
    
    text += "请选择要管理的支付方式："
    
    markup = telebot.types.InlineKeyboardMarkup(row_width=2)
    for method in methods:
        btn_text = f"💳 {method['method_name']}"
        markup.add(telebot.types.InlineKeyboardButton(btn_text, callback_data=f"manage_payment_method_{method['method_id']}"))
    
    markup.add(
        telebot.types.InlineKeyboardButton("⬅️ 返回管理面板", callback_data="back_to_stats"),
        telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main")
    )
    
    if message_id:
        try:
            bot.edit_message_text(text, chat_id=chat_id, message_id=message_id, parse_mode='Markdown', reply_markup=markup)
        except:
            bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    else:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)

def show_payment_method_detail_management(user_id, chat_id, method_id, message_id):
    """显示支付方式详情管理"""
    if not is_admin(user_id):
        return
    
    methods = payment_method_manager.get_all_methods()
    method = next((m for m in methods if m['method_id'] == method_id), None)
    
    if not method:
        return
    
    status = "🟢 已启用" if method['is_enabled'] else "🔴 已禁用"
    
    text = f"""💳 **支付方式详情管理**

**{method['method_name']}**
├─ {status}
├─ 支付ID：{method['method_id']}
└─ 排序：{method['display_order']}

请选择要修改的项目："""
    
    markup = telebot.types.InlineKeyboardMarkup(row_width=2)
    markup.add(
        telebot.types.InlineKeyboardButton("✏️ 修改名称", callback_data=f"edit_payment_name_{method_id}"),
        telebot.types.InlineKeyboardButton("🔢 修改排序", callback_data=f"edit_payment_order_{method_id}")
    )
    markup.add(
        telebot.types.InlineKeyboardButton("🔄 启用/禁用", callback_data=f"toggle_payment_status_{method_id}"),
        telebot.types.InlineKeyboardButton("⬅️ 返回列表", callback_data="manage_payment_methods")
    )
    markup.add(telebot.types.InlineKeyboardButton("🏠 返回菜单", callback_data="back_to_main"))
    
    try:
        bot.edit_message_text(text, chat_id=chat_id, message_id=message_id, parse_mode='Markdown', reply_markup=markup)
    except:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)

def ask_for_new_payment_name(user_id, chat_id, method_id, message_id):
    """询问新的支付方式名称"""
    if not is_admin(user_id):
        return
    
    methods = payment_method_manager.get_all_methods()
    method = next((m for m in methods if m['method_id'] == method_id), None)
    
    if not method:
        return
    
    text = f"""✏️ **修改支付方式名称**

当前名称：{method['method_name']}
支付ID：{method['method_id']}

请输入新的支付方式名称：
例如："微信扫码支付"、"支付宝转账"

⚠️ 最多20个字符"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(telebot.types.InlineKeyboardButton("❌ 取消", callback_data=f"manage_payment_method_{method_id}"))
    
    try:
        bot.edit_message_text(text, chat_id=chat_id, message_id=message_id, parse_mode='Markdown', reply_markup=markup)
    except:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    user_sessions = getattr(bot, 'user_sessions', {})
    user_sessions[user_id] = {'action': 'edit_payment_name', 'method_id': method_id}
    bot.user_sessions = user_sessions

def ask_for_new_payment_order(user_id, chat_id, method_id, message_id):
    """询问新的支付方式排序"""
    if not is_admin(user_id):
        return
    
    methods = payment_method_manager.get_all_methods()
    method = next((m for m in methods if m['method_id'] == method_id), None)
    
    if not method:
        return
    
    text = f"""🔢 **修改支付方式排序**

支付方式：{method['method_name']}
当前排序：{method['display_order']}

请输入新的排序号：
数字越小，显示越靠前

⚠️ 必须是整数，建议1-100"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    markup.add(telebot.types.InlineKeyboardButton("❌ 取消", callback_data=f"manage_payment_method_{method_id}"))
    
    try:
        bot.edit_message_text(text, chat_id=chat_id, message_id=message_id, parse_mode='Markdown', reply_markup=markup)
    except:
        bot.send_message(chat_id, text, parse_mode='Markdown', reply_markup=markup)
    
    user_sessions = getattr(bot, 'user_sessions', {})
    user_sessions[user_id] = {'action': 'edit_payment_order', 'method_id': method_id}
    bot.user_sessions = user_sessions

def handle_payment_name_input(user_id, chat_id, text, session):
    """处理支付方式名称输入"""
    method_id = session.get('method_id')
    
    if not text or len(text) > 20:
        bot.send_message(chat_id, "❌ 支付方式名称不能为空且不能超过20个字符")
        return
    
    success, message = payment_method_manager.update_method_name(method_id, text)
    
    if success:
        markup = telebot.types.InlineKeyboardMarkup()
        markup.add(telebot.types.InlineKeyboardButton("⬅️ 返回管理", callback_data=f"manage_payment_method_{method_id}"))
        bot.send_message(chat_id, f"✅ 支付方式名称修改成功！\n\n新名称：{text}", parse_mode='Markdown', reply_markup=markup)
    else:
        bot.send_message(chat_id, message)

def handle_payment_order_input(user_id, chat_id, text, session):
    """处理支付方式排序输入"""
    method_id = session.get('method_id')
    
    try:
        new_order = int(text)
        if new_order < 0 or new_order > 100:
            raise ValueError("排序号必须在0-100之间")
        
        success, message = payment_method_manager.update_method_order(method_id, new_order)
        
        if success:
            markup = telebot.types.InlineKeyboardMarkup()
            markup.add(telebot.types.InlineKeyboardButton("⬅️ 返回管理", callback_data=f"manage_payment_method_{method_id}"))
            bot.send_message(chat_id, f"✅ 支付方式排序修改成功！\n\n新排序：{new_order}", parse_mode='Markdown', reply_markup=markup)
        else:
            bot.send_message(chat_id, message)
            
    except ValueError:
        bot.send_message(chat_id, "❌ 排序号格式错误！请输入0-100之间的整数")            
if __name__ == "__main__":
    main()
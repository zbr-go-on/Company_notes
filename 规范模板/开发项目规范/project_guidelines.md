# project_guidelines.md

**Python 项目开发规范（适用于 Cursor 工具与工程化项目）**
版本：1.0
更新人：Zhong 项目组

---

## 1. 项目目录结构规范

统一采用简洁、分层、易维护的结构：

```plaintext
project_root/
│
├── src/
│   ├── config/
│   │   ├── settings.py           # 环境变量、多数据库配置加载
│   │   └── logging_config.py     # 日志配置
│   │
│   ├── core/
│   │   ├── database.py           # 数据库连接类 + 动态查询封装 + 多数据库支持
│   │   └── exceptions.py         # 自定义业务异常、数据库异常
│   │
│   ├── services/
│   │   └── ...                   # 业务逻辑层
│   │
│   ├── utils/
│   │   └── validators.py         # 输入校验工具类
│   │
│   │
│   ├── main.py                   # 程序入口
│   └── __init__.py
│
├── tests/
│
├── .env                          # 环境变量配置
├── .env.example                  # 环境变量示例
├── requirements.txt
└── README.md
```

---

## 2. 环境变量（.env）规范

敏感信息禁止写入代码，全部放在 `.env`。支持多套数据库配置（如 db1 / db2 / analytics …）。

示例：

```
APP_ENV=dev
LOG_LEVEL=INFO

# 第一套数据库
DB1_HOST=127.0.0.1
DB1_PORT=3306
DB1_USER=root
DB1_PASSWORD=123456
DB1_DATABASE=app_db

# 第二套数据库
DB2_HOST=127.0.0.1
DB2_PORT=3307
DB2_USER=app
DB2_PASSWORD=abc
DB2_DATABASE=log_db
```

同时提供 `.env.example` 防止敏感信息泄露：

```
APP_ENV=
LOG_LEVEL=

DB1_HOST=
DB1_PORT=
DB1_USER=
DB1_PASSWORD=
DB1_DATABASE=

DB2_HOST=
DB2_PORT=
DB2_USER=
DB2_PASSWORD=
DB2_DATABASE=
```

---

## 3. 配置加载（settings.py）

支持多数据库加载，按前缀动态读取：

```python
from dotenv import load_dotenv
import os

load_dotenv()

class DBConfig:
    def __init__(self, prefix: str):
        self.host = os.getenv(f"{prefix}_HOST")
        self.port = int(os.getenv(f"{prefix}_PORT"))
        self.user = os.getenv(f"{prefix}_USER")
        self.password = os.getenv(f"{prefix}_PASSWORD")
        self.database = os.getenv(f"{prefix}_DATABASE")

class Settings:
    DB_CONFIGS = {
        "db1": DBConfig("DB1"),
        "db2": DBConfig("DB2"),
    }
    DEFAULT_DB = "db1"

    ENV = os.getenv("APP_ENV", "dev")
    LOG_LEVEL = os.getenv("LOG_LEVEL", "INFO")

settings = Settings()
```

---

## 4. 日志系统规范（logging_config.py）

日志必须：

* 记录时间、级别、模块、行号、内容
* 同时输出到文件与控制台
* 自动滚动
* 错误必须记录 traceback

```python
import logging
import os
from logging.handlers import RotatingFileHandler

log_dir = "logs"
os.makedirs(log_dir, exist_ok=True)

log_file = os.path.join(log_dir, "app.log")

formatter = logging.Formatter(
    "[%(asctime)s] [%(levelname)s] %(filename)s:%(lineno)d - %(message)s"
)

file_handler = RotatingFileHandler(log_file, maxBytes=5*1024*1024, backupCount=5)
file_handler.setFormatter(formatter)

console_handler = logging.StreamHandler()
console_handler.setFormatter(formatter)

logger = logging.getLogger("app")
logger.setLevel(logging.INFO)
logger.addHandler(file_handler)
logger.addHandler(console_handler)
```

---

## 5. 异常规范（exceptions.py）

所有异常必须记录日志，不可吞掉错误。

```python
class BusinessError(Exception):
    """业务逻辑异常"""
    pass

class DatabaseError(Exception):
    """数据库执行异常"""
    pass
```

---

## 6. 多数据库 + 动态查询规范（database.py）

数据库类可选择配置（db1 / db2）。

具备：

* 自定义异常
* 自动日志
* 动态参数化查询
* 可扩展多套 DB

```python
import pymysql
from pymysql.cursors import DictCursor
from config.settings import settings
from config.logging_config import logger
from core.exceptions import DatabaseError

class Database:
    def __init__(self, db_key=None):
        db_key = db_key or settings.DEFAULT_DB
        cfg = settings.DB_CONFIGS.get(db_key)

        if not cfg:
            raise ValueError(f"未找到数据库配置: {db_key}")

        try:
            self.conn = pymysql.connect(
                host=cfg.host,
                port=cfg.port,
                user=cfg.user,
                password=cfg.password,
                database=cfg.database,
                cursorclass=DictCursor
            )
            logger.info(f"数据库[{db_key}] 连接成功 → {cfg.host}:{cfg.port}")
        except Exception as e:
            logger.error(f"数据库[{db_key}] 连接失败: {e}", exc_info=True)
            raise DatabaseError("数据库连接失败")

    def fetch(self, sql, params=None):
        logger.debug(f"执行 SQL: {sql} | 参数={params}")
        try:
            with self.conn.cursor() as cursor:
                cursor.execute(sql, params or ())
                return cursor.fetchall()
        except Exception as e:
            logger.error(f"查询失败: {e}", exc_info=True)
            raise DatabaseError("SQL 查询异常")

    def execute(self, sql, params=None):
        logger.debug(f"执行修改 SQL: {sql} | 参数={params}")
        try:
            with self.conn.cursor() as cursor:
                cursor.execute(sql, params or ())
            self.conn.commit()
        except Exception as e:
            logger.error(f"执行失败: {e}", exc_info=True)
            self.conn.rollback()
            raise DatabaseError("SQL 执行异常")
```

---

## 7. 动态查询示例（服务层）

```python
def find_orders(user_id=None, status=None, db_key="db1"):
    sql = "SELECT * FROM orders WHERE 1=1"
    params = []

    if user_id:
        sql += " AND user_id = %s"
        params.append(user_id)

    if status:
        sql += " AND status = %s"
        params.append(status)

    db = Database(db_key)
    return db.fetch(sql, params)
```

特点：

* 使用参数化防 SQL 注入
* 支持可选条件
* DB 切换方便

---


### main.py（项目入口）

```python

```

---



---

## 10. 自动化测试规范

```
tests/
  test_database.py
  test_cli.py
  ...
```

测试要点：

* 数据库连接
* 动态查询逻辑
* 多数据库切换
* CLI 命令是否可运行
* 异常是否正确抛出

---

## 11. README 要求

必须包含：

* 安装依赖
* 配置 `.env`
* CLI 用法
* 日志目录
* 数据库配置说明

示例：

```
pip install -r requirements.txt
cp .env.example .env
python src/main.py --help
```

---

## 12. 后续扩展（可选）

本规范可继续扩展支持：

* FastAPI REST 服务
* ORM（SQLAlchemy）
* Redis 缓存
* RabbitMQ / Kafka
* APScheduler 定时任务
* CI/CD
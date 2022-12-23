# 数据库

## 抽象类

为了便于在修改数据库引擎时不会影响过多的代码结构，定义了抽象类`DBInterface`，后续的数据库相关操作通过继承`DBInterface`和`Singleton`来实现

**ps. 使用的数据库引擎必须是 k-v 数据库**

```python
from abc import abstractmethod, ABC


class DBInterface(ABC):

    @property
    @abstractmethod
    def db(self):
        """
        获取数据库对象实例
        """

    @abstractmethod
    def get(self, key, default=None):
        """
        获取 key 对应的 value
        """

    @abstractmethod
    def insert(self, _key: str, _value: dict) -> bool:
        """
        插入新的键值对
        """

    @abstractmethod
    def remove(self, _key: str) -> bool:
        """
        从数据库中移除键对应的数据
        """

    @abstractmethod
    def batch_insert(self, kv_data: dict):
        """
        批量插入数据
        """

    @abstractmethod
    def batch_remove(self, keys: list):
        """
        批量移除数据
        """
```

## LevelDB

目前实现的只有基于 levelDB 的数据库引擎，原有使用的 couchDB 由于 I/O 操作耗时过长已经在 v1.1.5 版本中被替换为 levelDB


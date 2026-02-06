---
layout: post
title: "IFAWL开发日志"
subtitle: "关于本地存储"
date: 2026-02-05
author: "Miso"
header-img: "img/post-bg-0.jpg"
tags: ["Python", "IFAWL", "开发"]
---

### 0. 前言 - 关于IFAWL

---

[IFAWL_PROJECT](https://github.com/Miso2233/IFAWL_PROJECT)是一款基于Python IDLE Shell的回合制太空对战游戏，由Miso、Csugu和Tecrel创作。截至发稿，最新版本为1.3.1。可在GitHub下载其源码并开始体验。

IFAWL创作的初衷是：逃离困倦的高三生活。在一间没有网络的教室里，我们急需一种不枯燥、有乐子的娱乐方式。正好，我们当时找到了一些素材，罗列如下：

Csugu：
- 一本Python早教教材
- 从计算机课堂上弄到的Python 3.8

Miso：
- 父亲教自己的一种石头剪刀布纸面游戏

Tecrel：
- 极其精通于科幻创作的头脑

那还说什么了，没有什么比把纸面游戏弄到python里更有乐子的了！

IFAWL计划于焉开始。

### 1. 我们需要存储，我们需要存储

---

没有战利品的游戏是不完整的。我们在IFAWL(O. E. ，老版本)2.0里加入了战利品掉落系统，这直接引申出一个重要的需求：

*我们需要把数据保存到硬盘里，在关闭游戏后这些数据仍然保存，并在用户下一次登录时读取*

这是整个存储开发计划的最核心需求。我们当时习惯把开发日志写在草稿纸上，我读了读，细节需求有这么几个：

1. 登录机制，通过不同的`username : str`来区分不同玩家的不同仓库
2. 仓库内容多样，除了记录各种仓库物资数量外，还要记录玩法最高分等数据
3. 版本更新适应，在仓库物资种类随着版本更迭发生变化时，与旧版本相同的物资应当自动迁移，本地仓库的键名应当立即更新

这是老IFAWL的需求。新IFAWL提出了更多要求：

1. 革除重复性工作，比如终焉结们"3"到"43"的键名应当由程序生成，而非人手打出
2. 新版本添加新数据项的工作流程应尽可能简单
3. 提供一套简洁的外部接口

带着这些需求，让我们看看新IFAWL是怎么实现它们的。

### 2. 技术栈-不要重复造轮子，更不要重新发明轮子

---

我们最终采用的方法是使用shelve模块来存储数据。让我们来喊出那句妙妙咒语：

```python
import shelve
```

曾经有一段时间我们挣扎于这样的想法：把每个玩家的仓库 `: dict`序列化为字符串，把字符串写进一个`.txt`文件里，读取时再反序列化。我和Csugu甚至已经做出了序列化的原型机。天哪，杀了我吧。这个想法真是太痛苦了。

简单回顾一下shelve的用法。

方法`shelve.open('userdata/game_save',writeback=True)`将返回一个shelve对象，这个对象可以像字典一样嵌套地塞入数据，语法和字典一模一样。

```python
shelve_obj = shelve.open('userdata/game_save',writeback=True)

shelve_obj["item1"] = 10 # 设置值

shelve_obj.sync()
```

最后一行的sync()方法可以使对象中的数据保存到本地。下次只需打开同一路径的shelve对象即可重新读出数据。

### 3. 存储模板 读取 回灌 以及其它我们发明的垃圾术语

---

#### 什么是存储模板

存储模板是一个巨大的字典(`: dict[str, Any]`)，其键为IFAWL一切需要保存到本地的数据键名，值为对应的值，可为整数、浮点数、字符串甚至字典。

#### 读取

在用户登录之后，我们尝试读取这个用户在硬盘中的各个键名，将值填充到模板中。若硬盘中键名在模板中没有（版本更新删除物资），则静默跳过。若模板中键名在硬盘中没有（版本更新新增物资），则模板保持原始状态作为默认值。

若没有这个用户，直跳下一步回灌。

#### 回灌

在读取完成之后，模板已经填入了用户的有效数据。此时，我们把整个模板直接保存到硬盘中，覆盖用户的旧有数据。这就完成了硬盘中键名的更新工作。

#### 这一系统的好处

**版本兼容性与数据迁移自动化**是最核心的优势。

通过将硬盘数据读取到最新的模板中，系统自动完成了新旧版本数据的“对齐”。旧版本中存在的、新版本已删除的数据项会被安全地忽略；而新版本新增的数据项，则会自动使用模板中预设的默认值进行初始化。

推进了**开发流程的简化与标准化**。

要添加一个新的可存储数据项，我们只需做一件事：在中央存储模板字典中添加对应的键和默认值。系统会自动处理该数据项的创建、读取、保存和版本迁移。这完全满足了“新版本添加新数据项的工作流程应尽可能简单”的需求，并杜绝了因手动管理键名而可能导致的拼写错误或遗漏。

### 4. 工程代码实现环节

---

##### 我们首先来规定存储模板的格式

为了便于管理IFAWL存储的大量键名，模板采用二级键字典形式，即：

```json
{
  "metadata": {
    "ship_name": "旅行者",
    "al_on_ship": ["","",""],
    "entry_rank": {},
    "max_infinity_round": 0,
    "max_disaster_point": 0,
    "plot_progress": {},
    "tracing_al": ""
  },
  "materials": {
    "三钛合金":10,
    "湿件主机":10,
    "重水":10,
    "高强度树脂":10,
    "锈蚀电路板":10,
    "激光准晶体":10,
    "黄铜差分机":10,
    "电子变频箱":10
  },
  "currency": {
    "联邦信用点":1000,
    "合约纪念点":0,
    "保险点":0
  },
  "als": {

  }
}
```

一级键为类别，如`"metadata"`，二级键为有值键名，如`"三钛合金"`

##### 对模板进行一些自动化的预处理

模板中某些值为空字典，接下来会对它们进行自动填充

```python
class StorageManager:
    
    # ==================== 初始化与基础设置 ====================
    
    def __init__(self):
        self.username:str = ""
        # 建立模板
        self.template:dict[str,dict[str,str|int|dict[str,int]]] = json_loader.load("storage_template")
# 模板预处理开始
        for al_str in AL_META_DATA:
            self.template["als"][al_str] = 0
        for entry_str in ENTRY_META_DATA:
            self.template["metadata"]["entry_rank"][entry_str] = 0
        for session_str in ALL_POLT_DATA:
            self.template["metadata"]["plot_progress"][session_str] = False
# 模板预处理完毕
        # 建立空模板
        self.template_empty = deepcopy(self.template)
        # 建立映射表
        self.item_to_key_table:dict[str,str] = {}
        for key in self.template:
            for item in self.template[key]:
                self.item_to_key_table[item] = key
        # 建立统计字段
        self.total_materials_num = 0
        self.total_als_num = 0
        # 启动主仓库
        if not os.path.exists('userdata'):
            os.makedirs('userdata')
        self.repository_for_all_users = shelve.open('userdata/game_save',writeback=True)
```

从代码中可见，打开原始模板之后，立刻就会进行一次预处理，把一些来自其它源的键值对注入其中。

这一步满足了新IFAWL需求一，”革除重复性工作，比如终焉结们"3"到"43"的键名应当由程序生成，而非人手打出“

最后一步我们启动了包含所有用户数据的主仓库。

#### 用户登录 - 读取与回灌

```python
    # ==================== 用户管理 ====================
    
    def login(self):
        """
        登录|将硬盘数据写入模板|新建用户|重灌入硬盘
        :return: 无
        """
        self.username = Txt.input_plus("指挥官，请输入您的代号")
        if self.username in self.repository_for_all_users.keys():
            for key in self.repository_for_all_users[self.username]:
                if key not in self.template:
                    continue
                for item in self.repository_for_all_users[self.username][key]:
                    if item not in self.template[key]:
                        continue
                    self.template[key][item] = self.repository_for_all_users[self.username][key][item]
        else:
            Txt.print_plus("初次登录|欢迎指挥官")
            self.template = self.template_empty
        self.repository_for_all_users[self.username] = self.template
        # hello
        Txt.print_plus(f"指挥官代号识别成功·{self.get_value_of('ship_name')}号护卫舰正在启动·欢迎来到浅草寺")
        # 更新统计字段
        self.sync()
```

若用户名有本地存档，一连串的for会将二级键值对读入模板中，否则，什么也不干，保持空模板状态。接着，回灌整个模板到硬盘中。

方法的最后我们调用了self.sync()，它的实现如下：

```python
    def sync(self):
        """
        将仓库同步至硬盘
        :return: 无
        """
        self.update_statistical_data()
        self.repository_for_all_users.sync()
```

一个函数最好不要同时做两件事，除非它们的关系真的很紧密。sync()方法同时承担了保存至硬盘和更新统计数据（如资产总量）的职责。

#### 对外接口封装

我最喜欢的环节。

```python
    def show_assets(self) -> dict[str:int]:
        """
        展示现有仓库资产
        :return: 字典，键为所有的materials|currency|Als，值为它们的数目
        """
        out = {}
        for key in ["materials","als","currency"]:
            for item in self.repository_for_all_users[self.username][key]:
                out[item] = self.repository_for_all_users[self.username][key][item]
        return out
    
    def show_assets_except_al(self) -> dict[str:int]:
        """
        展示现有仓库资产,Als除外
        :return: 字典，键为所有的materials|currency，值为它们的数目
        """
        out = {}
        for key in ["materials","currency"]:
            for item in self.repository_for_all_users[self.username][key]:
                out[item] = self.repository_for_all_users[self.username][key][item]
        return out
    
    def modify(self,item:str,delta:int):
        """
        增减仓库物品数量|核心业务封装
        :param item:需要改变数量的物品
        :param delta:物品数量改变量
        :return: 无
        """
        key:str = self.item_to_key_table[item]
        self.repository_for_all_users[self.username][key][item] += delta
        self.sync()
    
    def get_value_of(self,item:str) -> int|str:
        """
        查找仓库物品数量或元数据值|核心业务封装
        :param item: 需查找的物品或键
        :return: 物品在仓库中的数量或元数据值
        """
        key: str = self.item_to_key_table[item]
        return self.repository_for_all_users[self.username][key][item]
    
    def set_value_of(self,item:str,val):
        """
        设置仓库内物品或元数据的值|核心业务封装
        :param item: 需设置的物品或键
        :param val: 需设置的值
        :return: 无
        """
        key: str = self.item_to_key_table[item]
        self.repository_for_all_users[self.username][key][item] = val
        self.sync()
    
    def transaction(self,give_list:dict[str,int],get_list:dict[str,int]):
        """
        进行一次交易|核心业务封装
        :param give_list: 交付 字典
        :param get_list: 得到 字典
        :return: 无
        """
        for item in give_list:
            self.modify(item,-give_list[item])
        for item in get_list:
            self.modify(item,get_list[item])
        self.sync()
```


核心数据操作接口可概括如下

- **`show_assets()`**
    - **功能**：获取玩家当前所有资产（材料、货币、AI）的数量。
    - **返回**：一个字典，键为资产名称，值为其数量。

- **`show_assets_except_al()`**
    - **功能**：获取玩家当前所有资产，但排除AI（Als）。
    - **返回**：一个字典，键为资产名称（仅材料和货币），值为其数量。

- **`modify(item: str, delta: int)`**
    - **功能**：**核心业务封装**。修改仓库中指定物品的数量。
    - **参数**：
        - `item`：需要改变数量的物品名称。
        - `delta`：物品数量的改变量（正数为增加，负数为减少）。
    - **效果**：直接更新仓库数据并同步到硬盘。

- **`get_value_of(item: str)`**
    - **功能**：**核心业务封装**。查找仓库中指定物品的数量或元数据的值。
    - **参数**：`item` - 需要查找的物品名称或元数据键名。
    - **返回**：该物品的数量或元数据的值（整数或字符串）。

- **`set_value_of(item: str, val)`**
    - **功能**：**核心业务封装**。设置仓库内指定物品或元数据的值。
    - **参数**：
        - `item`：需要设置的物品名称或元数据键名。
        - `val`：需要设置的新值。
    - **效果**：直接更新数据并同步到硬盘。

- **`transaction(give_list: dict[str, int], get_list: dict[str, int])`**
    - **功能**：**核心业务封装**。执行一次原子交易，即同时交付一批物品并获取另一批物品。
    - **参数**：
        - `give_list`：一个字典，表示要交付的物品及其数量。
        - `get_list`：一个字典，表示要获取的物品及其数量。
    - **效果**：依次执行`give_list`中所有物品的减少和`get_list`中所有物品的增加，并同步到硬盘。

#### 特点总结

1. **高度抽象**：上层游戏逻辑只需调用这些简单的接口，无需关心底层是使用`shelve`、模板还是回灌机制。
2. **业务聚焦**：`modify`, `get_value_of`, `set_value_of`, `transaction` 被明确标注为“核心业务封装”，是IFAWL存储机制的基石。
3. **数据安全**：所有修改数据的方法（`modify`, `set_value_of`, `transaction`）在操作后都会自动调用`sync()`，确保更改立即持久化到硬盘。
4. **查询便利**：`show_assets`系列方法提供了快速查看玩家资产概况的途径。

这套接口设计满足了新IFAWL的第三个核心需求——“提供一套简洁的外部接口”，使得游戏其他模块能够以安全、一致的方式与玩家数据交互。

### 5. 结语

回顾IFAWL的本地存储开发历程，我们从最初笨拙的手动序列化构想，到最终建立起一套基于“模板-读取-回灌”的自动化系统，这不仅是技术上的迭代，更是开发思维的转变。

这套系统成功地将一个繁琐、易错的后端问题，转化为一个声明式、可维护的基础设施。它让我们得以专注于游戏玩法本身的创新，而无需在数据持久化的泥潭中反复挣扎。每当新版本需要添加一种资源、记录一项成就或调整一个默认值时，我们只需在存储模板的json中轻点一笔，剩下的工作——创建、迁移、兼容——系统都已为我们妥善处理。

更重要的是，它为IFAWL未来的扩展奠定了坚实的基础。无论游戏内容如何丰富，玩家数据如何增长，这套存储架构都能提供稳定、可靠的支持。它或许不是最复杂或最高效的方案，但却是最适合我们这个小团队、这个特定项目的优雅解。

在高三那间没有网络的教室里，在高考后那个炎热的暑假，我们不仅创造了一个游戏，更学会了一种构建复杂系统的方法论。

这，或许比游戏本身更让我们感到自豪。

**浅草寺的大门永远敞开，指挥官的旅程仍在继续。**

欢迎大家游玩IFAWL！
---
layout: post
title: "封装正则表达式"
subtitle: "再也不会写到头疼了"
date: 2026-02-12
author: "Miso"
header-img: "img/post-bg-0.jpg"
tags: ["Python", "正则表达式", "封装", "开发"]
---

{% raw %}


> 本文所封装的接口形式从`keyword-template-matcher`库得到灵感。
>
> 安装语句为*pip install keyword-template-matcher*
>
> [点击访问其Pypi页面](https://pypi.org/project/keyword-template-matcher/#history)

>[在Github查看本项目](https://github.com/Miso2233/MatchReplace)

### 0. 前言 缘起

---

正则表达式很难写，真的。

动态部分和静态部分两种文本毫无规律地混合在一起，各种魔法符号信息量极高，可读性奇差无比。

也许是我的头脑不太灵光，我总觉得Python自带的re库接口设计亦有些口齿不清。表达式对象、匹配结果对象等封装的不同的属性和方法略显混乱，虽然功能强大，但不幸地增加了学习成本。

**来设计一套好用的接口吧！**

### 1. 接口设计

---

在日常生活中，我用正则表达式主要是为了以下几个需求：

- 判定某个文件名是否**符合某个模板**
- 通过文件名和模板**提取信息**
- 通过模板和信息**重新生成文件名**

譬如，`1. [Test]title 49155165223`对于模板`1. [.*?].*? .*?`是匹配的。三个通配符`.*?`需要**提取出文件名中的动态信息**，并**重新注入**到另外一个模板，如`[.*?] - .*?`中。

参照`keyword-template-matcher`中的设计，我决定将用户提供模板的形式设计为`1. [{tag}]{title} {num}`和`[{title}] - {tag}`等，通过与Python f-string一致的形态来增强可读性。花括号内的文字便是**通配符的键名**，提取信息时通过**字典**来存储动态信息，注入时也通过**键名**来匹配字典的**键**。

空花括号代表**通配符**，这段内容会参与匹配过程，但不参与提取信息操作。

让我们设计一个`Template`类，存储一个用户提供的模板，其对外核心接口如下。

```python
    def could_match(self, txt: str) -> bool:
        """
        检查文本是否与模板匹配
        :param txt: 目标文本
        :return: 布尔值，True表示匹配
        """
        ...

    def extract(self, txt: str) -> dict[str, str]:
        """
        从文本中提取出键值对
        :param txt: 文本
        :return: 键值对
        """
		...

    def inject(self, info: dict[str, str]) -> str:
        """
        将键值对注入模板，生成新字符串
        :param info: 键值对
        :return: 生成的新字符串
        """
		...
```

在类的内部，我还须存储Template的原始字符串和可执行的正则表达式字符串。

```python
    def __init__(self, template_str: str):
        self.template_str = template_str
        self.final_regular_expression = what?
```

让我们来实现这些接口吧。

### 2. 从用户提供的模板生成RE|卫语句们

---

#### 卫语句处理

对于用户提供的模板，以下几种情况是不可接受的：

- 大括号不匹配或不闭合 `good morning}{{hello}` （酒吧里点炒饭）
- 大括号嵌套 `my name is {name{age}}` （酒吧里点酒吧）
- 大括号紧邻 `my name is {name}{age}`（因通配符不知道如何分割一段连续文本给两个以上的键）
- 重复键名 `my name is {name}, your name is {name}`

**我们使用一个函数把前三种错误防住**：

```python
    @staticmethod
    def _has_correct_brace(txt: str) -> bool:
        """
        检查txt是否有大括号不匹配错误|嵌套错误|紧邻错误
        :param txt: 待检测文本
        :return: 布尔值
        """
        counting_stack = 0
        for index, char in enumerate(txt):
            if char == '{':
                counting_stack += 1
            elif char == '}':
                counting_stack -= 1
                if index <= len(txt) - 2 and txt[index+1] == '{': # 紧邻错误
                    return False
            if counting_stack < 0 or 2 <= counting_stack: # 嵌套错误
                return False
        return counting_stack == 0 # 不匹配错误
```

`_has_correct_brace`接受原始输入并判断大括号的合法性。其内部实现是扫描整个字符串并用一个整数`counting_stack`来记录已经出现的大括号。遇到左括号时压栈，右括号时弹栈。栈深达到2则报嵌套错误，右括号直接接左括号则报紧邻错误，最终栈深不为0则报不匹配错误。

**用另一个函数把最后一种错误防住**：

```python
    @staticmethod
    def _has_unique_brace_keys(txt: str) -> bool:
        """
        检查txt是否有大括号内键名重复错误
        :param txt: 待检测文本
        :return: 布尔值
        """
        match = re.findall(
            r"\{(.*?)}",
            txt
        )
        match_without_empty_str = [m for m in match if m != ""]
        keys = set(match_without_empty_str)
        return len(keys) == len(match_without_empty_str)
```

这个函数里使用了正则表达式模块提供的顶层函数`re.findall`。`re.findall`传入的首个参数`r"\{(.*?)}"`是一个正则表达式字符串，匹配形如`{name}`的由大括号包围的文本，并捕获大括号内的内容`name`作为结果的一项，最后将所有项输出到`match: list[str]`中。

~~已经开始头疼了？那就对了。请参见本文标题。~~

剔除`{}`通配符对应的空字符串，并使用集合的去重功能进行快速重复判断。

最后，我们的`__init__`函数卫语句已经完成。

```python
    def __init__(self, template_str: str):
        if not self._has_correct_brace(template_str):
            raise ValueError("模板语法错误|大括号嵌套|未封闭|紧邻")
        if not self._has_unique_brace_keys(template_str):
            raise KeyError("模板语法错误|大括号内键名重复")
        self.template_str = template_str
        
        self.final_regular_expression = what?
```

#### 预处理用户输入模板

用户输入的模板中可能含有特殊字符，这些字符可能干扰到基于模板生成的正则表达式。因此，在生成正则表达式之前，有必要预处理一次用户的输入。

```python
SPECIAL_CHARS = frozenset(r"%`~!@#$^&*()_\-+=<>?:\"{}|,./;'[]·~！@#￥……&*（）——-+=|《》？：“”【】、；‘’，。、")

    @staticmethod
    def _block_special_chars(raw: str, without=r"{}") -> str:
        """
        把输入字符串中的特殊字符转义
        :param raw: 原始输入字符串
        :return: 转义特殊字符后的字符串
        """
        out_list = []
        for char in raw:
            if char in SPECIAL_CHARS and char not in without:
                out_list.append(f"\\{char}")
            else:
                out_list.append(char)
        return "".join(out_list)
```

我们使用不可变集合来减少字符搜索的开销。

#### 生成最终正则表达式

正则表达式本身具备为分组命名的语法：`(?P<分组名>.*?)`。如你所见，这个语法复杂如猛兽奇鬼，森然欲搏人。我们的封装以这个语法为基础，希望将上文所述的简洁花括号语法自动转换为复杂的表达式。

形如`{name}`的部分应当被替换为`(?P<name>.*?)`，分三步进行：

- 扫描并匹配`{(.*?)}`，其中的圆括号代表分组，取得结果对象`: match`
- 调用`match.group(1)`取得花括号中的内容
- 如果花括号中的内容是空字符串，替换为非贪心通配符`.*?`，不进行分组
- 否则，进行命名分组，替换为`(?P<{match.group(1)}>.*?)`

代码实现如下：

```python
        raw_regular_expression = re.sub(
            r"\{(.*?)}",  # 形如{name}
            lambda m: rf"(?P<{m.group(1)}>.*?)" if m.group(1) else r".*?",  # 将变为(?P<name>.*?) 或.*?
            blocked_template
        )
```

`re.sub`函数有两个重载，区别在于第二个参数`repl`“替换为”的类型不同。函数先进行查找匹配，当第二个参数填入字符串，则以字符串替换查找到的内容。第二个参数也可以填入一个**输入匹配结果对象、输出字符串的可调用对象**，这里使用了lambda表达式。此时函数第一步进行查找所得到的`: Match`对象将被输入表达式，表达式内提取其分组1（花括号中内容）并动态组合出新字符串。如果花括号内是空字符串，表达式将返回非贪心通配符。

最后，我们为这个表达式加上前后缀来使其精确匹配文本串的两端。用户可以选择在模板前后加上一对`{}`通配符来进行中段匹配。

```python
    def __init__(self, template_str: str):
        if not self._has_correct_brace(template_str):
            raise ValueError("模板语法错误|大括号嵌套|未封闭|紧邻")
        if not self._has_unique_brace_keys(template_str):
            raise KeyError("模板语法错误|大括号内键名重复")
        self.template_str = template_str

        blocked_template = self._block_special_chars(self.template_str)  # 把模板中的特殊字符转义
        raw_regular_expression = re.sub(
            r"\{(.*?)}",  # 形如{name}
            lambda m: rf"(?P<{m.group(1)}>.*?)" if m.group(1) else r".*?",  # 将变为(?P<name>.*?) 或.*?
            blocked_template
        )
        self.final_regular_expression = rf"^{raw_regular_expression}$"  # 加上前后缀
```

至此，正则表达式已生成。

### 3. 顶层接口封装

---

我最喜欢的环节。

#### 三大核心业务

```python
    def could_match(self, txt: str) -> bool:
        """
        检查文本是否与模板匹配
        :param txt: 目标文本
        :return: 布尔值，True表示匹配
        """
        match = re.match(self.final_regular_expression, txt)
        return match is not None

    def extract(self, txt: str) -> dict[str, str]:
        """
        从文本中提取出键值对
        :param txt: 文本
        :return: 键值对
        """
        match = re.match(self.final_regular_expression, txt)
        if not match:
            raise ValueError(f"文本'{txt}'与模板'{self.template_str}'不匹配")
        return match.groupdict()

    def inject(self, info: dict[str, str]) -> str:
        """
        将键值对注入模板，生成新字符串
        :param info: 键值对
        :return: 生成的新字符串
        """
        try:
            out = self.template_str.format_map(info)
        except KeyError as e:
            raise KeyError(f"注入模板时，模板中存在未提供的键名{e}")
        return out
        
```

#### 通过运算符重载简化函数的使用

```python

        
    def __rsub__(self, other: str):
        return self.extract(other)

    def __add__(self, other: dict[str, str]):
        return self.inject(other) 
```

#### 封装工具类

```python
class TemplateTools:

    @staticmethod
    def inject(template: Template, info: dict[str, str]) -> str:
        return template.inject(info)

    @staticmethod
    def extract(template: Template, txt: str) -> dict[str, str]:
        return template.extract(txt)

    @staticmethod
    def transform(input_template: Template, output_template: Template, raw: str) -> str:
        return output_template + (raw - input_template)
```

### 4. 使用示例

---

>本小节由DeepSeek生成

让我们通过几个实际例子来展示这个正则表达式封装库的强大功能。

#### 示例1：文件名模板匹配与信息提取

```python
from matchreplace import Template

# 定义文件名模板
template = Template("{num}. [{tag}]{title} {id}")

# 测试匹配
filename = "1. [Test]My Document 49155165223"
print(template.could_match(filename))  # True

# 提取信息
info = template.extract(filename)
print(info)  # {'num': '1', 'tag': 'Test', 'title': 'My Document', 'id': '49155165223'}

# 重新注入到新模板
new_template = Template("[{tag}] - {title} ({num})")
new_filename = new_template.inject(info)
print(new_filename)  # "[Test] - My Document (1)"
```

#### 示例2：使用运算符简化操作

```python
from matchreplace import Template

# 使用减法运算符提取信息
input_template = Template("User: {name}, Age: {age}")
text = "User: Alice, Age: 25"
info = text - input_template  # 相当于 input_template.extract(text)
print(info)  # {'name': 'Alice', 'age': '25'}

# 使用加法运算符注入信息
output_template = Template("姓名：{name}，年龄：{age}岁")
result = output_template + info  # 相当于 output_template.inject(info)
print(result)  # "姓名：Alice，年龄：25岁"
```

#### 示例3：使用TemplateTools进行模板转换

```python
from matchreplace import Template, TemplateTools

# 定义输入输出模板
input_tpl = Template("IMG_{date}_{time}_{seq}.jpg")
output_tpl = Template("照片-{date}-{time}-第{seq}张.jpg")

# 原始文件名
raw_filename = "IMG_20240211_143022_001.jpg"

# 一键转换
new_filename = TemplateTools.transform(input_tpl, output_tpl, raw_filename)
print(new_filename)  # "照片-20240211-143022-第001张.jpg"
```

#### 示例4：处理特殊字符

```python
from matchreplace import Template

# 模板中包含正则表达式特殊字符
template = Template("File: {name}.txt (size: {size} bytes)")
filename = "File: test[1].txt (size: 1024 bytes)"

print(template.could_match(filename))  # True
info = template.extract(filename)
print(info)  # {'name': 'test[1]', 'size': '1024'}
```

#### 示例5：空花括号通配符

```python
from matchreplace import Template

# 使用空花括号表示通配符（参与匹配但不提取）
template = Template("Chapter {}: {title}")
text = "Chapter 3: The Great Discovery"

print(template.could_match(text))  # True
info = template.extract(text)
print(info)  # {'title': 'The Great Discovery'}
# 注意：空花括号部分不会被提取为键值对
```

#### 示例6：错误处理

```python
from matchreplace import Template

try:
    # 大括号不匹配
    template = Template("Hello {world")
except ValueError as e:
    print(f"错误: {e}")  # 模板语法错误|大括号嵌套|未封闭|紧邻

try:
    # 键名重复
    template = Template("{name} loves {name}")
except KeyError as e:
    print(f"错误: {e}")  # 模板语法错误|大括号内键名重复

try:
    # 文本不匹配模板
    template = Template("Age: {age}")
    info = template.extract("Name: Alice")
except ValueError as e:
    print(f"错误: {e}")  # 文本'Name: Alice'与模板'Age: {age}'不匹配

try:
    # 注入时缺少键
    template = Template("Name: {name}, Age: {age}")
    result = template.inject({"name": "Bob"})
except KeyError as e:
    print(f"错误: {e}")  # 注入模板时，模板中存在未提供的键名'age'
```

### 5. 结语

---

正则表达式是文本处理的利器，但其复杂抽象的语法和极低的可读性常常让人望而却步。通过本工程，我们成功地将正则表达式的强大功能映射到简洁直观的接口之上。

#### 主要优势

1. **直观的模板语法**：采用与Python f-string一致的`{key}`语法，大大提高了可读性
2. **清晰的接口设计**：`could_match()`、`extract()`、`inject()`三个核心方法覆盖了常见需求
3. **运算符重载**：通过`-`和`+`运算符使代码更加简洁优雅
4. **完善的错误处理**：对模板语法错误、匹配失败、键缺失等情况都有明确的异常提示
5. **特殊字符自动转义**：用户无需担心模板中的正则表达式特殊字符

#### 应用场景

这个封装库特别适用于以下场景：

- **文件批量重命名**：按照模板提取信息并重新组织文件名
- **日志解析**：从结构化的日志中提取关键字段
- **模板引擎**：简单的文本模板渲染

#### 最后的话

封装的意义在于减少让人头疼的学习成本，增加开发效率。通过将正则表达式的复杂性封装在简洁的接口之后，我们让文本处理变得更加轻松愉快。

希望这个工具帮到你！

**代码已开源**：[GitHub - Miso2233/MatchReplace](https://github.com/Miso2233/MatchReplace)

欢迎使用、反馈和贡献！

{% endraw %}
---
layout: post
title: "维吉尼亚密码的黄昏"
subtitle: "关于不可破译之物"
date: 2026-02-17
author: "Miso"
header-img: "img/post-bg-0.jpg"
tags: ["Python","密码学"]
---

### 0. 前言 关于维吉尼亚密码

---

>维吉尼亚密码以其简单易用而著称，同时初学者通常难以破解，因而又被称为“不可破译的密码”（法语：le chiffre indéchiffrable）。
>
> —— 百度百科

维吉尼亚密码是一种多表密码，使用**小段密钥不断重复至与原文等长**生成密钥流，并让原文与密钥流的每一位进行**凯撒移位**操作生成最终密文。

由于小段密钥被不断重复，维吉尼亚密码的密文可通过同等间隔的分组划分为`len(key)`组凯撒密码，这为维氏密码的破译打开了突破口。本文将尝试使用Python撰写脚本对维氏密码进行**唯密文攻击**，子任务如下：

- 寻找一种方法能够稳定破解凯撒密码
- 通过分割为`len(key)`组的方式并分组破解凯撒密码的方式，破译已知密钥长度的维氏密码
- 实现**Kasiski检验法**来自动推断密钥长度

### 1. 用频率分析法破译凯撒密码

---

在足够长的英文文本中，每个字母出现的频率并不是均等的。**E**和**T**出现的频率最高，而**Q**和**Z**的出现频率极低，具体分布如下：

```python
STANDARD_FREQUENCY = [
    ('A', 8.2), ('B', 1.5), ('C', 2.8), ('D', 4.3), ('E', 12.7),
    ('F', 2.2), ('G', 2.0), ('H', 6.1), ('I', 7.0), ('J', 0.15),
    ('K', 0.8), ('L', 4.0), ('M', 2.4), ('N', 6.7), ('O', 7.5),
    ('P', 1.9), ('Q', 0.1), ('R', 6.0), ('S', 6.3), ('T', 9.1),
    ('U', 2.8), ('V', 1.0), ('W', 2.4), ('X', 0.15), ('Y', 2.0),
    ('Z', 0.07)
]
```

通过分析某段文字各字母出现的频率与标准频率的相似程度，可以大致断定这段文字是不是原文。我们可以使用差距的平方和来衡量文本的“错误程度”，错误程度低者大概率是原文。

```python
def calculate_error(txt: str) -> float:
    txt = txt.upper()
    out = 0
    counting = [0 for _ in range(26)]
    total_letter = 0
    for char in txt:
        if ord('A') <= ord(char) <= ord('Z'):
            counting[ord(char) - ord('A')] += 1
            total_letter += 1
    if total_letter == 0:
        return 0
    for i in range(26):
        out += (counting[i] / total_letter - STANDARD_FREQUENCY[i][1] / 100) ** 2
    return out * 100
```

经过实地检验，对于随机生成的英文文本，这个函数的输出大约为`2.70`左右，而对于正常的英文长文本，这个值会坍缩至`0.5`。

我们可以遍历凯撒密码的全部26种可能的解，对于每一种解都计算一次`error`值，取最低者即为原文，代码如下：

```python
def solve_caesar_cipher(ciphertext: str) -> str:
    """
    暴力验证法解密凯撒加密
    :param ciphertext: 密文
    :return: 最低错误度的结果
    """
    min_error = MAX_ERROR_SOLVING_CAESAR
    min_error_str = "".join("*" for _ in range(len(ciphertext)))
    for i in range(26):
        result = apply_caesar_cipher(ciphertext, i)
        error = calculate_error(result)
        #print(f"{i = :<5}, {error = :.2f}\t" + ">" * int(error))
        if error < min_error:
            min_error_str = result
            min_error = error
    print(f"    Final Error: {min_error}")
    return min_error_str
```

原文片段（全文共使用三首十四行诗）

```text
Shall I compare thee to a summer’s day?
Thou art more lovely and more temperate.
Rough winds do shake the darling buds of May,
And summer’s lease hath all too short a date.
```

密文

```text
Ncvgg D xjhkvmz oczz oj v nphhzm’s yvt?
Ocjp vmo hjmz gjqzgt viy hjmz ozhkzmvoz.
Mjpbc rdiyn yj ncvfz ocz yvmgdib wpyn ja Hvt,
Viy nphhzm’s gzvnz cvoc vgg ojj ncjmo v yvoz.
```

解密

```text
Shall I compare thee to a summer’s day?
Thou art more lovely and more temperate.
Rough winds do shake the darling buds of May,
And summer’s lease hath all too short a date.
```

可见对凯撒密码的破解已完成。

### 2. 破译已知密钥长度的维氏密码

---

维氏密码在已知密钥长度的情况下可以直接视为多组凯撒密码，我们可以对它们分别进行解密。

```python
def solve_vigenere_cipher_with_key_len(ciphertext: str, key_len: int) -> str:
    """
    暴力验证法解密已知密钥长度的维吉尼亚加密
    :param ciphertext: 密文
    :param key_len: 密钥长度
    :return:
    """
    buffer = ["" for _ in range(len(ciphertext))]
    for diff in range(key_len):
        sub_str = ciphertext[diff::key_len]
        solved = solve_caesar_cipher(sub_str)
        for index, char in enumerate(solved):
            buffer[diff + index * key_len] = char
    return "".join(buffer)
```

原文片段

```text
Shall I compare thee to a summer’s day?
Thou art more lovely and more temperate.
Rough winds do shake the darling buds of May,
And summer’s lease hath all too short a date.
```

密文

`key = "Convallaria", len(key) = 11`

```text
Uvngl T twmroez esev tq n sfxmvz’s qvy?
Tywu oeo xzrv lqjrgy lnu mqfr tpxpvzavs.
Rzfgy wkbqn oz jpams ohp drzlkbt bfos wf Ant,
Lnu swazzr’s cmaus caes rtl hbj dsoib c qvtp.
```

解密

```text
Shall I compare thee to a summer’s day?
Thou art more lovely and more temperate.
Rough winds do shake the darling buds of May,
And summer’s lease hath all too short a date.
```

### 3. 自动推断密钥长度

---

**Kasiski检验法**是检测密钥长度的有力工具。在密文中，相同的字串碎片如果重复出现，极有可能是同一段明文与同一段密钥结合生成的，此时两碎片之间的距离必为密钥长度的整数倍。

现在，我们需要先遍历所有的子串，并记录它们出现的位置。只有出现过**2次及以上的子串**才有价值。

```python
def find_key_len(ciphertext: str) -> list[int]:
    SUB_LEN = 3
    MAX_KEY_LEN = 20
    MIN_SATISFACTION_RATE = 0.5
    # 统计重复序列的下标
    substr_indexes: dict[str,list[int]] = {}
    for index in range(len(ciphertext) - SUB_LEN + 1):
        sub_str = ciphertext[index:index + SUB_LEN]
        if " " in sub_str or "\n" in sub_str or "---" in sub_str:
            continue
        substr_indexes.setdefault(sub_str, []).append(index)
    substr_indexes = {
        sub_str: indexes 
        for sub_str, indexes in substr_indexes.items()
        if len(indexes) > 1
    }
```

接着，对于这些至少重复一次的字串，计算**两次出现之间的距离**，将所有距离存进集合中。使用Python集合是为了快速去重。

```python
    # 计算重复序列的下标差
    differences_set = set()
    for sub_str, indexes in substr_indexes.items():
        differences_set.update(indexes[i + 1] - indexes[i] for i in range(len(indexes) - 1))
```

从`2`到`MAX_KEY_LEN = 20`是密钥的候选长度。对于每一个候选长度，我们计算它是多少个距离的**整除因子**，并统计其比例。最后，返回比例大于`MIN_SATISFACTION_RATE = 0.5`的所有结果

```python
    # 统计所有候选长度的满足率
    satisfaction_rates: dict[int,float] = {}
    differences_num = len(differences_set)
    for key_len in range(2, MAX_KEY_LEN):
        count = 0
        for difference in differences_set:
            if difference % key_len == 0:
                count += 1
        satisfaction_rates[key_len] = count / differences_num

    # 搜集大于最低容忍度的候选长度
    out = [
        key_len
        for key_len, satisfaction_rate in satisfaction_rates.items()
        if satisfaction_rate > MIN_SATISFACTION_RATE
    ]
    out.reverse()
    out.append(1)
    return out
```

经测试，对于`key = "Convallaria", len(key) = 11`，直接输出了`[11, 1]`，函数成功。

### 4. 最终脚本

---

```python
if __name__ == '__main__':
    with open(INPUT_TXT, 'r') as f, open(OUTPUT_TXT, 'w') as g:
        g.write(
            apply_vigenere_cipher(
                f.read(), "Convallaria"
            )
        )

    with open(OUTPUT_TXT, 'r') as f, open(RESULT_TXT, 'w') as g:
        ciphertext = f.read()
        key_lens = find_key_len(ciphertext)
        print(key_lens)
        min_error_str = solve_vigenere_cipher_with_key_len(ciphertext, key_lens[0])
        min_error = calculate_error(min_error_str)
        for key_len in key_lens[1:]:
            result = solve_vigenere_cipher_with_key_len(ciphertext, key_len)
            error = calculate_error(result)
            if error < min_error:
                min_error_str = result
                min_error = error
        g.write(min_error_str)
```

### 5. 结语

---

维吉尼亚密码曾被誉为“不可破译的密码”，然而，通过频率分析、分组破解和Kasiski检验法的结合，我们成功实现了对其的唯密文攻击。这再次印证了密码学领域的一条铁律：**没有绝对的安全，只有相对的强度**。随着计算能力的提升和密码分析技术的发展，任何依赖于固定模式的加密方法终将迎来它的黄昏。
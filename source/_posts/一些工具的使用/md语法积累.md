---
title: md语法积累
categories:
  - [一些工具的使用]
tags:
  - null 
date: 2022/1/5
updated:
comments:
published:
---
# 内容

1. latex语法
2. md语法

# latex语法

## 符号

### 运算符

| 含义                        | 语法  | 效果  |
| --------------------------- | ----- | ----- |
| 大于等于(greater equal, ge) | `\ge` | $\ge$ |
| 小于等于(less equal, le)    | `\le` | $\le$ |
| 不等于(not equal, ne)       | `\ne` | $\ne$ |

| 含义            | 语法                | 效果                |
| --------------- | ------------------- | ------------------- |
| 向上取整(ceil)  | `\lceil x \rceil`   | $\lceil x \rceil$   |
| 向下取整(floor) | `\lfloor x \rfloor` | $\lfloor x \rfloor$ |

### 箭头

更多箭头参考：https://www.sascha-frank.com/Arrow/latex-arrows.html

| 含义   | 语法                                                       | 效果                                                       |
| ------ | ---------------------------------------------------------- | ---------------------------------------------------------- |
| 等效于 | `\Leftrightarrow`、`\leftrightarrow`、`\rightleftharpoons` | $\Leftrightarrow$、$\leftrightarrow$、$\rightleftharpoons$ |
| 可推出 | ` \Rightarrow`、`\rightarrow`、`\longrightarrow`           | $ \Rightarrow$、$\rightarrow$、$\longrightarrow$           |

### 波浪线

由于latex中`~`符号有特殊用途（强制空格）， 因此需要用其他方式才能打出。

方式1：一般文字环境下，`\textasciitilde`。效果：$\textasciitilde$

方式2：公式环境下，`\sim`。效果：$\sim$

区别：用公式环境下打出，波浪线会大一些。

## 对数

1. `\log_2 ...` => $\log_2(n+1)$
2. `\ln ...` => $\ln(n+1)$
3. `\lg ...` => $\lg(n+1)$

## 多行公式

### 左对齐

```
证明：设完全二叉树的高度为$h$，则有
$$
\begin{flalign}
&~~~~~~2^h-1<n\le2^{h+1}-1\\
&\Leftrightarrow 2^h<n+1\le2^{h+1}\\
取对数&\Leftrightarrow h<\log_2(n+1)\le h+1&
\end{flalign}
$$
```

#### 要点

1. 开头独行，`\begin{flalign}`；结尾独行，`\end{flalign}`
2. **公式换行符：`\\`**

#### 左对齐效果

证明：设完全二叉树的高度为$h$，则有
$$
\begin{flalign}
&~~~~~~2^h-1<n\le2^{h+1}-1\\
&\Leftrightarrow 2^h<n+1\le2^{h+1}\\
取对数&\Leftrightarrow h<\log_2(n+1)\le h+1&
\end{flalign}
$$
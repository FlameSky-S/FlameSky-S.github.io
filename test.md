# 拼音输入法实验

## 代码功能介绍

### 1. 代码实现的功能

1. 基于“字的n-gram模型(n=2,3,4)带平滑处理”的拼音转文字
2. 爬取最新sina news并生成测试数据
3. 对比模型输出结果和答案，给出正确率
4. 基于词的n-gram模型(n=2)的拼音转文字

### 2. 文件目录结构说明

```text
$root_dir/
├── data/
│   ├── characters/     # 汉字和拼音字典
│   ├── test_data/      # 测试数据
│   │   ├── news.txt    # 测试数据原稿，抓自sina news
│   │   ├── answer.txt  # 测试数据规范化，用作答案
│   │   └── pinyin.txt  # 测试数据拼音，符合作业输入文件要求
│   └── train_data/     # 训练数据，来自作业提供的sina-news-gbk
├── image/              # 实验报告图库
├── model/              # 存放训练好的模型
├── src/                # 源代码
│   ├── main.py         # 参数读取，程序控制
│   ├── train.py        # 模型训练
│   ├── predict.py      # 模型预测
│   ├── spider.py       # 爬虫，爬取最新sina news国际和国内板块
│   └── data_pre.py     # 测试数据预处理
└── README.md           # 实验报告
```

### 3. 参数说明

```text
❯ python src/main.py --help
usage: main.py [-h] [-n N_GRAM] [-i INPUT] [-o OUTPUT] [-a ANSWER] [-Q] [-R] [-S]

optional arguments:
  -h, --help            show this help message and exit
  -n N_GRAM, --n-gram N_GRAM
                        use n-gram model, n between 2 and 4, default: 2
  -i INPUT, --input INPUT
                        full path of input file
  -o OUTPUT, --output OUTPUT
                        full path of output file
  -a ANSWER, --answer ANSWER
                        full path of answer file
  -Q, --query           query mode for displaying records within models
  -R, --retrain         force retrain model
  -S, --spider          recrawl test data using spider
```

* `-n` 指定n-gram模型，$2 \le n \le 4$，默认值：2

* `-i` 指定输入文件，未指定则进入命令行测试模式

* `-o` 指定输出文件，默认值："$root_dir/data/test_data/output.txt"

* `-a` 指定比对文件，未指定则不进行正确率计算

* `-Q` 查询模式，查询模型字典中的键值

* `-R` 强制重新训练模型

* `-S` 重新爬取测试数据，会覆盖旧的测试数据，慎用

### 4. 复现方法

1. 将sina_news_gbk文件夹下的数据文件放入`train_data`文件夹，***注意***不能包含README文件
2. 命令行运行python ./src/main.py -n 4 -i "./data/dev_data/pinyin.txt" -a "./data/dev_data/answer.txt"
3. 需要python版本大于3.8

### 5. 实验结果

二元模型准确率达到87%

---

## 算法分析

### 1. 字的二元模型

* **概率计算**

根据条件概率公式，字符串$w_1w_2w_3 \ldots w_n$出现的概率为：
$$P(w_1w_2w_3 \ldots w_n) = P(w_1) \cdot P(w_2|w_1) \cdot P(w_3|w_1,w_2) \cdots P(w_n|w_1,w_2 \ldots ,w_{n-1})$$
根据马尔科夫假设，一个字出现的概率仅与它前面的N个字有关,以这样的假设为前提的模型就是N-gram语言模型。
在中文里，以字为基本单元的N-gram模型叫做字的N-gram模型，以词为基本单元的N-gram模型叫做词的N-gram模型。
本节讨论以字为基本单元，$N=2$的模型，即“字的二元模型”

在字的二元模型中，$w_i$代表一个汉字，则句子$w_1w_2w_3 \ldots w_n$出现的概率为：
$$P(w_1w_2 \ldots w_n) = P(w_1|S_{start}) \cdot P(w_2|w_1) \cdots P(w_n|w_{n-1}) \cdot P(S_{end}|w_n) = P(w_1|S_{start}) \cdot \prod_{i=2}^n {P(w_i|w_{i-1})} \cdot P(S_{end}|w_n) \tag{1}$$
其中，$S_{start}$和$S_{end}$分别代表句子的开头和结尾。
>注：这里相比PPT中给出的公式多了对句首句尾的处理，更加的make sense，从结果来看效果也更好

假设语料库的总字数为$M$，$w_{i-1}$和$w_i$同时出现的次数为$C(w_i|w_{i-1})$，$w_i$出现的次数为$C(w_i)$，那么有:
$$P(w_i|w_{i-1}) = \frac {P(w_{i-1},w_i)}{P(w_{i-1})} = \cfrac {\cfrac {C(w_{i-1},w_i)}{M}}{\cfrac {C(w_{i-1})}{M}} = \frac {C(w_{i-1},w_i)}{C(w_{i-1})} \tag{2}$$

所以训练模型时只需统计单个字以及连续两个字出现的次数即可。

* **平滑处理**

使用模型进行预测时，根据(1)(2)式进行计算，会遇到问题：当$w_{i-1},w_i$在语料库中没有出现过时，即$C(w_i|w_{i-1}) = 0$，导致连乘后整个句子的概率都为0，这显然是不正确的。为解决该问题，我们可以简单地将语料库中不存在的序列的出现概率设定为一个很小的值，比如1e-6（数学上不严谨），也可以使用平滑算法。常见的平滑算法有：加一(Add-one smoothing)，插值(Model interpolation)，Good-Turing等。这里使用插值算法（加一可以看做插值的[一种](https://medium.com/mti-technology/n-gram-language-model-b7c2fc322799 "详细解释")），其中$\lambda$为权重：
$$P_S(w_i|w_{i-1}) = \lambda \cdot P(w_i|w_{i-1}) + (1-\lambda) \cdot P(w_i) \tag{3}$$

结合(1)(2)(3)式，得到最终计算公式
$$P(w_1w_2 \ldots w_n) = \left(\lambda\frac{C(S_{start},w_1)}{C(S_{start})} + (1-\lambda)\frac{C(w_1)}{M}\right) \cdot \left(\lambda\frac {C(w_n,S_{end})}{C(w_n)} + (1-\lambda)\frac{C(S_{end})}{M}\right) \cdot \prod_{i=2}^n \left(\lambda\frac{C(w_{i-1},w_i)}{C(w_{i-1})} + (1-\lambda)\frac{C(w_i)}{M}\right) \tag{4}$$

在实际测试后发现，插值算法对于字的二元模型的正确率提升非常有限,仅为0.13个百分点，而时间消耗增加了约50%：

![字的二元模型插值权重$\lambda$对准确率的影响](image/lambda-accuracy.png "λ-准确率")

* **代码实现**

在代码实现过程中，需要统计单个字以及连续两个字的序列出现的次数。最直观的想法是建立两个字典，一个统计单字出现次数：`model[char]`；一个统计序列出现次数：`model[prev_char][char]`。但是这样不仅增加了模型的体积，也使得模型加载更为复杂。考虑将两个字典合并的方法：在统计序列的字典中加入一个字符串`'total'`作为key，即`model[char]['total']`，对应的value即为`char`出现的总次数。对于语料库的总字数$M$，在模型中添加一个键`model['all']['total']`，将每个字符的`'total'`值加和即可。

代码中对句首句尾的处理做了简化。
首先考虑在句首句尾添加标记字符，例如句首加`'#'`,句尾加`'$'`。那么通过公式(4)可以看出，需要统计以`'#'`为前缀和以`'$'`为后缀的序列出现的次数，以及`'#'`、`'$'`出现的总次数。由于`'#'`前缀和`'$'`后缀互不影响，且`'#'`和`'$'`出现的总次数相等，方便起见，将句首尾标记统一为`'#'`。那么需要额外统计的有：`model['#'][char]`、`model[char]['#']`和`model['#']['total']`。

在实现过程中，首先对训练数据进行预处理，将非中文字符替换为换行符，然后逐行进行统计。
最终统计部分代码如下：（省略实现细节）

```python
  # count occurences of characters
  for line in f:
      Prev_Char = '#'
      for c in line:
          if c == '\n':
              hanzi_dict_2[Prev_Char]['#'] += 1
              hanzi_dict_2['#']['total'] += 1
          else:
              hanzi_dict_2[c]['total'] += 1
              hanzi_dict_2[Prev_Char][c] += 1
              Prev_Char = c
```

预测部分代码如下：（省略实现细节）

```python
  def viterbi_2(pinyin, pinyin_dict, hanzi_dict, alpha):
      """
      Viterbi algorithm for bigram models
      args:
          pinyin: a string of pinyin that needs to be converted
          pinyin_dict: pinyin to chinese dictionary
          hanzi_dict: bigram model for chinese characters
          Lambda: weight of the bigram model in smoothing
      """
      node_list_old = [('#', 1)]    # previous step
      for i in range(1, len(pinyin_list)):
          node_list_new = []    # this step
          for c in pinyin_dict[pinyin_list[i]]:
              best_p = ('', 0)    # store best path
              for prev_node in node_list_old:
                  calculateP()
                  if p > best_p[1]:
                      best_p = (prev_node[0] + c, p)
              node_list_new.append(best_p)
          node_list_old = node_list_new   # discard previous step, only store 1 step
      return node_list_old[0][0]
```

### 2. 字的三元模型

* **概率计算**

继续前面的分析，在二元模型的基础上上升到三元模型，仍然使用Interpolation，公式如下：
$$P_S(w_3|w_1,w_2) = \lambda_1 \cdot P_1(w_3) + \lambda_2 \cdot P_2(w_3|w_2) + \lambda_3 \cdot P_3(w_3|w_1,w_2) \tag{5}$$
将(2)式代入，得：
$$P_S(w_3|w_1,w_2) = \lambda_1 \frac {C(w_3)}{M} + \lambda_2 \frac {C(w_2,w_3)}{C(w_2)} + \lambda_3 \frac {C(w_1,w_2,w_3)}{C(w_1,w_2)}$$

由于三元模型需要前两个字的信息，所以在句首尾的处理上与二元模型有所不同。首先考虑句首，添加两个句首标记`'#'`，

---
title: Elasticsearch 分析器
date: 2022-02-22 21:01:01
order: 09
categories:
  - 数据库
  - 搜索引擎数据库
  - Elasticsearch
tags:
  - 数据库
  - 搜索引擎数据库
  - Elasticsearch
  - 分词
permalink: /pages/6bfb0fbf/
---

# Elasticsearch 分析器

文本分析是把全文本转换为一系列单词（term/token）的过程，也叫分词。在 Elasticsearch 中，分词是通过 analyzer（分析器）来完成的，不管是索引还是搜索，都需要使用 analyzer（分析器）。分析器，分为**内置分析器**和**自定义的分析器**。

分析器可以进一步细分为**字符过滤器**（**Character Filters**）、**分词器**（**Tokenizer**）和**词元过滤器**（**Token Filters**）三部分。它的执行顺序如下：

**_character filters_** -> **_tokenizer_** -> **_token filters_**

## 字符过滤器（Character Filters）

character filter 的输入是原始的文本 text，如果配置了多个，它会按照配置的顺序执行，目前 ES 自带的 character filter 主要有如下 3 类：

1. **html strip character filter**：从文本中剥离 HTML 元素，并用其解码值替换 HTML 实体（如，将 **_`＆amp;`_** 替换为 **_`＆`_**）。
2. **mapping character filter**：自定义一个 map 映射，可以进行一些自定义的替换，如常用的大写变小写也可以在该环节设置。
3. **pattern replace character filter**：使用 java 正则表达式来匹配应替换为指定替换字符串的字符，此外，替换字符串可以引用正则表达式中的捕获组。

### HTML strip character filter

HTML strip 如下示例：

```bash
GET /_analyze
{
  "tokenizer": "keyword",
  "char_filter": [
    "html_strip"
  ],
  "text": "<p>I&apos;m so <b>happy</b>!</p>"
}
```

经过 **_`html_strip`_** 字符过滤器处理后，输出如下：

```
[ \nI'm so happy!\n ]
```

### Mapping character filter

Mapping character filter 接收键和值映射（key => value）作为配置参数，每当在预处理过程中遇到与键值映射中的键相同的字符串时，就会使用该键对应的值去替换它。

原始文本中的字符串和键值映射中的键的匹配是贪心的，在对给定的文本进行预处理过程中如果配置的键值映射存在包含关系，会优先**匹配最长键**。同样也可以用空字符串进行替换。

mapping char_filter 不像 html_strip 那样拆箱即可用，必须先进行配置才能使用，它有两个属性可以配置：

| 参数名称              | 参数说明                                                                                       |
| :-------------------- | :--------------------------------------------------------------------------------------------- |
| **_`mappings`_**      | 一组映射，每个元素的格式为 _key => value_。                                                    |
| **_`mappings_path`_** | 一个相对或者绝对的文件路径，指向一个每行包含一个 _key =>value_ 映射的 UTF-8 编码文本映射文件。 |

mapping char_filter 示例如下：

```bash
GET /_analyze
{
  "tokenizer": "keyword",
  "char_filter": [
    {
      "type": "mapping",
      "mappings": [
        "٠ => 0",
        "١ => 1",
        "٢ => 2",
        "٣ => 3",
        "٤ => 4",
        "٥ => 5",
        "٦ => 6",
        "٧ => 7",
        "٨ => 8",
        "٩ => 9"
      ]
    }
  ],
  "text": "My license plate is ٢٥٠١٥"
}
```

分析结果如下：

```
[ My license plate is 25015 ]
```

### Pattern Replace character filter

Pattern Replace character filter 支持如下三个参数：

| 参数名称            | 参数说明                                                                       |
| :------------------ | :----------------------------------------------------------------------------- |
| **_`pattern`_**     | 必填参数，一个 java 的正则表达式。                                             |
| **_`replacement`_** | 替换字符串，可以使用 **_`$1 ... $9`_** 语法来引用捕获组。                      |
| **_`flags`_**       | Java 正则表达式的标志，具体参考 java 的 java.util.regex.Pattern 类的标志属性。 |

如将输入的 text 中大于一个的空格都转变为一个空格，在 settings 时，配置示例如下：

```bash
"char_filter": {
  "multi_space_2_one": {
    "pattern": "[ ]+",
    "type": "pattern_replace",
    "replacement": " "
  },
  ...
}
```

## 分词器（Tokenizer）

tokenizer 即分词器，也是 analyzer 最重要的组件，它对文本进行分词；**一个 analyzer 必需且只可包含一个 tokenizer**。

ES 自带默认的分词器是 standard tokenizer，标准分词器提供基于语法的分词（基于 Unicode 文本分割算法），并且适用于大多数语言。

此外有很多第三方的分词插件，如中文分词界最经典的 ik 分词器，它对应的 tokenizer 分为 ik_smart 和 ik_max_word，一个是智能分词（针对搜索侧），一个是全切分词（针对索引侧）。

ES 默认提供的分词器 standard 对中文分词不优化，效果差，一般会安装第三方中文分词插件，通常首先 [elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik) 插件，它其实是 ik 针对的 ES 的定制版。

### elasticsearch-plugin 使用

在安装 elasticsearch-analysis-ik 第三方之前，我们首先要了解 es 的插件管理工具 **_`elasticsearch-plugin`_** 的使用。

现在的 elasticsearch 安装完后，在安装目录的 bin 目录下会存在 elasticsearch-plugin 命令工具，用它来对 es 插件进行管理。

```
bin/elasticsearch-plugin
```

其实该命令的是软连接，原始路径是：

```
libexec/bin/elasticsearch-plugin
```

再进一步看脚本代码，你会发现，它是通过 **_`elasticsearch-cli`_** 执行 `libexec/lib/tools/plugin-cli/elasticsearch-plugin-cli-x.x.x.jar`。

但一般使用者了解 elasticsearch-plugin 命令使用就可：

```bash
#  安装指定的插件到当前 ES 节点中
elasticsearch-plugin install {plugin_url}

#  显示当前 ES 节点已经安装的插件列表
elasticsearch-plugin list

#  删除已安装的插件
elasticsearch-plugin remove {plugin_name}
```

> 在安装插件时，要保证安装的插件与 ES 版本一致。

### elasticsearch-analysis-ik 安装

在确定要安装的 ik 版本之后，执行如下命令：

```bash
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v{X.X.X}/elasticsearch-analysis-ik-{X.X.X}.zip
```

执行完安装命令后，我们会发现在 plugins 中多了 analysis-ik 目录，这里面主要存放的是源码 jar 包，此外，在 config 文件里也多了 analysis-ik 目录，里面主要是 ik 相关的配置，如 IKAnalyzer.cfg.xml 配置、词典文件等。

```bash
#  两个新增目录路径
libexec/plugins/analysis-ik/
libexec/config/analysis-ik/
```

### elasticsearch-analysis-ik 使用

ES 5.X 版本开始安装完的 elasticsearch-analysis-ik 提供了两个分词器，分别对应名称是 **_ik_max_word_** 和 **_ik_smart_**，ik_max_word 是索引侧的分词器，走全切模式，ik_smart 是搜索侧的分词器，走智能分词，属于搜索模式。

#### 索引 mapping 设置

安装完 elasticsearch-analysis-ik 后，我们可以指定索引及指定字段设置可用的分析器（analyzer），示例如下：

```json
{
  "qa": {
    "mappings": {
      "qa": {
        "_all": {
          "enabled": false
        },
        "properties": {
          "question": {
            "type": "text",
            "store": true,
            "similarity": "BM25",
            "analyzer": "ik_max_word",
            "search_analyzer": "ik_smart"
          },
          "answer": {
            "type": "text",
            "store": false,
            "similarity": "BM25",
            "analyzer": "ik_max_word",
            "search_analyzer": "ik_smart"
          },
          ...
        }
      }
    }
  }
}
```

如上示例中，analyzer 指定 ik_max_word，即索引侧使用 ik 全切模式，search_analyzer 设置 ik_smart，即搜索侧使用 ik 智能分词模式。

#### 查看 ik 分词结果

es 提供了查看分词结果的 api **`analyze`**，具体示例如下：

```bash
GET {index}/_analyze
{
  "analyzer" : "ik_smart",
  "text" : "es 中文分词器安装"
}
```

输出如下：

```json
{
  "tokens": [
    {
      "token": "es",
      "start_offset": 0,
      "end_offset": 2,
      "type": "CN_WORD",
      "position": 0
    },
    {
      "token": "中文",
      "start_offset": 3,
      "end_offset": 5,
      "type": "CN_WORD",
      "position": 1
    },
    {
      "token": "分词器",
      "start_offset": 5,
      "end_offset": 8,
      "type": "CN_WORD",
      "position": 2
    },
    {
      "token": "安装",
      "start_offset": 8,
      "end_offset": 10,
      "type": "CN_WORD",
      "position": 3
    }
  ]
}
```

#### elasticsearch-analysis-ik 自定义词典

elasticsearch-analysis-ik 本质是 ik 分词器，使用者根据实际需求可以扩展自定义的词典，具体主要分为如下 2 大类，每类又分为本地配置和远程配置 2 种：

1. 自定义扩展词典；
2. 自定义扩展停用词典；

elasticsearch-analysis-ik 配置文件为 `IKAnalyzer.cfg.xml`，它位于 `libexec/config/analysis-ik` 目录下，具体配置结构如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict"></entry>
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords"></entry>
	<!--用户可以在这里配置远程扩展字典 -->
	<!-- <entry key="remote_ext_dict">words_location</entry> -->
	<!--用户可以在这里配置远程扩展停止词字典-->
	<!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>
```

> 当然，如果开发者认为 ik 默认的词表有问题，也可以进行调整，文件都在 `libexec/config/analysis-ik` 下，如 main.dic 为主词典，stopword.dic 为停用词表。

## 词元过滤器（Token Filters）

token filters 叫词元过滤器，或词项过滤器，对 tokenizer 分出的词进行过滤处理。常用的有转小写、停用词处理、同义词处理等等。**一个 analyzer 可包含 0 个或多个词项过滤器，按配置顺序进行过滤**。

以同义词过滤器的使用示例，具体如下：

```bash
PUT /test_index
{
  "settings": {
    "index": {
      "analysis": {
        "analyzer": {
          "synonym": {
            "tokenizer": "standard",
            "filter": [ "my_stop", "synonym" ]
          }
        },
        "filter": {
          "my_stop": {
            "type": "stop",
            "stopwords": [ "bar" ]
          },
          "synonym": {
            "type": "synonym",
            "lenient": true,
            "synonyms": [ "foo, bar => baz" ]
          }
        }
      }
    }
  }
}
```

### 同义词

Elasticsearch 同义词通过专有的同义词过滤器（synonym token filter）来进行工作，它允许在分析（analysis）过程中方便地处理同义词，一般是通过配置文件配置同义词。此外，同义词可以再建索引时（index-time synonyms）或者检索时（search-time synonyms）使用。

#### 同义词（synonym）配置语法

如上例子所示，es 同义词配置的 filter 语法具体如下选项：

- **_`type`_**：指定 synonym，表示同义词 filter；

- **_`synonyms_path`_**：指定同义词配置文件路径；

- **`expand`**：该参数决定映射行为的模式，默认为 true，表示扩展模式，具体示例如下：

  - 当 **`expand == true`** 时，

    ```
    ipod, i-pod, i pod
    ```

    等价于：

    ```
    ipod, i-pod, i pod => ipod, i-pod, i pod
    ```

    当 **_`expand == false`_** 时，

    ```
    ipod, i-pod, i pod
    ```

    仅映射第一个单词，等价于：

    ```
    ipod, i-pod, i pod => ipod
    ```

- **_`lenient`_**：如果值为 true 时，遇到那些无法解析的同义词规则时，忽略异常。默认为 false。

#### 同义词文档格式

elasticsearch 的同义词有如下两种形式：

- 单向同义词：

  ```
  ipod, i-pod, i pod => ipod
  ```

- 双向同义词：

  ```
  马铃薯, 土豆, potato
  ```

单向同义词不管索引还是检索时，箭头左侧的词都会映射成箭头右侧的词；

双向同义词是索引时，都建立同义词的倒排索引，检索时，同义词之间都会进行倒排索引的匹配。

> 同义词的文档化时，需要注意的是，同一个词在不同的同义词关系中出现时，其它同义词之间不具有传递性，这点需要注意。

假设如上示例中，如果“马铃薯”和其它两个同义词分成两行写：

```
马铃薯,土豆
马铃薯,potato
```

此时，elasticsearch 中不会将“土豆”和“potato”视为同义词关系，所以多个同义词要写在一起，这往往是开发中经常容易疏忽的点。

## 参考资料

- [Elasticsearch 教程](https://www.knowledgedict.com/tutorial/elasticsearch-intro.html)
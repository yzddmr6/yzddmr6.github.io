# 大模型应用之RAG技术学习


<meta name="referrer" content="no-referrer" />



## 什么是RAG
检索增强生成（Retrieval Augmented Generation, RAG）是一种技术，它通过从数据源中检索信息来辅助大语言模型（Large Language Model, LLM）生成答案。

简而言之，RAG 结合了搜索技术和大语言模型的提示词功能，即向模型提出问题，并以搜索算法找到的信息作为背景上下文，这些查询和检索到的上下文信息都会被整合进发送给大语言模型的提示中。

### 为什么会出现RAG
思考一个问题：为什么会出现RAG这一项技术？

一般来说，想要给大模型输入指定的知识有两种办法：一种是在prompt时加入上下文，另一种是通过fine-tuning。

如果知识较少且较为固定的时候还好，但是当知识库非常大且需要频繁迭代的时候，这两种办法的局限性就更加突出了：

1. 如果通过prompt，大模型是有token数量限制的。面对庞大的知识库，大模型不可能一下子读取所有内容。
2. 如果通过微调，加一点知识就要微调一次，各方面成本很高。

因此RAG技术应运而生。

RAG是怎么解决这个问题的呢？

简单来说，就是“外挂”一个向量数据库，每次查询之前在知识库中进行一次相似度检索，将较为精确的知识片段截取出来后，再拼接到prompt的上下文里作为背景知识。这样就同时解决了大模型token上限，以及知识库频繁迭代的问题。同时，通过prompt约束，还可以防止大模型幻觉的产生。

一个经典的图如下：

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729581037819-dba6c98b-8b3f-483e-b478-1a97f3137341.png)

通常流程是：

文档向量化过程：文档->分词->embedding->向量数据库

用户查询过程：用户query->向量数据库查询->TOP N->上下文+ 用户提问 + prompt -> LLM -> 返回结果



这里涉及到几个关键名词：embedding、向量数据库、Retrieval

### embedding
embedding是将现实中的物体通过向量化的方法转化为高维向量，可被机器学习模型所识别。他是一种映射，同时也保证了能清晰地表达现实物体的特征。基于此，可以进行一些归类分析、回归分析等。

### 向量数据库
向量数据库底层存储的是一堆向量，它提供了根据向量相似度进行查询的能力，一般情况下，向量相似度代表了现实世界中物体的相似度。比如”我的名字是小明“ 和“我叫小明”这两句话所代表的含义几乎是相同的，那么在embedding之后，基于向量数据库进行查询的时候，它们俩的相似度就会很近

### Retrieval
文档检索（Retrieval）是信息检索领域中的一个关键任务，旨在从大量文档中找到与查询最相关的文档。根据具体使用的技术和方法，检索模式（Retrieval Mode）通常可以分为以下几种：

1. Embedding Only

Embedding Only 模式使用向量嵌入（vector embeddings）来表示文档和查询。向量嵌入是一种将高维数据映射到低维连续向量空间的技术，以便进行更高效的计算和比较。常见的Embedding技术包括Word2Vec、GloVe、BERT等。

+ 工作方式: 将查询和文档都转换成向量嵌入，通过计算它们之间的距离（如余弦相似度）来进行匹配。
+ 优点: 能捕捉到单词和短语之间的语义相似度，处理同义词和多义词效果较好。
+ 缺点: 对于未在训练集中见过的新单词可能表现较差，且计算嵌入向量可能需要较高的计算资源。
2. Keyword Only

Keyword Only 模式基于关键词匹配来进行文档检索。这种方法通常依赖于布尔检索模型或TF-IDF（Term Frequency-Inverse Document Frequency）等统计方法。

+ 工作方式: 将查询和文档转换为关键词集合，基于这些关键词进行匹配和排序。
+ 优点: 实现简单，计算效率高，适用于精确匹配的场景。
+ 缺点: 无法处理同义词和语义相似度，可能会错过语义相关但不包含指定关键词的文档。
3. Hybrid

Hybrid 模式结合了Embedding Only和Keyword Only两种方法，以便在检索过程中利用双方的优势。

+ 工作方式: 先使用Keyword Only方法进行初步过滤，然后再用Embedding Only方法进行精细排序，或者同时使用两种方法并融合它们的结果。
+ 优点: 在保证计算效率的同时，提高了语义理解的能力，能够更好地处理复杂查询。
+ 缺点: 实现复杂度较高，可能需要更多的计算资源和时间。

## 搭建过程
这里采用了手工编写代码搭建，以及使用阿里云人工智能PAI平台搭建两种方式。

### 自动搭建
商业化带来便利，提供婴儿级的一键式搭建服务。

阿里云的PAI平台支持一键部署大模型RAG对话系统，一行代码都不用写。

官方文档：[https://help.aliyun.com/zh/pai/user-guide/deploy-a-rag-based-dialogue-system#2d416a8753chr](https://help.aliyun.com/zh/pai/user-guide/deploy-a-rag-based-dialogue-system#2d416a8753chr)

首先选择要使用哪个大模型。由于只是测试，选一个7b的Qwen2就够了。越大的模型需要越大的GPU，成本++

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729583956956-0e90a4ca-a83d-425f-9307-17f5544a1c89.png)

另外还有一个向量数据库的选择，其他类型数据库都需要单独开通服务。FAISS可以直接存储在内存或者磁盘上，对我们测试来说基本够用了，比较方便。

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729584021573-b0bc6bc9-799b-4910-a9db-65b51d504e88.png)

点击部署之后等一会就可以看到Web控制台了

Setting里基本不用修改已经配好，我们只需要上传我们的知识库文件。

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729663675459-73ce8e2e-f12d-4b77-a42a-acdcfb9bd364.png)

在这里我搜集了一些阿里云云安全中心的公开文档，以及加了一条我个人的说明。

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729841922790-960933f7-3cbb-4e11-ab01-43eff6cdeaa1.png)

传完之后，会进行向量化的操作，接着就可以开始提问了。



问一些数据库中有的知识

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729663729033-21a56b59-0da5-4bf9-9832-63f7e59ea309.png)

问一些没有的知识，会进行拒答。

通过左边的prompt模板，可以设置拒答的内容

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729663739747-dc8e6bbc-87bb-48d2-8047-bbeceeb1dfac.png)



左边的选项，还可以仅开启数据库检索，或者仅开启大模型功能

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729664027028-e49d053f-2605-4ac3-9dab-9d076b0dccc1.png)



选择仅调用大模型的时候就不再受知识库的约束了，这里回答的是模型自身拥有的知识。

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729663940276-f58c6e73-d6e2-470e-a5e9-0dd147e94157.png)

### 手动搭建
由于langchain这个工具库的存在，我们可以很方便地通过代码手动搭建RAG。

直接搜索出来的demo样例，都是基于openai的。但是众所周知，由于一些神秘魔法的存在，国内无法直接访问chatgpt跟huggingface，需要做本土化兼容。



![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729748932315-e0443b0a-d057-492e-83fb-c12fa9df0401.png)



因此我用Tongyi来代替Openai，另外给huggingface设置镜像。注意`os.environ`得在import huggingface库相关语句之前执行。

```python
import os
os.environ['HF_ENDPOINT'] = 'https://hf-mirror.com'
```



完整代码如下：

```java
import os
os.environ['HF_ENDPOINT'] = 'https://hf-mirror.com'
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.text_splitter import CharacterTextSplitter
from langchain.vectorstores import FAISS
from langchain.document_loaders import TextLoader
from langchain.chains import RetrievalQA
from langchain.llms import Tongyi

# 设置环境变量
os.environ["DASHSCOPE_API_KEY"] = "xxxxx"

# 初始化文本嵌入模型
embeddings = HuggingFaceEmbeddings(model_name="sentence-transformers/paraphrase-multilingual-mpnet-base-v2")

# 加载文档
loader = TextLoader("知识库路径")
documents = loader.load()

# 分割文本
text_splitter = CharacterTextSplitter(chunk_size=500, chunk_overlap=0)
texts = text_splitter.split_documents(documents)

# 创建向量存储
vectorstore = FAISS.from_documents(texts, embeddings)

# 初始化通义千问模型
llm = Tongyi(model="qwen-turbo")

# 创建检索问答链
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=vectorstore.as_retriever(),
    return_source_documents=True
)

# 使用函数进行问答
def ask_question(question):
    result = qa_chain({"query": question})
    return result["result"], result["source_documents"]

# 示例使用
question = "云安全中心有什么功能？"
answer, sources = ask_question(question)

print(f"问题: {question}")
print(f"答案: {answer}")
# print("\n来源文档:")
# for i, doc in enumerate(sources):
#     print(f"文档 {i+1}:")
#     print(doc.page_content)
#     print("-" * 50)
```



至于参数的选择，embedding模型就找一个下载量高的，Claude给我推荐的是sentence-transformers/paraphrase-multilingual-mpnet-base-v2

chain_type如果没有特殊要求，或者特殊场景一般默认的Stuff就够用了。其他的几个类型定义以及对比如下：

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729834513949-8cde02ae-a283-4c89-bf01-cf0b459a0885.png)



好了，上面的代码不出意外，是可以直接运行的

不过还是出了点意外，自己本地跑的时候一直有一个报错

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729751469756-0b53d7a7-53d3-4ff0-b802-e99a51e59676.png)

debug看一下，发现提示 Workspace.AccessDenied

后来发现应该是通义的API迭代了，之前申请的key是没有业务空间的概念的，虽然有效但是没办法使用。

重新申请了一个解决

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729754132734-fb262b1d-df6f-4d0a-b2aa-514695aa61a6.png)



开始提问：云安全中心有什么功能

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729755542226-8e6239bd-fb18-4a28-8786-2c52c05d52a1.png)

谁是yzddmr6

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729754405752-5f0ebc22-983a-4e66-981a-d5cf6ea8befd.png)

问一个不在知识库里的问题：谁是yzddmr7。虽然没有找到，但是进行了相似的推荐。

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729754466184-f35dde5b-a4c6-4d5b-aae4-20b905fe1594.png)



接着一个不在知识库里的问题：介绍一下腾讯公司

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729822826588-843eb798-a15f-4553-a5b5-a301e358a9f1.png)

可以看到，虽然大模型回答没有找到腾讯公司的信息，但是还是以另外一种方式介绍了腾讯公司。这肯定是不符合预期的。



我们可以通过修改prompt来进行约束，禁止大模型回答知识库以外的内容。

```java
你是一个只基于检索到的信息回答问题的ai助手，这里是对你的要求：
1,请仔细分析提供的上下文信息，并只使用其中包含的事实来回答问题。
2.如果问题无法完全用检索到的信息回答，请说明你无法回答该问题或该问题的某些部分。
3.不要编造，推测或添加任何未在检索结果中明确提供的信息。
```

这样大模型就不会再回答知识库以外的内容了。

![](https://intranetproxy.alipay.com/skylark/lark/0/2024/png/304053/1729823288769-f25e6073-ab50-4152-ae69-d860434c9c38.png)

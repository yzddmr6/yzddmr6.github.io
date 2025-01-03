# 大模型应用之工具调用与流程编排


<meta name="referrer" content="no-referrer" />


## 背景
如果把大模型比作大脑的话，调用工具就相当于给大模型安装上了四肢和双眼，不再局限于文字形式，可以跟物理世界产生交互。通过工具调用，开发人员就可以构建复杂的大模型应用，可以利用 LLM 访问、交互和操作外部资源，例如数据库、文件和 API等。

本文以开源框架LangChain以及阿里云的商业化平台百炼进行学习调研。

## LangChain
大模型开源框架中最出名的便是LangChain，在这里以LangChain为例，学习一下如何在大模型中调用工具。

### 搞个Demo
这里我模拟了一个场景，根据当前的天气推荐对应的音乐，设置了两个工具：

+ WeatherTool：获取当前的天气。这里直接为了方便就直接random一下了。
+ MusicTool：获取音乐列表。这里也是返回一个固定列表，并且会带有音乐的情绪风格。

同上篇文章一样，为了兼容国内的网络环境，将模型切换为通义千问。

```plain
import random
from langchain.agents import Tool
from langchain.agents import AgentType
from langchain.memory import ConversationBufferMemory
from langchain.agents import initialize_agent
from langchain.llms import Tongyi


# 定义WeatherTool
class WeatherTool:
    def get_weather(self, location: str):
        return random.choice(["Sunny", "Rainy", "Snowy"])


# 定义MusicTool
class MusicTool:
    def get_music(self, song: str):
        return [
            {"name": "HappySong", "type": "happy"},
            {"name": "SadSong", "type": "Sad"},
            {"name": "AngrySong", "type": "Angry"}
        ]


# 创建工具实例
weather_tool = WeatherTool()
music_tool = MusicTool()

# 定义LangChain工具
tools = [
    Tool(
        name="Weather",
        func=weather_tool.get_weather,
        description="Useful for getting the current weather"
    ),
    Tool(
        name="Music",
        func=music_tool.get_music,
        description="Useful for getting music information"
    )
]

# 初始化语言模型
# llm = ChatOpenAI(temperature=0)
import os

os.environ["DASHSCOPE_API_KEY"] = "xxxx"
llm = Tongyi(model="qwen-turbo")
# 初始化内存
memory = ConversationBufferMemory(memory_key="chat_history", return_messages=True)

# 初始化agent
agent = initialize_agent(
    tools,
    llm,
    agent=AgentType.CHAT_CONVERSATIONAL_REACT_DESCRIPTION,
    memory=memory,
    verbose=True
)

# 使用agent
response = agent.run("What's the weather like today?")
print(response)

response = agent.run("Can you recommend a song based on the weather?")
print(response)

```



点击运行，LangChain默认会把当前的思考以及工具调用过程打印出来。



最终，根据当前的天气是晴天，大模型给我们推荐了一首欢快的歌曲。

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1734080525525-631bfac0-a281-426e-9370-ceab46b5c1de.png)



### ReAct模式
从输出结果中我们可以看出，大模型是经过了一系列的思考过程，这个过程是怎么实现的呢？这里涉及到一个概念：ReAct模式

> ReAct是Reasoning and Acting的缩写，意思是LLM可以根据逻辑推理（Reason），构建完整系列行动（Act），从而达成期望目标。LLM灵感来源是人类行为和推理之间的协同关系。人类根据这种协同关系学习新知识，做出决策，然后执行。
>

ReAct方式的作用就是协调LLM模型和外部的信息获取，与其他功能交互。如果说LLM模型是大脑，那ReAct框架就是这个大脑的手脚和五官。同时具备帮助LLM模型获取信息、输出内容与执行决策的能力。对于一个指定的任务目标，ReAct框架会自动补齐LLM应该具备的知识和相关信息，然后再让LLM模型做出决策，并执行LLM的决策。



一般分为如下几个步骤：

+ Action：使用工具进行搜索、执行、查询等操作
+ Observation：对行动结果进行观察和分析
+ Thought：根据反馈进行思考，规划下一步行动
+ Final Answer：得出结论，完成任务

可以在这里看到比较典型的ReAct Prompt：[https://smith.langchain.com/hub/hwchase17/react](https://smith.langchain.com/hub/hwchase17/react)

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735291763180-d2f26d31-728a-4d90-9b4d-f04081727832.png)



跟着代码调试一下过程更容易理解这个过程：



在langchain.agents.agent.AgentExecutor._call中，会进入核心的大循环

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1734082436636-c9a23a5d-2c4d-40e6-b42a-a867c16b338a.png)



<font style="color:#080808;background-color:#ffffff;">可以看到有两个退出的条件</font>

1. <font style="color:#080808;background-color:#ffffff;">超过最大执行时间限制</font>
2. <font style="color:#080808;background-color:#ffffff;">大模型认为任务结束，进入AgentFinish状态，return，终止循环</font>

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1734082481463-c5b43c79-4314-433e-b22f-96f8e0e1d5f3.png)

将上一步的输出output，作为下一步的action

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1734082545426-8b992731-be7c-4c35-a7af-5aa3d6e155d1.png)

<font style="color:#080808;background-color:#ffffff;">每一步的中间过程会加入intermediate_steps这个数组</font>

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1734083076637-4f7afb2c-67d7-410f-9fd0-2f37a873f301.png)

```plain
TOOL RESPONSE: 
---------------------
[{'name': 'HappySong', 'type': 'happy'}, {'name': 'SadSong', 'type': 'Sad'}, {'name': 'AngrySong', 'type': 'Angry'}]

USER'S INPUT
--------------------

Okay, so what is the response to my last comment? If using information obtained from the tools you must mention it explicitly without mentioning the tool names - I have forgotten all TOOL RESPONSES! Remember to respond with a markdown code snippet of a json blob with a single action, and NOTHING else.
```

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1734081274512-8192270a-eb49-47c3-a501-9530a47862c6.png)



![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1734081575215-d3e22682-9f08-48a0-9748-af9a7dab8119.png)

在langchain.agents.conversational_chat.output_parser.ConvoOutputParser.parse中

遇到Final AnsWer，向上return AgentFinish状态，该调用链结束。根据意图识别的结果来决定，如果还有未解决的问题就再进入新的下一轮循环。

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1734081914482-05fe12c7-c6a9-4d15-8b6e-aeab7305c950.png)

直到解决所有问题，获取最终的答案。

```plain
{
    "action": "Final Answer",
    "action_input": "Since it's rainy, you might enjoy listening to SadSong to match the weather."
}
```



### 小总结
LangChain总体的体验还是不错的，灵活度比较高。通过少量的代码就可以直接完成意图识别、参数提取、工具调用的动作。

但是其中遇到一个问题：关于音乐列表这个工具，我的设想只是通过一个接口拿到所有的音乐列表，并不需要传参数。但是发现langchain的工具似乎默认必须要有一个参数，不然就会报错。不知道是不是我的操作有问题，查阅文档以及GPT都没有解决这个问题。

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1734080560829-1365ad1f-98ed-4e82-9347-b5e2b2ba1ad3.png)

## 阿里云百炼
阿里云百炼平台上提供了较为方便的流程编排功能，通过拖拽组件可以实现比较复杂的流程处理。

### 流程管理
百炼里面有好几个比较类似的概念：有一个流程管理、还有一个流程编排应用，智能体应用、智能体编排应用，一开始有点傻傻分不清楚。

最开始用的是这个流程管理，但是发现写起来体验并不好，不能单组件测试，以及没有变量引用自动补全。

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735290234732-b7be5e53-3797-415c-bc2c-2a562c227bcf.png)

后来发现，原来这个 流程编排应用 其实等于 升级版的流程管理，操作更加简单方便，于是后续改为采用流程编排应用实现。

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735309574005-dea934f4-bbef-48e4-9860-0478091f6f49.png)

### 流程编排应用
这里内置了很多节点，我们可以自由组合

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735310370425-f26f7d30-dc3a-42bf-861e-a16f1ef40d4f.png)



添加意图识别

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735309712415-8d8b9faf-53b0-40aa-a486-36ae03600c57.png)

其中，查询天气需要城市跟日期两个参数，但是百炼没有专门的参数提取节点。不过没关系，我们可以通过大模型节点来实现参数提取的功能。



```plain
你是一个从自然语言中提取参数的助手，请根据用户的输入，提取指定的参数
根据用户的输入，提取城市：city，以及日期：date两个参数，并用json格式输出。
输出样例如下：
{
    "city":"hangzhou",
    "date":"2024-01-01"
}

用户的输入如下：
${sys.query}
```



![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735289375635-ed12ea09-920d-4d06-9179-090ab9c3bc6d.png)

然后在脚本中，通过json解析取出对应字段。

```plain
def main():
    import json
    import random
    city=json.loads(params["input"])["city"] 
    date=json.loads(params["input"])["date"] 
    ret = {
        "result": {
           "city": city, 
           "date": date, 
            "ret":random.choice(["Sunny", "Rainy", "Snowy"])
            }
    }
    return ret

```



模拟查询音乐列表的工具，由于不需要参数，比较简单

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735214365906-3f5de42c-d62e-42e8-b38e-1bfbcaf56cfd.png)



```plain
def main(): 
  import json  
  ret = {        
    "result":  [
            {"name": "HappySong", "type": "happy"},
            {"name": "SadSong", "type": "Sad"},
            {"name": "AngrySong", "type": "Angry"}
        ]    
  }    
  return ret
```



测试一下，没有问题

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735289285879-b162e1bc-8b90-415e-b46d-88337e991552.png)



但是当我把问题改为：帮我查询杭州明天的天气，并根据天气推荐一首音乐 这样需要调用两个工具的复杂意图的时候，只调用了第一个意图。

看起来流程编排并不能像LangChain一样对提供的工具列表进行自动识别并多次调用，我也没有找到哪里可以实现循环的地方。

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735289458780-87f77c8c-52de-48fa-bee3-7c20007c06c0.png)

整体流程布局如下

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735289640784-1e1c0819-ffd3-414e-8357-ac11f50b0b18.png)



### 为什么不用智能体？
目前百炼的智能体里支持调用外部方式的一个是插件，一个是流程。

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735308209947-1f6dff37-1dbf-434f-9cb9-fd9873429c4c.png)

插件只能用yaml格式来调用接口，不能进行数据的二次处理；

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735289712535-c31f33fe-2208-4bab-83d2-668057d8e716.png)

而流程现在只支持了老版本不好用的的流程管理，不支持新版本的流程编排应用。

加上直觉感觉并不能解决我上面遇到的问题，于是就没有再进一步测试。

### 小总结
发现了一个小BUG，在切换标签之后，会自动把用户自己的代码替换为模板。

复现步骤如下：

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735308580560-57ba3f1b-edb9-4f0b-8dcc-b19652c8c811.png)

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735308606848-edbe1bac-a3d7-4228-b6bf-6e634f5b7ea2.png)

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1735308656682-121ca43c-6fb8-4393-a57c-3f44efc39d97.png)



产品似乎正处在迭代过程中，同时存在新旧两套逻辑，同时官方文档给出的样例并不是非常的丰富，有些功能的设计需要摸索跟猜测。

## 总结
AI结合物理世界是一个不可逆转的大趋势，现在已经出现用可以控制PC/手机的AI Agent，只需要说一句话就可以帮你订机票，点外卖。相信在不久的将来，AI会像水、电、天然气一样融入生活中的方方面面。

对于安全行业来讲，以前想要达到RCE需要利用构造好的二进制与代码，以后的RCE可能就是通过一条精心设计的Prompt；以前的危害可能是能控制一台远程服务器，获取赛博世界的权限，以后可能就是控制汽车/机器人发起对物理世界的攻击。

LangChain这样的积木式框架以及百炼这样的图形化编排平台，大大降低了大模型的使用门槛，以后人人都可以拥有专属于自己的Agent。除此以外，还有很多优秀的大模型应用框架以及平台，在后续的文章中我会继续介绍。



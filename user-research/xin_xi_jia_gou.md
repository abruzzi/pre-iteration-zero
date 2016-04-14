## 信息架构

以前在互联网公司工作的时候，会长期扑在一个项目上少则半年多则两年，好处是能充分发挥一颗螺丝钉的特长，坏处是没有办法站在全局的高度看到产品生长过程。现在的工作与之前有很大的不同－－软件交付，需要在3－6个月当中为客户交付一个落地的产品，由于时间紧变化快，没办法深究细节，需要快速让产品从0到1 ，这要求设计师必须具有全局观，能掌控每一个节点并且又快又好。

我在前面的文章里有陆续讲到用户调研和用户画像的基本知识，这篇文章主要想讨论从用户画像到产品信息架构的过程是什么，怎么样才能确保搭建架构的速度又快，质量又高。

为了方便理解，我们把这个过程变成调制鸡尾酒的过程，假如你是一位鸡尾酒师，你要为顾客调制一杯符合他口味的鸡尾酒，你首先会问他想喝什么样的，如果顾客是个行家，就直接告诉你酒名了，但是我们遇到的大多数客户都是迷茫的，只会描述一些他的需求：“我喜欢橙子的味道、喜欢红色的、酒精可以多一点！”

- 【**建立persona**】这时，你可以迅速地在你心里构建一个persona，将这个顾客和之前的某个顾客对应起来；

- 【**构建场景**】构建一个场景让确认一下他想要的感觉：“您是不是想要一种高兴热烈的感觉?就像沐浴阳光一样？”；

- 【**确认需求**】确认一下他想要的：“我们用橙子+樱桃两种水果，调出来是红色的，配上高酒精度的龙舌兰，可以吗？”

- 【**建立信息架构**】顾客说“OK”，你就可以滚去调酒了：切樱桃、切橙片、拿出龙舌兰和红石榴糖浆，哐哐哐哐调好一杯“龙舌兰日出”

**所以从构建persona到信息架构，最粗略的可以分为三步：构建场景--确认需求--建立信息架构**

### 构建场景

构建场景的目的是为了突出人的目的、感知和期望，我们需要帮每个persona构建情景场景，把人物放到场景中去探索产品如何更好的服务于Persona。

场景可以类似于这种：小明忙碌工作的一天，早上起床打了个电话给mary约会议时间，中午和客户沟通见面时间，在日历上打上记号，下午出去打印合同blabla。

### 确认需求

你需要从persona和它所处的场景中揪出哪些需求是必要的，哪些是不必要的，并且找到利益相关者做确认。有以下三种需求需要考虑：

- 数据需求：记录下场景中出现的人、地址、文件、消息、歌曲。
- 功能需求：归纳出场景中需要的功能模块：打电话模块、聊天模块、视频模块、好友模块
- 情景需求：把完成任务需要做的一整串行为动作列出来，如果需要角色之间需要切换（类似工作流），也列下来。

### 建立信息架构和关键路径（user journey）

- 确定大致的形式要素：做移动端或者网页端，ios还是安卓等
- 让数据、功能会被当作原材料进入到系统架构当中，把类似的功能和类似的数据放在一起。例如打电话功能和好友功能在一组，聊天功能和视频功能在一组。
- 把情景需求简化成几个关键的user journey。（如果是2B产品，面临很多角色和任务的扭转，所以在画user journey之前还需要画一个泳道图表明任务承接关系和角色之间的上下游关系）
- 把user journey里面最关键的数据和功能抽出来做优先级排序、重点设计。(这一步决定了产品最主要的功能是什么，是否能满足用户价值)
- 找用户测试一下设计出来的信息架构是否能够帮助他们实现价值并且操作简便；找运营或者产品经理询问，抓取的这些数据能不能够支撑产品。

*以上是根据我自己做项目的经验加上理论知识归纳出的【构建信息架构的方式】，然而根据场景不同，步骤也会稍有不同，仅供参考。*




## CPU

- 高频查询，应该使用set或dict，而不是list
- 列表添加可迭代对象的多个元素时，应该用exted或+=，而不是append



## 内存

- 当产生大量对象，能使用类成员的情况下，就不要用对象成员

- 当产生大量对象，使用\_\_slots\_\_，从而节省对象的\_\_dict\_\_开销

- 池化技术：动态语言高频的对象申请，gc不及时很容易带来内存上涨以及内存占用不稳定等问题，因此对于大量高频的小块内存申请，一般考虑对象池优化。

  ```python
  # before optimization
  class CElemMgr:
      def __init__(self):
          self.m_Elems: Dict = {}
  
      def AddElem(self, oData) -> CElem:
          oElem = CElem(oData.info)
          # ...
          self.m_Elems[oData.info] = oElem
          return oElem
  
  # after optimization
  # 使用池来存储被destroy的oElem，需要新增对象时再从池中取，并初始化
  class CElemMgr:
      def __init__(self):
          self.m_Elems: Dict = {}
  
      def AddElem(self, oData) -> CElem:
          oElem = CElem.New(oData.info)
          # ...
          return oElem
  
      def DestroyElem(self, oData):
          oElem: CElem = self.m_Elems.pop(oData.info, None)
          if oElem:
              g_ElemPool.append(oElem)
  
  class CElem:
      @classmethod
      def New(cls, oInfo) -> CElem:
          if g_ElemPool:
              oElem = g_ElemPool.pop()
              oElem.__init__(oInfo)
          else:
              oElem = CElem(oInfo)
          return oElem
  
  if "g_ElemPool" not in globals():
      g_ElemPool: List[CElem] = []
  ```

  本质上这个优化是没有减少内存的总消耗的，反而由于g_ElemPool中的内存冗余，会导致内存消耗增加，核心目的是为了避免小内存的高频申请与gc带来的总内存的不稳定的上升。（如果是go语言，那就可以直接申请一大块内存，用完后自动gc大块内存，这样会更好）

- 享元模式：共享数据从而减少内存消耗

  ```python
  class CElem:
      def __init__(self):
          self.m_ID: int = 0
          self.m_OtherInfo: List = []
      
      def __eq__(self, oOther: CElem):
          # ...
      
      def __hash__(self) -> int:
          # ...
  
  
  if "g_Flyweight" not in globals():
      g_Flyweight: Dict[int, int] = {}
      g_TempElem: CElem = CElem()
  
  
  def NewElem(iID: int, lInfo: List) -> CElem:
      g_TempElem.m_ID = iID
      g_TempElem.m_OtherInfo.extend(lInfo)
      oTrueElem = g_Flyweight.get(g_TempElem, None)
      if oTrueELem:
          return oTrueElem
     	oTrueElem = CElem()
      oTrueElem.m_ID = iID
      oTrueElem.m_OtherInfo.extend(lInfo)
      g_Flyweight[oTrueElem] = oTrueElem
      return oTrueElem
  ```



## 极限优化

### AOI

- 无AOI时，遍历会有很大的消耗（用AOI应该是可以优化一个数量级的）
- 合适的AOI数据结构能够帮助节省内存（本质上和AOI无关，主要是数据结构上的优化）

### 存盘

- 不使用对象，直接用str，减少内存开销，并且减少序列化和反序列化的开销。当然需要操作数据时需要在str和具体数据格式之间进行转化，但是如果这类操作是低频操作，那么这部分额外的开销可以忽略。

### 递归

- 慎用递归，使用非递归替代。（主要是安全问题，消耗方面也会有优化，具体问题具体分析）
- 中间变量的处理：以坐标为例，中间变量为元组和中间变量为int，这两者的内存差别是很大的，虽然都是中间变量，但是申请释放内存，以及这部分的时间消耗，都会对程序的性能有影响。

### 自定义对象

- 自定义对象存储数据时，消耗是较大的，因为需要维护一个字典\_\_dict\_\_
- 元组和变长参数的转换：有时候用元组（列表应该也是一样）存储，可以考虑用变长参数（例如这里的元组元素都是int，则用变长参数维护这些int就行，会更节省内存）



## 卡帧

- 例如满级号的gm，服务器一帧需要跑几十上百个节点，部分节点里面的调用又很多，会导致服务器卡顿。此时可以在部分消耗较大的节点执行时先停一下，用一个定时器，两秒后再跑，这是一种很简便的定时分帧处理。
- 类似秒杀系统。秒杀系统使用的是消息队列，例如同时有100万个用户，可以根据客户端的请求的先后放入到消息队列中，因为“放入消息队列”这个操作很简单，只需要存储用户id和商品的相关信息就行，没有相关的逻辑处理。然后服务器再从消息队列中按批次取一定量的数据，例如一千个一次去取，分开处理即可。
- 数据安全：为了保证大量数据不丢失，设置主从服务器。主服务器定时向从服务器同步当前的消息队列，如果主服务器宕机，则立刻拉起从服务器来处理当前的消息。但是一般来说主服务器不会宕机，因为使用消息队列的情况下，服务器只需要简单存储客户端的请求，再按照第二点说的分批次一点一点处理就行了。
- 估算失误：原本准备了10台主服务器，20台从服务器，能够应对1000万用户的请求，但是由于广告做的太好了，吸引了3000万用户，为原估算的三倍，这时可能就会出现卡顿了。可以准备一些应急方案：。。。（例如从服务器直接拿来用）
- 用户还是觉得卡的原因：对服务器来说，还是正常地从消息队列中拿数据进行逻辑处理，但是客户端需要等待服务器逻辑处理完毕后的返回，比如告诉客户端库存不够，或者购买成功等，这段逻辑处理的时间加上消息队列等待的时间就是用户等待的时间了（当然还有其他耗时的地方，但整体过程就是：客户端请求->服务器消息队列存储->消息队列满（这里满是指批次处理的请求个数）->消息队列取出数据依次处理->返回客户端处理结果）。**（秒杀的这部分内容可以再去网上看一下）**

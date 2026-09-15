项目名称：Benchmark 驱动的 VLM-Agent 视觉证据数据引擎
项目背景：
项目动作：
1. 检索：
    与长视频复杂推理项目中不同的是：视频的获取基于每个类型已有的视频样例，属于根据视频搜集视频要求视频类型完全一致，检索使用拓展类检索，由caption->关键词->url->抽帧筛选->可用url
2. 抓取：
    自研fetch平台：抓取可用url->本地视频->转存为tos链接
3. 数据构造（重中之重，利用阿里rewatch-r1公开的技术生成带有时间戳的video-caption、video-qa、video-cot）：
  1. caption合成：
    1. 低帧率语义切段：语义分段模型mseg以低帧率浏览完整视频，将视频切分为k个事件的完整时间区间si=【tstrat，i tend，i】。依据‘语义连贯、事件完整’原则进行切分，而非固定时长。
    2. 高帧率局部描述：描述模型mcap以高帧率逐片段率浏览si，在si内部识别多个事件cij，并为每个事件输出相对时间戳Tij。
      ***分层的目的***：全局片段找到事件边界，局部阶段把视觉集中到短片段，保留动作、物体、人物交互和先后关系
    3. 时间戳重对齐：全视频绝对时间戳tij=tstart，i+Tij，所有分段事件合并为 ***Cdetail（V）***
      caption=带时间戳索引的事件库，用于后续QA生成和Observer检索
  2. QA合成：重点解决出题必须依赖题目细节
      采用对比差异生成，再利用三层过滤去除事实错误与文本捷径。
    1. 构造信息差异
      1. 由model把caption Cdetail（V）压缩为Summary Csum
      2. 再由model同时读取Cdetail（V）与Csum，生成能够由Cdetail（V）回答，但是Csum不能回答的原始QA
      3. QA生成被扩展10类任务，避免题型单一。
        ***注意***Csum属于负信息参考，代表只看视频梗概时可获得的知识。所以必须把问题放到Cdetail（V）与Csum的差异中，才能制造细节差异。
    2. 三层过滤
      1. 答案核验，验证输入Q+Cdetail（V）的输出answer是否与构造的A一致。筛掉错答与无依据的答案
        合格（下一层）/不合格（舍弃）
      2. 文本偏执核验，验证只输入Q的answer是否与构造的A一致。筛掉常识类与语言先验捷径
        合格（下一层）/不合格（舍弃）
      3. 摘要偏执核验，验证输入Q+Csum的输出answer是否与构造的A一致。筛掉仅凭梗概就可答的简单题
  3. cot合成：流程是Reasoner下检索指令->Observer查资料并汇报->Reasoner继续判断或作答->Convert整理并记录生成
    1. 定义2个agent：
      Reasoner AR：生成思考T和动作Act
      Observer AO：在Cdetail（V）上执行动作Act，并返回观察结果Obs
      对于给定Q，两个agent循环交互:
      - Reasoner AR是动作决策与发起者，根据问题和历史轨迹判断缺少什么证据，生成思考Thought，再提出动作Action也就是segment_retrieval()和segment_query()
      - Observer AO是执行者，接收Reasoner 提出的Action，在Cdetail（V）中执行检索，并返回Observation
      - Reasoner再消费Observation：判断证据是否充分，如果不足，则继续提出动作Action；如果充分则生成Final Answer
    2. 定义1个llm：
      Convert：在agent交互结束且返回Final Answer后，把完整轨迹整理为带有<action>和<observation>标签的自然语言cot
    



4. 数据交付

项目结果：
项目思考：


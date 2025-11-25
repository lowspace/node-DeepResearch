# DeepResearch

[Official UI](https://search.jina.ai/) | [UI Code](https://github.com/jina-ai/deepsearch-ui) | [Stable API](https://jina.ai/deepsearch) | [Blog](https://jina.ai/news/a-practical-guide-to-implementing-deepsearch-deepresearch)

**架构**：

<img width="1080" height="450" alt="image" src="https://github.com/user-attachments/assets/4b75934f-7a4d-47d9-aa96-720605cfaf18" />

```
// 主推理循环
while (tokenUsage < tokenBudget && badAttempts <= maxBadAttempts) {
  // 追踪进度
  step++; totalStep++;

  // 从 gaps 队列中获取当前问题，如果没有则使用原始问题
  const currentQuestion = gaps.length > 0 ? gaps.shift() : question;

  // 根据当前上下文和允许的操作生成提示词
  system = getPrompt(diaryContext, allQuestions, allKeywords,
                    allowReflect, allowAnswer, allowRead, allowSearch, allowCoding,
                    badContext, allKnowledge, unvisitedURLs);

  // 让 LLM 决定下一步行动
  const result = await LLM.generateStructuredResponse(system, messages, schema);
  thisStep = result.object;

  // 执行所选的行动（回答、反思、搜索、访问、编码）
  if (thisStep.action === 'answer') {
    // 处理回答行动...
  } else if (thisStep.action === 'reflect') {
    // 处理反思行动...
  } // ... 其他行动依此类推
}
```


## Findings

- https://github.com/lowspace/node-DeepResearch/issues/3 Jina AI 的架构和 [Langchain](https://github.com/langchain-ai/open_deep_research/tree/main) 的不同，Jina AI 采用了「先拆分 TOC，然后拆分不同 TOC 的分治方法」，Langchain 在早期也使用了这种方案；Manus 在最开始出来的时候，也是通过罗列大量的 to-do list 进行任务分解，然后按照既定线路逐一完成。这种提前规划的方案容易出现「最终报告的内容一致性和文本连贯性的问题」，因为 TOC 以下的内容基本是独立进行了，相互之间没有任何通信，同时即使在任务过程中出现了 tangential connection 或者超越当前 knowledge cutoff 的新的权威知识/领域，很有可能没法 cover 到 TOC 里面。
- 在 deep reserach 中不使用 RAG，而直接把所有的信息扔到 context 里面进行处理 => 需要大量的 context engineering。
- 「rewrite user query」被认为是核心提点措施，不仅能将 user input query 转化为更适合 BM25 算法处理的关键词形式，还能拓展查询范围，覆盖多语言、语调、格式下的潜在答案。Jina AI 写了一个多语言的包含 thinking 过程的 few-shots 的 rewriter 来干这活。
- https://github.com/lowspace/node-DeepResearch/discussions/9 Jina 设计了一个评估模块来判断回答的质量，这个评估模块会根据「预定义的规则集」对回答质量进行评估。我认为这个模块缺乏弹性以及可维护性 —— 1. 细致的规则可能会在缺乏足够上下文的时候判断出错，比如认为回答太过于发散 2. 细致的规则集之间的化学反应很难 debug，可能过于严格了 3. 规则的后续维护是大问题，永远需要有人去维护一个很大的规则集，而且模型版本更新后，模型性格也会变，prompt 可能也会随之改变。llm-as-judge 是不错的选择，但给定规则集有点太死板了。
- https://github.com/lowspace/node-DeepResearch/discussions/7 thinking budget 既可以通过在提示词中要求推理模型使用「wait」等 thinking tokens 延长推理时间，又可以通过 API 中 `reasoning_effort` 进行控制。`reasoning_effort` 的实现结合了 hard limit 和 soft limit 两种，前者是通过 token counter 直接进行截断，比如只允许思考 2k tokens，到达的时候直接截断 thinking；后者是通过 RL 训练 llm 自主选择回复的长度（应该可以通过注入训练时提醒模型生成长文的 token 要求模型一直生成长回复）。
- https://github.com/lowspace/node-DeepResearch/discussions/15 很有意思的是，Jina 中数据传递基本都使用的是 JSON，因此 Jina 很看重 JSON schema。Langchain 和 Anthropic 在介绍 deep research 的系统中都没有提到，Langchain 在具体实现中只使用了一两次 JSON schema。我认为这里可能是 Jina 有 JSON 的路径依赖了，开发中有各种 state，使用 JSON 是必然的结果 => 没有使用 langgraph 这些框架还是有其劣势。
- https://github.com/lowspace/node-DeepResearch/discussions/1#discussioncomment-14554758 Jina 开发了一种基于 embedding 的滑动分块方法：先转 embedding；再确定分块的级别，字符、句子、段落、语义等；再滑动计算 similarity 提取最相关的文本块。就像滑动窗口一样，提取出来的文本块是 embedding 最相似的。
- https://github.com/lowspace/node-DeepResearch/discussions/1#discussioncomment-14574970 Jina 使用了一个 rerank 策略对 webpage 进行打分，该策略考虑了「频率信号」，越热门越权威；「路径结构」，路径越短越重要；「语义相关性」，文本和 query 之间的联系程度；「最后更新时间」，越新的最重要；还有其它一些关于网络结构的特定优化，比如 paywall 的 blacklist，同一域名下的数量限制等。

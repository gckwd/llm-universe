# Embedding封装讲解
本文对应源代码在[此处](https://github.com/datawhalechina/llm-universe/blob/main/notebook/C3%20%E6%90%AD%E5%BB%BA%E7%9F%A5%E8%AF%86%E5%BA%93/%E9%99%84LangChain%E8%87%AA%E5%AE%9A%E4%B9%89Embedding%E5%B0%81%E8%A3%85%E8%AE%B2%E8%A7%A3.ipynb)，如需复现可下载运行源代码。

LangChain 为基于 LLM 开发自定义应用提供了高效的开发框架，便于开发者迅速地激发 LLM 的强大能力，搭建 LLM 应用。LangChain 也同样支持多种大模型的 Embeddings，内置了 OpenAI、LLAMA 等大模型 Embeddings 的调用接口。但是，LangChain 并没有内置所有大模型，它通过允许用户自定义 Embeddings 类型，来提供强大的可扩展性。

在本部分，我们以智谱 AI 为例，讲述如何基于 LangChain 自定义 Embeddings。

本部分涉及相对更多 LangChain、大模型调用的技术细节，有精力同学可以学习部署，如无精力可以直接使用后续代码来支持调用。

要实现自定义 Embeddings，需要定义一个自定义类继承自 LangChain 的 Embeddings 基类，然后定义三个函数：① _embed 方法，其接受一个字符串，并返回一个存放 Embeddings 的 List[float]，即模型的核心调用；② embed_query 方法，用于对单个字符串（query）进行 embedding。③ embed_documents 方法，用于对字符串列表（documents）进行 embedding。

首先我们导入所需的第三方库：


```python
from __future__ import annotations

import logging
from typing import Dict, List, Any

from langchain.embeddings.base import Embeddings
from langchain.pydantic_v1 import BaseModel, root_validator

logger = logging.getLogger(__name__)
```

这里我们定义一个继承自 Embeddings 类的自定义 Embeddings 类：


```python
class ZhipuAIEmbeddings(BaseModel, Embeddings):
    """`Zhipuai Embeddings` embedding models."""

    client: Any
    """`zhipuai.ZhipuAI"""
```

在 Python 中，root_validator 是 Pydantic 模块中一个用于自定义数据校验的装饰器函数。root_validator 用于在校验整个数据模型之前对整个数据模型进行自定义校验，以确保所有的数据都符合所期望的数据结构。

root_validator 接收一个函数作为参数，该函数包含需要校验的逻辑。函数应该返回一个字典，其中包含经过校验的数据。如果校验失败，则抛出一个 ValueError 异常。

这里我们只需将`.env`文件中`ZHIPUAI_API_KEY`配置好即可，`zhipuai.ZhipuAI`会自动获取`ZHIPUAI_API_KEY`。



```python
@root_validator()
def validate_environment(cls, values: Dict) -> Dict:
    """
    实例化ZhipuAI为values["client"]

    Args:

        values (Dict): 包含配置信息的字典，必须包含 client 的字段.
    Returns:

        values (Dict): 包含配置信息的字典。如果环境中有zhipuai库，则将返回实例化的ZhipuAI类；否则将报错 'ModuleNotFoundError: No module named 'zhipuai''.
    """
    from zhipuai import ZhipuAI
    values["client"] = ZhipuAI()
    return values
```

接下来我们重写 `_embed` 方法（在本章第二节有定义及演示），调用远程 API 并解析 embedding 结果。


```python
def _embed(self, texts: str) -> List[float]:
    embeddings = self.client.embeddings.create(
        model="embedding-2",
        input=texts
    )
    return embeddings.data[0].embedding
```

重写 embed_documents 方法，因为这里 `_embed` 已经定义好了，可以直接传入文本并返回结果即可。


```python
def embed_documents(self, texts: List[str]) -> List[List[float]]:
    """
    生成输入文本列表的 embedding.
    Args:
        texts (List[str]): 要生成 embedding 的文本列表.

    Returns:
        List[List[float]]: 输入列表中每个文档的 embedding 列表。每个 embedding 都表示为一个浮点值列表。
    """
    return [self._embed(text) for text in texts]
```

`embed_query` 是对单个文本计算 embedding 的方法，因为我们已经定义好对文档列表计算 embedding 的方法`embed_documents` 了，这里可以直接将单个文本组装成 list 的形式传给 `embed_documents`。


```python
def embed_query(self, text: str) -> List[float]:
    """
    生成输入文本的 embedding.

    Args:
        texts (str): 要生成 embedding 的文本.

    Return:
        embeddings (List[float]): 输入文本的 embedding，一个浮点数值列表.
    """
    resp = self.embed_documents([text])
    return resp[0]
```

可能大家会好奇，为什么要先定义`embed_documents`再用 `embed_query` 调用呢，不返过来呢，其实也是可以的，`embed_query` 单独请求也是可以的。

对于 `embed_documents` 可以加入一些内容处理后再请求 embedding，比如如果文档特别长，我们可以考虑对文档分段，防止超过最大 token 限制，这些都是可以的，靠大家发挥自己的主观能动性完善啦，这里只是给出一个简单的 demo。

通过上述步骤，我们就可以基于 LangChain 与 智谱 AI 定义 embedding 的调用方式了。我们将此代码封装在 zhipuai_embedding.py 文件中。

本文对应源代码在[此处](https://github.com/datawhalechina/llm-universe/blob/main/notebook/C3%20%E6%90%AD%E5%BB%BA%E7%9F%A5%E8%AF%86%E5%BA%93/%E9%99%84LangChain%E8%87%AA%E5%AE%9A%E4%B9%89Embedding%E5%B0%81%E8%A3%85%E8%AE%B2%E8%A7%A3.ipynb)，如需复现可下载运行源代码。
[TOC]

# 简介

前面已经看过了数据处理的 `tokenizer` 和 `Data Collator` 了.
下面会进入到模型层面, 主要以 bert 为例来看看模型是如何定义的.

# PreTrainedModel

首先当然是来看看基础的模型类.

先看看 `ModuleUtilsMixin` 类中一些有趣的方法.

```python
class ModuleUtilsMixin:
    @property
    def device(self) -> device:
        """
        `torch.device`: The device on which the module is (assuming that all the module parameters are on the same
        device).
        """
        return get_parameter_device(self)

    @property
    def dtype(self) -> torch.dtype:
        """
        `torch.dtype`: The dtype of the module (assuming that all the module parameters have the same dtype).
        """
        return get_parameter_dtype(self)
```

大部分是实用工具类的方法. `device` 和 `dtype` 这两个属性挺有用的. 两者的实现也是大同小异.

```python
def get_parameter_device(parameter: Union[nn.Module, GenerationMixin, "ModuleUtilsMixin"]):
    """获取参数所在的设备"""
    try:
        return next(parameter.parameters()).device
    except StopIteration:
        # For nn.DataParallel compatibility in PyTorch 1.5

        def find_tensor_attributes(module: nn.Module) -> List[Tuple[str, Tensor]]:
            tuples = [(k, v) for k, v in module.__dict__.items() if torch.is_tensor(v)]
            return tuples

        gen = parameter._named_members(get_members_fn=find_tensor_attributes)
        first_tuple = next(gen)
        # 这获取的 first_tuple 是前面的 (k, v), 所以要取出索引为 1 的
        return first_tuple[1].device
```

其他暂时没发现什么值得注意的了, 就继续来看 `PreTrainedModel`.

```python
class PreTrainedModel(nn.Module, ModuleUtilsMixin, GenerationMixin, PushToHubMixin):
```

`PreTrainedModel` 继承了三个类, 最关键的是 `nn.Module`, 毕竟要定义一个模型都需要继承该类,
然后是前面看过的带有一些实用函数的类 `ModuleUtilsMixin`. `PushToHubMixin` 就不看了, 是继承 hub 推送的类.

```python
def resize_token_embeddings(self, new_num_tokens: Optional[int] = None) -> nn.Embedding:
    model_embeds = self._resize_token_embeddings(new_num_tokens)
    if new_num_tokens is None:
        return model_embeds

    # 更新词汇表大小
    # Update base model and current model config
    self.config.vocab_size = new_num_tokens
    self.vocab_size = new_num_tokens

    # Tie weights again if needed
    self.tie_weights()

    return model_embeds
```

缩放 token 嵌入挺有意思的, 就是用来调整词汇表大小的. 主要的内容在 `model_embeds = self._resize_token_embeddings(new_num_tokens)` 中,
简单来讲就是如果是缩小词汇表大小, 就保留前 N 个词的嵌入; 如果是放大词汇表, 保留当前所有的词嵌入, 后面多出来的词的权重就随机初始化了.

对应的, 还有对位置嵌入的缩放, 可惜当前类中没有实现.

```python
def resize_position_embeddings(self, new_num_position_embeddings: int):
    raise NotImplementedError(
        f"`resize_position_embeddings` is not implemented for {self.__class__}`. To implement it, you should "
        f"overwrite this method in the class {self.__class__} in `modeling_{self.__class__.__module__}.py`"
    )
```

```python
base_model_prefix = ""

@property
def base_model(self) -> nn.Module:
    """
    `torch.nn.Module`: The main body of the model.
    """
    return getattr(self, self.base_model_prefix, self)
```

`base_model` 属性是用来获取模型的主体模型的.

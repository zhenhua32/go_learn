[TOC]

# 简介

前面看到了 tokenizer 部分. 现在还是继续来看数据处理部分的代码.
这次主要是和 `Data Collator` 有关.

`Data Collator` 是用于处理批量数据的, 输入本身是来自于 dataset 的数据,
经过一定处理后, 再返回, 是对输入数据的动态加工.

# DataCollator

```python
class DataCollatorMixin:
    def __call__(self, features, return_tensors=None):
        if return_tensors is None:
            return_tensors = self.return_tensors
        # 根据不同框架, 调用不同的方法
        if return_tensors == "tf":
            return self.tf_call(features)
        elif return_tensors == "pt":
            return self.torch_call(features)
        elif return_tensors == "np":
            return self.numpy_call(features)
        else:
            raise ValueError(f"Framework '{return_tensors}' not recognized!")
```

最重要的就是 `__call__` 方法, 对于不同框架, 使用不同的调用方法.

```python
def default_data_collator(features: List[InputDataClass], return_tensors="pt") -> Dict[str, Any]:
    # In this function we'll make the assumption that all `features` in the batch
    # have the same attributes.
    # So we will look at the first element as a proxy for what attributes exist
    # on the whole batch.

    if return_tensors == "pt":
        return torch_default_data_collator(features)
    elif return_tensors == "tf":
        return tf_default_data_collator(features)
    elif return_tensors == "np":
        return numpy_default_data_collator(features)

@dataclass
class DefaultDataCollator(DataCollatorMixin):
    return_tensors: str = "pt"

    def __call__(self, features: List[Dict[str, Any]], return_tensors=None) -> Dict[str, Any]:
        if return_tensors is None:
            return_tensors = self.return_tensors
        return default_data_collator(features, return_tensors)
```

`DefaultDataCollator` 就继承自 `DataCollatorMixin`, 没想到直接覆盖了 `__call__`.
这是因为自身没有定义 `self.tf_call` 之类的方法, 所以用外部的函数 `default_data_collator` 作为调用.

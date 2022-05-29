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

接着以 torch 为例, 来看看 `torch_default_data_collator`.

```python
def torch_default_data_collator(features: List[InputDataClass]) -> Dict[str, Any]:
    import torch

    if not isinstance(features[0], (dict, BatchEncoding)):
        features = [vars(f) for f in features]
    first = features[0]
    batch = {}

    if "label" in first and first["label"] is not None:
        label = first["label"].item() if isinstance(first["label"], torch.Tensor) else first["label"]
        dtype = torch.long if isinstance(label, int) else torch.float
        batch["labels"] = torch.tensor([f["label"] for f in features], dtype=dtype)
    elif "label_ids" in first and first["label_ids"] is not None:
        if isinstance(first["label_ids"], torch.Tensor):
            batch["labels"] = torch.stack([f["label_ids"] for f in features])
        else:
            dtype = torch.long if type(first["label_ids"][0]) is int else torch.float
            batch["labels"] = torch.tensor([f["label_ids"] for f in features], dtype=dtype)

    for k, v in first.items():
        if k not in ("label", "label_ids") and v is not None and not isinstance(v, str):
            if isinstance(v, torch.Tensor):
                batch[k] = torch.stack([f[k] for f in features])
            else:
                batch[k] = torch.tensor([f[k] for f in features])

    return batch
```

`torch_default_data_collator` 主要是处理了下数据, 最后返回字典形式.
就是将字典的数组转换成字典, 里面的每个 key 是一个数组.
其他两种框架的也是类似的处理, 就不多看了.

看完了 `DefaultDataCollator` 之后, 还有其他类似的类, 就挑 `DataCollatorWithPadding` 和 `DataCollatorForTokenClassification` 看看.

# DataCollatorWithPadding 和 DataCollatorForTokenClassification

```python
@dataclass
class DataCollatorWithPadding:
    tokenizer: PreTrainedTokenizerBase
    padding: Union[bool, str, PaddingStrategy] = True
    max_length: Optional[int] = None
    pad_to_multiple_of: Optional[int] = None
    return_tensors: str = "pt"

    def __call__(self, features: List[Dict[str, Any]]) -> Dict[str, Any]:
        batch = self.tokenizer.pad(
            features,
            padding=self.padding,
            max_length=self.max_length,
            pad_to_multiple_of=self.pad_to_multiple_of,
            return_tensors=self.return_tensors,
        )
        if "label" in batch:
            batch["labels"] = batch["label"]
            del batch["label"]
        if "label_ids" in batch:
            batch["labels"] = batch["label_ids"]
            del batch["label_ids"]
        return batch
```

`DataCollatorWithPadding` 的 `__call__` 方法比较简单, 主要是使用 `tokenizer` 的 `pad` 方法进行长度填充.
然后对标签字段做了统一, 将 `label` 和 `label_ids` 都转换成了 `labels`.

```python
@dataclass
class DataCollatorForTokenClassification(DataCollatorMixin):
    tokenizer: PreTrainedTokenizerBase
    padding: Union[bool, str, PaddingStrategy] = True
    max_length: Optional[int] = None
    pad_to_multiple_of: Optional[int] = None
    label_pad_token_id: int = -100
    return_tensors: str = "pt"

    def torch_call(self, features):
        import torch

        label_name = "label" if "label" in features[0].keys() else "labels"
        labels = [feature[label_name] for feature in features] if label_name in features[0].keys() else None
        batch = self.tokenizer.pad(
            features,
            padding=self.padding,
            max_length=self.max_length,
            pad_to_multiple_of=self.pad_to_multiple_of,
            # Conversion to tensors will fail if we have labels as they are not of the same length yet.
            return_tensors="pt" if labels is None else None,
        )

        if labels is None:
            return batch

        sequence_length = torch.tensor(batch["input_ids"]).shape[1]
        padding_side = self.tokenizer.padding_side
        if padding_side == "right":
            batch[label_name] = [
                list(label) + [self.label_pad_token_id] * (sequence_length - len(label)) for label in labels
            ]
        else:
            batch[label_name] = [
                [self.label_pad_token_id] * (sequence_length - len(label)) + list(label) for label in labels
            ]

        batch = {k: torch.tensor(v, dtype=torch.int64) for k, v in batch.items()}
        return batch
```

`DataCollatorForTokenClassification` 继承自 `DataCollatorMixin`, 没有重新定义 `__call__` 方法,
所以需要分别定义 `torch_call` 和 `tf_call` 和 `numpy_call`. 这里以 `torch_call` 为例来看看.

首先是取出了 labels, 然后和上面的一样使用 `self.tokenizer` 填充. 如果有标签, 还需要进一步处理, 因为标签的长度也要填充.

# DataCollatorForLanguageModeling

```python
@dataclass
class DataCollatorForLanguageModeling(DataCollatorMixin):
    tokenizer: PreTrainedTokenizerBase
    mlm: bool = True
    mlm_probability: float = 0.15
    pad_to_multiple_of: Optional[int] = None
    tf_experimental_compile: bool = False
    return_tensors: str = "pt"

    def __post_init__(self):
        if self.mlm and self.tokenizer.mask_token is None:
            raise ValueError(
                "This tokenizer does not have a mask token which is necessary for masked language modeling. "
                "You should pass `mlm=False` to train on causal language modeling instead."
            )
        if self.tf_experimental_compile:
            import tensorflow as tf

            self.tf_mask_tokens = tf.function(self.tf_mask_tokens, jit_compile=True)

    def torch_call(self, examples: List[Union[List[int], Any, Dict[str, Any]]]) -> Dict[str, Any]:
        # Handle dict or lists with proper padding and conversion to tensor.
        if isinstance(examples[0], (dict, BatchEncoding)):
            batch = self.tokenizer.pad(examples, return_tensors="pt", pad_to_multiple_of=self.pad_to_multiple_of)
        else:
            batch = {
                "input_ids": _torch_collate_batch(examples, self.tokenizer, pad_to_multiple_of=self.pad_to_multiple_of)
            }

        # If special token mask has been preprocessed, pop it from the dict.
        special_tokens_mask = batch.pop("special_tokens_mask", None)
        if self.mlm:
            batch["input_ids"], batch["labels"] = self.torch_mask_tokens(
                batch["input_ids"], special_tokens_mask=special_tokens_mask
            )
        else:
            labels = batch["input_ids"].clone()
            if self.tokenizer.pad_token_id is not None:
                labels[labels == self.tokenizer.pad_token_id] = -100
            batch["labels"] = labels
        return batch

    def torch_mask_tokens(self, inputs: Any, special_tokens_mask: Optional[Any] = None) -> Tuple[Any, Any]:
        """
        Prepare masked tokens inputs/labels for masked language modeling: 80% MASK, 10% random, 10% original.
        """
        import torch

        labels = inputs.clone()
        # We sample a few tokens in each sequence for MLM training (with probability `self.mlm_probability`)
        probability_matrix = torch.full(labels.shape, self.mlm_probability)
        if special_tokens_mask is None:
            special_tokens_mask = [
                self.tokenizer.get_special_tokens_mask(val, already_has_special_tokens=True) for val in labels.tolist()
            ]
            special_tokens_mask = torch.tensor(special_tokens_mask, dtype=torch.bool)
        else:
            special_tokens_mask = special_tokens_mask.bool()

        probability_matrix.masked_fill_(special_tokens_mask, value=0.0)
        masked_indices = torch.bernoulli(probability_matrix).bool()
        labels[~masked_indices] = -100  # We only compute loss on masked tokens

        # 80% of the time, we replace masked input tokens with tokenizer.mask_token ([MASK])
        indices_replaced = torch.bernoulli(torch.full(labels.shape, 0.8)).bool() & masked_indices
        inputs[indices_replaced] = self.tokenizer.convert_tokens_to_ids(self.tokenizer.mask_token)

        # 10% of the time, we replace masked input tokens with random word
        indices_random = torch.bernoulli(torch.full(labels.shape, 0.5)).bool() & masked_indices & ~indices_replaced
        random_words = torch.randint(len(self.tokenizer), labels.shape, dtype=torch.long)
        inputs[indices_random] = random_words[indices_random]

        # The rest of the time (10% of the time) we keep the masked input tokens unchanged
        return inputs, labels
```

原本想就到此为止的, 后来发现 `DataCollatorForLanguageModeling` 也是挺有趣的, 就继续看看吧.

`__post_init__` 是 dataclass 的一个后处理函数, 在 `__init__` 的最后被调用. 这里验证了下参数, 并且可选开启 tf 的 jit 编译.

因为又是继承自 `DataCollatorMixin`, 所以依旧是只看 `torch_call` 这一部分.

`torch_call` 里前面是构建 batch, 其中 `_torch_collate_batch` 会根据当前批次的最大长度进行填充, 填充到一样的长度.
然后如果开启了 `self.mlm`, 会使用 `self.torch_mask_tokens` 进行随机掩码.

在 `torch_mask_tokens` 中, 首先是构建标签 labels. 构建了一个概率矩阵 `probability_matrix`, 并将特殊 token 部分都设置为 0.
然后使用 `torch.bernoulli` 进行伯努利二进制采样, 得到 `masked_indices`的大概有 `self.mlm_probability` 的概率是 True.
最后使用反向操作, 将这些 labels 都变成 -100 了, 也就是这些都是无需预测标签的.

构建完标签后, 就使用开始对 `inputs` 进行随机掩码了. 有 80% 的概率替换成 `[MASK]`, 10% 的概率替换成词汇表中的随机单词, 10% 的概率保持不变.

# 小结

`Data Collator` 是在训练中动态处理数据的. 数据层面的已经看的差不多, 后续会进入到模型层面的.

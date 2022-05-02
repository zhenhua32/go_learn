[TOC]

# 简介

看完了配置类, 来看看分词器吧.

对应的官方文档是 [Tokenizer](https://huggingface.co/docs/transformers/main_classes/tokenizer).

# PreTrainedTokenizerBase

Tokenizer 有两类实现, `PreTrainedTokenizer` 和 `PreTrainedTokenizerFast`.
`PreTrainedTokenizer` 是完整的 python 实现.
`PreTrainedTokenizerFast` 是基于 Rust 写的, 原始库是 [tokenizers](https://github.com/huggingface/tokenizers).

这不管这些, 这两个实现都是基于父类 `PreTrainedTokenizerBase` 的, 所以先来看看这个.

```python
@add_end_docstrings(INIT_TOKENIZER_DOCSTRING)
class PreTrainedTokenizerBase(SpecialTokensMixin, PushToHubMixin):
    ...
```

本身继承了两个父类, `PushToHubMixin` 是和 hub 网站交互的代码, 一律跳过. 在看 `SpecialTokensMixin` 类之前, 先看看怎么拼接文档字符串的.

```python
def add_end_docstrings(*docstr):
    def docstring_decorator(fn):
        fn.__doc__ = (fn.__doc__ if fn.__doc__ is not None else "") + "".join(docstr)
        return fn

    return docstring_decorator
```

看过之后才发现很简单, `add_end_docstrings` 是个装饰器, 所以返回了一个函数. `__doc__` 属性是获取文档字符串的,
直接将传入的 `*docstr` 拼接起来, 在最前面加上函数本身的文档字符串就行了.
这个就是文档在多个函数内有重复的, 也不能一直重复自己, 所以搞了个装饰器拼接文档字符串.

```python
class SpecialTokensMixin:
    SPECIAL_TOKENS_ATTRIBUTES = [
        "bos_token",
        "eos_token",
        "unk_token",
        "sep_token",
        "pad_token",
        "cls_token",
        "mask_token",
        "additional_special_tokens",
    ]
```

`SpecialTokensMixin` 类维护着一堆特殊的 token, 从 `SPECIAL_TOKENS_ATTRIBUTES` 属性中可以看出有哪些类型.

先看看是怎么实例化的:

```python
def __init__(self, verbose=True, **kwargs):
    self._bos_token = None
    self._eos_token = None
    self._unk_token = None
    self._sep_token = None
    self._pad_token = None
    self._cls_token = None
    self._mask_token = None
    self._pad_token_type_id = 0
    self._additional_special_tokens = []
    self.verbose = verbose

    # We directly set the hidden value to allow initialization with special tokens
    # which are not yet in the vocabulary. Necessary for serialization/de-serialization
    # TODO clean this up at some point (probably by switching to fast tokenizers)
    for key, value in kwargs.items():
        if value is None:
            continue
        if key in self.SPECIAL_TOKENS_ATTRIBUTES:
            if key == "additional_special_tokens":
                assert isinstance(value, (list, tuple)), f"Value {value} is not a list or tuple"
                assert all(
                    isinstance(t, (str, AddedToken)) for t in value
                ), "One of the tokens is not a string or an AddedToken"
                setattr(self, key, value)
            elif isinstance(value, (str, AddedToken)):
                setattr(self, key, value)
            else:
                raise TypeError(f"special token {key} has to be either str or AddedToken but got: {type(value)}")
```

初始化很简单, 先设置了一堆 `_` 开头的属性, 属性名和 `SPECIAL_TOKENS_ATTRIBUTES` 中的一样. 然后使用别的关键字参数初始化属性,
要求这些关键词的参数都在 `SPECIAL_TOKENS_ATTRIBUTES` 中, 且判断了参数值的类型. `additional_special_tokens` 是个列表或元组,
`additional_special_tokens` 中每个项的值和其他的参数值都要求是 `str` 或 `AddedToken` 的实例.

顺带看一下 `AddedToken` 的样子, 主要是 `content` 这个属性, 表示 token 所代表的字符串. 还有一些别的属性来指定这个 token 的行为.

```python
@dataclass(frozen=True, eq=True)
class AddedToken:
    """
    AddedToken represents a token to be added to a Tokenizer An AddedToken can have special options defining the
    way it should behave.
    """

    content: str = field(default_factory=str)
    single_word: bool = False
    lstrip: bool = False
    rstrip: bool = False
    normalized: bool = True

    def __getstate__(self):
        return self.__dict__
```

`add_tokens` 是用来添加 token 的, 还没实现.

```python
def add_tokens(
    self, new_tokens: Union[str, AddedToken, List[Union[str, AddedToken]]], special_tokens: bool = False
) -> int:
    if not new_tokens:
        return 0

    if not isinstance(new_tokens, (list, tuple)):
        new_tokens = [new_tokens]

    return self._add_tokens(new_tokens, special_tokens=special_tokens)

def _add_tokens(self, new_tokens: Union[List[str], List[AddedToken]], special_tokens: bool = False) -> int:
    raise NotImplementedError
```

另外有两个和特殊 token 相关的方法, 本质上都是调用 `add_tokens`, `add_special_tokens` 里面多了一些对类型的断言.

```python
def sanitize_special_tokens(self) -> int:
    return self.add_tokens(self.all_special_tokens_extended, special_tokens=True)

def add_special_tokens(self, special_tokens_dict: Dict[str, Union[str, AddedToken]]) -> int:
    # 空的就跳过, 返回计数为 0
    if not special_tokens_dict:
        return 0

    added_tokens = 0
    for key, value in special_tokens_dict.items():
        # 要求 key 在 SPECIAL_TOKENS_ATTRIBUTES 中
        assert key in self.SPECIAL_TOKENS_ATTRIBUTES, f"Key {key} is not a special token"

        # verbose 会记录这个添加的过程
        if self.verbose:
            logger.info(f"Assigning {value} to the {key} key of the tokenizer")
        # 居然是直接将 key 添加为属性
        setattr(self, key, value)

        if key == "additional_special_tokens":
            # 又判断了一遍类型
            assert isinstance(value, (list, tuple)) and all(
                isinstance(t, (str, AddedToken)) for t in value
            ), f"Tokens {value} for key {key} should all be str or AddedToken instances"
            # 实际操作, 添加一个 token
            added_tokens += self.add_tokens(value, special_tokens=True)
        else:
            # 判断类型
            assert isinstance(
                value, (str, AddedToken)
            ), f"Token {value} for key {key} should be a str or an AddedToken instance"
            added_tokens += self.add_tokens([value], special_tokens=True)

    return added_tokens
```

看过这些方法之后, 再来看看每个特殊属性都有 getter 和 setter.

```python
@property
def bos_token(self) -> str:
    """
    `str`: Beginning of sentence token. Log an error if used while not having been set.
    """
    if self._bos_token is None and self.verbose:
        logger.error("Using bos_token, but it is not set yet.")
        return None
    return str(self._bos_token)

@bos_token.setter
def bos_token(self, value):
    self._bos_token = value
```

所有特殊字符的 getter 和 setter 都类似, getter 中会判断 `self.verbose`, 用于输出 error 级别的信息.

```python
@property
def bos_token_id(self) -> Optional[int]:
    """
    `Optional[int]`: Id of the beginning of sentence token in the vocabulary. Returns `None` if the token has not
    been set.
    """
    if self._bos_token is None:
        return None
    return self.convert_tokens_to_ids(self.bos_token)

@bos_token_id.setter
def bos_token_id(self, value):
    self._bos_token = self.convert_ids_to_tokens(value) if value is not None else None
```

除了值的 getter 和 setter 之外, 还有 id 的. `convert_tokens_to_ids` 和 `bos_token_id` 没有在这个类中提起, 是子类应该实现的方法.

除了单个的特殊 token 之外, 所有的特殊 token 和 id 也有一些属性, 没啥特别的, 这里也就不多说了. 接下来应该回过头来继续看 `PreTrainedTokenizerBase`.

```python
@add_end_docstrings(INIT_TOKENIZER_DOCSTRING)
class PreTrainedTokenizerBase(SpecialTokensMixin, PushToHubMixin):
    vocab_files_names: Dict[str, str] = {}
    pretrained_vocab_files_map: Dict[str, Dict[str, str]] = {}
    pretrained_init_configuration: Dict[str, Dict[str, Any]] = {} # 模型的最大输入长度
    max_model_input_sizes: Dict[str, Optional[int]] = {}
    _auto_class: Optional[str] = None

    # 模型输入的名字列表
    # first name has to correspond to main model input name
    # to make sure `tokenizer.pad(...)` works correctly
    model_input_names: List[str] = ["input_ids", "token_type_ids", "attention_mask"]
    # padding 方向, 在右边或左边
    padding_side: str = "right"
    # 截断方法, 在右边或左边
    truncation_side: str = "right"
    slow_tokenizer_class = None

    def __init__(self, **kwargs):
        # inputs and kwargs for saving and re-loading (see ``from_pretrained`` and ``save_pretrained``)
        self.init_inputs = ()
        self.init_kwargs = copy.deepcopy(kwargs)
        # 名字或路径
        self.name_or_path = kwargs.pop("name_or_path", "")
        # 处理器类
        self._processor_class = kwargs.pop("processor_class", None)

        # For backward compatibility we fallback to set model_max_length from max_len if provided
        model_max_length = kwargs.pop("model_max_length", kwargs.pop("max_len", None))
        self.model_max_length = model_max_length if model_max_length is not None else VERY_LARGE_INTEGER

        # Padding and truncation side are right by default and overridden in subclasses. If specified in the kwargs, it
        # is changed.
        self.padding_side = kwargs.pop("padding_side", self.padding_side)
        if self.padding_side not in ["right", "left"]:
            raise ValueError(
                f"Padding side should be selected between 'right' and 'left', current value: {self.padding_side}"
            )

        self.truncation_side = kwargs.pop("truncation_side", self.truncation_side)
        if self.truncation_side not in ["right", "left"]:
            raise ValueError(
                f"Padding side should be selected between 'right' and 'left', current value: {self.truncation_side}"
            )

        self.model_input_names = kwargs.pop("model_input_names", self.model_input_names)

        self.deprecation_warnings = (
            {}
        )  # Use to store when we have already noticed a deprecation warning (avoid overlogging).

        super().__init__(**kwargs)
```

首先来看看类的属性和初始化方法. 没有太特殊的地方, 只是对某些属性值进行了限制.

`from_pretrained` 和 `save_pretrained` 是两个主要的方法, 一个用于加载, 一个用于保存.
因为太长了, 这两个都有一个加下划线的方法, `_from_pretrained` 和 `_save_pretrained`.

`tokenize` 方法将字符串转换成 token 的列表. `encode` 将字符串转换为 id 的列表. 这两个方法都没实现.

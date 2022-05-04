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
太长也比较无聊, 就不详细描述了.

`tokenize` 方法将字符串转换成 token 的列表. `encode` 将字符串转换为 id 的列表. 这两个方法都没实现. `encode` 内部调用了 `encode_plus` 方法.

```python
@add_end_docstrings(ENCODE_KWARGS_DOCSTRING, ENCODE_PLUS_ADDITIONAL_KWARGS_DOCSTRING)
def encode_plus(
    self,
    text: Union[TextInput, PreTokenizedInput, EncodedInput],
    text_pair: Optional[Union[TextInput, PreTokenizedInput, EncodedInput]] = None,
    add_special_tokens: bool = True,
    padding: Union[bool, str, PaddingStrategy] = False,
    truncation: Union[bool, str, TruncationStrategy] = False,
    max_length: Optional[int] = None,
    stride: int = 0,
    is_split_into_words: bool = False,
    pad_to_multiple_of: Optional[int] = None,
    return_tensors: Optional[Union[str, TensorType]] = None,
    return_token_type_ids: Optional[bool] = None,
    return_attention_mask: Optional[bool] = None,
    return_overflowing_tokens: bool = False,
    return_special_tokens_mask: bool = False,
    return_offsets_mapping: bool = False,
    return_length: bool = False,
    verbose: bool = True,
    **kwargs
) -> BatchEncoding:
    # Backward compatibility for 'truncation_strategy', 'pad_to_max_length'
    padding_strategy, truncation_strategy, max_length, kwargs = self._get_padding_truncation_strategies(
        padding=padding,
        truncation=truncation,
        max_length=max_length,
        pad_to_multiple_of=pad_to_multiple_of,
        verbose=verbose,
        **kwargs,
    )

    return self._encode_plus(
        text=text,
        text_pair=text_pair,
        add_special_tokens=add_special_tokens,
        padding_strategy=padding_strategy,
        truncation_strategy=truncation_strategy,
        max_length=max_length,
        stride=stride,
        is_split_into_words=is_split_into_words,
        pad_to_multiple_of=pad_to_multiple_of,
        return_tensors=return_tensors,
        return_token_type_ids=return_token_type_ids,
        return_attention_mask=return_attention_mask,
        return_overflowing_tokens=return_overflowing_tokens,
        return_special_tokens_mask=return_special_tokens_mask,
        return_offsets_mapping=return_offsets_mapping,
        return_length=return_length,
        verbose=verbose,
        **kwargs,
    )
```

`encode_plus` 里面有个 padding 和 truncation 的策略函数, 来看看这个 `_get_padding_truncation_strategies`.

```python
def _get_padding_truncation_strategies(
    self, padding=False, truncation=False, max_length=None, pad_to_multiple_of=None, verbose=True, **kwargs
):
    """
    获取填充或截断的策略
    Find the correct padding/truncation strategy with backward compatibility for old arguments (truncation_strategy
    and pad_to_max_length) and behaviors.
    """
    old_truncation_strategy = kwargs.pop("truncation_strategy", "do_not_truncate")
    old_pad_to_max_length = kwargs.pop("pad_to_max_length", False)

    # Backward compatibility for previous behavior, maybe we should deprecate it:
    # If you only set max_length, it activates truncation for max_length
    if max_length is not None and padding is False and truncation is False:
        if verbose:
            if not self.deprecation_warnings.get("Truncation-not-explicitly-activated", False):
                logger.warning(
                    "Truncation was not explicitly activated but `max_length` is provided a specific value, "
                    "please use `truncation=True` to explicitly truncate examples to max length. "
                    "Defaulting to 'longest_first' truncation strategy. "
                    "If you encode pairs of sequences (GLUE-style) with the tokenizer you can select this strategy "
                    "more precisely by providing a specific strategy to `truncation`."
                )
            self.deprecation_warnings["Truncation-not-explicitly-activated"] = True
        truncation = "longest_first"

    # Get padding strategy
    if padding is False and old_pad_to_max_length:
        if verbose:
            warnings.warn(
                "The `pad_to_max_length` argument is deprecated and will be removed in a future version, "
                "use `padding=True` or `padding='longest'` to pad to the longest sequence in the batch, or "
                "use `padding='max_length'` to pad to a max length. In this case, you can give a specific "
                "length with `max_length` (e.g. `max_length=45`) or leave max_length to None to pad to the "
                "maximal input size of the model (e.g. 512 for Bert).",
                FutureWarning,
            )
        if max_length is None:
            padding_strategy = PaddingStrategy.LONGEST
        else:
            padding_strategy = PaddingStrategy.MAX_LENGTH
    elif padding is not False:
        if padding is True:
            if verbose:
                if max_length is not None and (truncation is False or truncation == "do_not_truncate"):
                    warnings.warn(
                        "`max_length` is ignored when `padding`=`True` and there is no truncation strategy. "
                        "To pad to max length, use `padding='max_length'`."
                    )
                if old_pad_to_max_length is not False:
                    warnings.warn("Though `pad_to_max_length` = `True`, it is ignored because `padding`=`True`.")
            padding_strategy = PaddingStrategy.LONGEST  # Default to pad to the longest sequence in the batch
        elif not isinstance(padding, PaddingStrategy):
            padding_strategy = PaddingStrategy(padding)
        elif isinstance(padding, PaddingStrategy):
            padding_strategy = padding
    else:
        padding_strategy = PaddingStrategy.DO_NOT_PAD

    # Get truncation strategy
    if truncation is False and old_truncation_strategy != "do_not_truncate":
        if verbose:
            warnings.warn(
                "The `truncation_strategy` argument is deprecated and will be removed in a future version, "
                "use `truncation=True` to truncate examples to a max length. You can give a specific "
                "length with `max_length` (e.g. `max_length=45`) or leave max_length to None to truncate to the "
                "maximal input size of the model (e.g. 512 for Bert). "
                " If you have pairs of inputs, you can give a specific truncation strategy selected among "
                "`truncation='only_first'` (will only truncate the first sentence in the pairs) "
                "`truncation='only_second'` (will only truncate the second sentence in the pairs) "
                "or `truncation='longest_first'` (will iteratively remove tokens from the longest sentence in the pairs).",
                FutureWarning,
            )
        truncation_strategy = TruncationStrategy(old_truncation_strategy)
    elif truncation is not False:
        if truncation is True:
            truncation_strategy = (
                TruncationStrategy.LONGEST_FIRST
            )  # Default to truncate the longest sequences in pairs of inputs
        elif not isinstance(truncation, TruncationStrategy):
            truncation_strategy = TruncationStrategy(truncation)
        elif isinstance(truncation, TruncationStrategy):
            truncation_strategy = truncation
    else:
        truncation_strategy = TruncationStrategy.DO_NOT_TRUNCATE

    # Set max length if needed
    if max_length is None:
        if padding_strategy == PaddingStrategy.MAX_LENGTH:
            if self.model_max_length > LARGE_INTEGER:
                if verbose:
                    if not self.deprecation_warnings.get("Asking-to-pad-to-max_length", False):
                        logger.warning(
                            "Asking to pad to max_length but no maximum length is provided and the model has no predefined maximum length. "
                            "Default to no padding."
                        )
                    self.deprecation_warnings["Asking-to-pad-to-max_length"] = True
                padding_strategy = PaddingStrategy.DO_NOT_PAD
            else:
                max_length = self.model_max_length

        if truncation_strategy != TruncationStrategy.DO_NOT_TRUNCATE:
            if self.model_max_length > LARGE_INTEGER:
                if verbose:
                    if not self.deprecation_warnings.get("Asking-to-truncate-to-max_length", False):
                        logger.warning(
                            "Asking to truncate to max_length but no maximum length is provided and the model has no predefined maximum length. "
                            "Default to no truncation."
                        )
                    self.deprecation_warnings["Asking-to-truncate-to-max_length"] = True
                truncation_strategy = TruncationStrategy.DO_NOT_TRUNCATE
            else:
                max_length = self.model_max_length

    # Test if we have a padding token
    if padding_strategy != PaddingStrategy.DO_NOT_PAD and (not self.pad_token or self.pad_token_id < 0):
        raise ValueError(
            "Asking to pad but the tokenizer does not have a padding token. "
            "Please select a token to use as `pad_token` `(tokenizer.pad_token = tokenizer.eos_token e.g.)` "
            "or add a new pad token via `tokenizer.add_special_tokens({'pad_token': '[PAD]'})`."
        )

    # Check that we will truncate to a multiple of pad_to_multiple_of if both are provided
    if (
        truncation_strategy != TruncationStrategy.DO_NOT_TRUNCATE
        and padding_strategy != PaddingStrategy.DO_NOT_PAD
        and pad_to_multiple_of is not None
        and max_length is not None
        and (max_length % pad_to_multiple_of != 0)
    ):
        raise ValueError(
            f"Truncation and padding are both activated but "
            f"truncation length ({max_length}) is not a multiple of pad_to_multiple_of ({pad_to_multiple_of})."
        )

    return padding_strategy, truncation_strategy, max_length, kwargs
```

这里面就是检查了 `padding_strategy, truncation_strategy, max_length` 这三个变量的合理性.
`pad_to_multiple_of` 如果设置了, 就要求 `max_length` 是 `pad_to_multiple_of` 的倍数.

另一点是都使用了枚举类, 然后因为参数可以是 bool 类型, 所以对 True 和 False 两个值都做了转换, 意思是接近的.
True 表示填充或截取到当前 batch 最长的那个序列的长度. False 表示不进行填充或截取.

`encode_plus` 还有一个批量的方法 `batch_encode_plus`, 当然也是没实现, 先不用关注了.

`decode` 是 `encode` 的相反操作, 将 id 的列表转换为字符串. 同样的, 也有批处理操作, `batch_decode`.
`batch_decode` 倒是实现了, 就是对序列中的每个元素调用了 `decode` 方法. 看来是没有优化的必要, 也就简单实现了.

再来看看核心的 `__call__` 方法, 用于直接将 tokenize 实例直接作为函数使用.

```python
@add_end_docstrings(ENCODE_KWARGS_DOCSTRING, ENCODE_PLUS_ADDITIONAL_KWARGS_DOCSTRING)
def __call__(
    self,
    text: Union[TextInput, PreTokenizedInput, List[TextInput], List[PreTokenizedInput]],
    text_pair: Optional[Union[TextInput, PreTokenizedInput, List[TextInput], List[PreTokenizedInput]]] = None,
    add_special_tokens: bool = True,
    padding: Union[bool, str, PaddingStrategy] = False,
    truncation: Union[bool, str, TruncationStrategy] = False,
    max_length: Optional[int] = None,
    stride: int = 0,
    is_split_into_words: bool = False,
    pad_to_multiple_of: Optional[int] = None,
    return_tensors: Optional[Union[str, TensorType]] = None,
    return_token_type_ids: Optional[bool] = None,
    return_attention_mask: Optional[bool] = None,
    return_overflowing_tokens: bool = False,
    return_special_tokens_mask: bool = False,
    return_offsets_mapping: bool = False,
    return_length: bool = False,
    verbose: bool = True,
    **kwargs
) -> BatchEncoding:
    # 检查输入的类型
    # Input type checking for clearer error
    def _is_valid_text_input(t):
        if isinstance(t, str):
            # Strings are fine
            return True
        elif isinstance(t, (list, tuple)):
            # 都是只检查第一个元素
            # List are fine as long as they are...
            if len(t) == 0:
                # ... empty
                return True
            elif isinstance(t[0], str):
                # ... list of strings
                return True
            elif isinstance(t[0], (list, tuple)):
                # 还能嵌套, 同样只检查第一个元素
                # ... list with an empty list or with a list of strings
                return len(t[0]) == 0 or isinstance(t[0][0], str)
            else:
                return False
        else:
            return False

    if not _is_valid_text_input(text):
        raise ValueError(
            "text input must of type `str` (single example), `List[str]` (batch or single pretokenized example) "
            "or `List[List[str]]` (batch of pretokenized examples)."
        )

    if text_pair is not None and not _is_valid_text_input(text_pair):
        raise ValueError(
            "text input must of type `str` (single example), `List[str]` (batch or single pretokenized example) "
            "or `List[List[str]]` (batch of pretokenized examples)."
        )

    # 输入是否已经预先切成单词了
    if is_split_into_words:
        # 判断是否是批次处理. 批次处理应该是 列表的列表
        is_batched = isinstance(text, (list, tuple)) and text and isinstance(text[0], (list, tuple))
    else:
        # 批次处理应该是 列表
        is_batched = isinstance(text, (list, tuple))

    if is_batched:
        if isinstance(text_pair, str):
            raise TypeError(
                "when tokenizing batches of text, `text_pair` must be a list or tuple with the same length as `text`."
            )
        if text_pair is not None and len(text) != len(text_pair):
            raise ValueError(
                f"batch length of `text`: {len(text)} does not match batch length of `text_pair`: {len(text_pair)}."
            )
        # 如果 text_pair 存在, 就合并 text 转换为元组的数组. text_pair 就是第二个序列
        batch_text_or_text_pairs = list(zip(text, text_pair)) if text_pair is not None else text
        # 批次处理调用 batch_encode_plus
        return self.batch_encode_plus(
            batch_text_or_text_pairs=batch_text_or_text_pairs,
            add_special_tokens=add_special_tokens,
            padding=padding,
            truncation=truncation,
            max_length=max_length,
            stride=stride,
            is_split_into_words=is_split_into_words,
            pad_to_multiple_of=pad_to_multiple_of,
            return_tensors=return_tensors,
            return_token_type_ids=return_token_type_ids,
            return_attention_mask=return_attention_mask,
            return_overflowing_tokens=return_overflowing_tokens,
            return_special_tokens_mask=return_special_tokens_mask,
            return_offsets_mapping=return_offsets_mapping,
            return_length=return_length,
            verbose=verbose,
            **kwargs,
        )
    else:
        # 如果不是批次处理, 也是调用 encode_plus
        return self.encode_plus(
            text=text,
            text_pair=text_pair,
            add_special_tokens=add_special_tokens,
            padding=padding,
            truncation=truncation,
            max_length=max_length,
            stride=stride,
            is_split_into_words=is_split_into_words,
            pad_to_multiple_of=pad_to_multiple_of,
            return_tensors=return_tensors,
            return_token_type_ids=return_token_type_ids,
            return_attention_mask=return_attention_mask,
            return_overflowing_tokens=return_overflowing_tokens,
            return_special_tokens_mask=return_special_tokens_mask,
            return_offsets_mapping=return_offsets_mapping,
            return_length=return_length,
            verbose=verbose,
            **kwargs,
        )
```

代码不多, 看起来长是因为参数多. 核心思想是先验证输入的类型是否正确, 然后判断下是否是批次输入. 如果是批次输入, 就调用 `batch_encode_plus`.
否则就调用 `encode_plus` 方法.

看了这么多的未实现方法, 很想看看子类是怎么实现的. 但在这之前, 先看看填充和截断是怎么实现的.

# PreTrainedTokenizerFast

# PreTrainedTokenizer

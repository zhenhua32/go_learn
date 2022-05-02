[TOC]

# 简介

前面刚看了 `__init__` 文件, 继续从基础看起, 先研究基础配置和通用模块.

对应的官方文档是 [Configuration](https://huggingface.co/docs/transformers/main_classes/configuration).

# PretrainedConfig

`PretrainedConfig` 类定义了预训练参数, 代码可以和文档结合看, 可以互相对照,
[文档地址](https://huggingface.co/docs/transformers/main_classes/configuration).

`__init__` 方法中设置了一大推属性, 也没有什么特别的, 可以直接跳过.

```python
@property
def num_labels(self) -> int:
    """
    `int`: The number of labels for classification models.
    """
    # 实际上用了 id2label, 而不是设置的 num_labels 属性
    return len(self.id2label)

@num_labels.setter
def num_labels(self, num_labels: int):
    # 如果没有 id2label 或者长度不一致, 就直接更新, 用的是固定结构的 label 名字, 从 0 开始的 LABEL_0
    if not hasattr(self, "id2label") or self.id2label is None or len(self.id2label) != num_labels:
        self.id2label = {i: f"LABEL_{i}" for i in range(num_labels)}
        self.label2id = dict(zip(self.id2label.values(), self.id2label.keys()))
```

`num_labels` 这个属性有点意思, 实际上的长度取决于 `self.id2label`. 然后如果设置了值, 那么也会直接更新 `self.id2label`.

方法中主要以 `save_pretrained` 和 `from_pretrained`, 分别用于保存和加载配置文件.

```python
def save_pretrained(self, save_directory: Union[str, os.PathLike], push_to_hub: bool = False, **kwargs):
    if os.path.isfile(save_directory):
        raise AssertionError(f"Provided path ({save_directory}) should be a directory, not a file")

    if push_to_hub:
        commit_message = kwargs.pop("commit_message", None)
        repo = self._create_or_get_repo(save_directory, **kwargs)

    os.makedirs(save_directory, exist_ok=True)

    # If we have a custom config, we copy the file defining it in the folder and set the attributes so it can be
    # loaded from the Hub.
    if self._auto_class is not None:
        custom_object_save(self, save_directory, config=self)

    # If we save using the predefined names, we can load using `from_pretrained`
    output_config_file = os.path.join(save_directory, CONFIG_NAME)

    # 保存配置文件
    self.to_json_file(output_config_file, use_diff=True)
    logger.info(f"Configuration saved in {output_config_file}")

    if push_to_hub:
        url = self._push_to_hub(repo, commit_message=commit_message)
        logger.info(f"Configuration pushed to the hub in this commit: {url}")
```

忽略大部分语句, 核心就是使用 `self.to_json_file(output_config_file, use_diff=True)` 将当前配置保存成 json 文件.

对应的, 载入配置文件也很简单:

```python
@classmethod
def from_pretrained(cls, pretrained_model_name_or_path: Union[str, os.PathLike], **kwargs) -> "PretrainedConfig":
    config_dict, kwargs = cls.get_config_dict(pretrained_model_name_or_path, **kwargs)
    # model_type 冲突时提示
    if "model_type" in config_dict and hasattr(cls, "model_type") and config_dict["model_type"] != cls.model_type:
        logger.warning(
            f"You are using a model of type {config_dict['model_type']} to instantiate a model of type "
            f"{cls.model_type}. This is not supported for all configurations of models and can yield errors."
        )

    return cls.from_dict(config_dict, **kwargs)
```

核心就是从配置文件中读取成 dict, 然后再将 dict 加载进来.

# AutoConfig

为了解决配置类太多的问题, 基本就是每个模型都需要一个特定的配置类, 所以有了 AutoConfig. 一起来看看吧.

AutoConfig 类是不能直接被实例化的, 而是通过类方法 `from_pretrained` 使用的, 先来看看这个类方法.

```python
@classmethod
@replace_list_option_in_docstrings()
def from_pretrained(cls, pretrained_model_name_or_path, **kwargs):
    kwargs["_from_auto"] = True
    kwargs["name_or_path"] = pretrained_model_name_or_path
    trust_remote_code = kwargs.pop("trust_remote_code", False)
    config_dict, _ = PretrainedConfig.get_config_dict(pretrained_model_name_or_path, **kwargs)
    if "auto_map" in config_dict and "AutoConfig" in config_dict["auto_map"]:
        if not trust_remote_code:
            raise ValueError(
                f"Loading {pretrained_model_name_or_path} requires you to execute the configuration file in that repo "
                "on your local machine. Make sure you have read the code there to avoid malicious use, then set "
                "the option `trust_remote_code=True` to remove this error."
            )
        if kwargs.get("revision", None) is None:
            logger.warning(
                "Explicitly passing a `revision` is encouraged when loading a configuration with custom code to "
                "ensure no malicious code has been contributed in a newer revision."
            )
        class_ref = config_dict["auto_map"]["AutoConfig"]
        module_file, class_name = class_ref.split(".")
        config_class = get_class_from_dynamic_module(
            pretrained_model_name_or_path, module_file + ".py", class_name, **kwargs
        )
        return config_class.from_pretrained(pretrained_model_name_or_path, **kwargs)
    elif "model_type" in config_dict:
        config_class = CONFIG_MAPPING[config_dict["model_type"]]
        return config_class.from_dict(config_dict, **kwargs)
    else:
        # Fallback: use pattern matching on the string.
        for pattern, config_class in CONFIG_MAPPING.items():
            if pattern in str(pretrained_model_name_or_path):
                return config_class.from_dict(config_dict, **kwargs)

    raise ValueError(
        f"Unrecognized model in {pretrained_model_name_or_path}. "
        f"Should have a `model_type` key in its {CONFIG_NAME}, or contain one of the following strings "
        f"in its name: {', '.join(CONFIG_MAPPING.keys())}"
    )
```

先通过 `PretrainedConfig` 获取了 `config_dict`, 然后根据情况, 尝试了三种方法读取 `config_class`.
第一种方式中获取的 `class_ref` 只是个字符串, 所以使用了动态加模块的函数 `get_class_from_dynamic_module`.
后面两种都是从 `CONFIG_MAPPING` 中获取的, 已经是加载后的模块了, 所以需要动态加载.

顺道看看如何动态加载模块吧, 也许以后自己也能用到.

```python
def get_class_from_dynamic_module(
    pretrained_model_name_or_path: Union[str, os.PathLike],
    module_file: str,
    class_name: str,
    cache_dir: Optional[Union[str, os.PathLike]] = None,
    force_download: bool = False,
    resume_download: bool = False,
    proxies: Optional[Dict[str, str]] = None,
    use_auth_token: Optional[Union[bool, str]] = None,
    revision: Optional[str] = None,
    local_files_only: bool = False,
    **kwargs,
):
    # And lastly we get the class inside our newly created module
    final_module = get_cached_module_file(
        pretrained_model_name_or_path,
        module_file,
        cache_dir=cache_dir,
        force_download=force_download,
        resume_download=resume_download,
        proxies=proxies,
        use_auth_token=use_auth_token,
        revision=revision,
        local_files_only=local_files_only,
    )
    return get_class_in_module(class_name, final_module.replace(".py", ""))
```

忽略前面找寻模块的函数, 来看看 `get_class_in_module`.

```python
def get_class_in_module(class_name, module_path):
    """
    Import a module on the cache directory for modules and extract a class from it.
    """
    module_path = module_path.replace(os.path.sep, ".")
    module = importlib.import_module(module_path)
    return getattr(module, class_name)
```

代码不多, 就三行, 就是使用了 `importlib.import_module` 加载模块, 最后通过类名获取了类.

看完如何使用后, 再来看看如何注册.

```python
@staticmethod
def register(model_type, config):
    if issubclass(config, PretrainedConfig) and config.model_type != model_type:
        raise ValueError(
            "The config you are passing has a `model_type` attribute that is not consistent with the model type "
            f"you passed (config has {config.model_type} and you passed {model_type}. Fix one of those so they "
            "match!"
        )
    CONFIG_MAPPING.register(model_type, config)
```

主要是使用 `CONFIG_MAPPING`, 前面也涉及到了这个变量, 看看是如何初始化的.

```python
CONFIG_MAPPING = _LazyConfigMapping(CONFIG_MAPPING_NAMES)

CONFIG_MAPPING_NAMES = OrderedDict(
    [
        # Add configs here
        ("tapex", "BartConfig"),
        ("dpt", "DPTConfig"),
        ...
    ]
)
```

`CONFIG_MAPPING` 是作为全局变量初始化的, 它的参数 `CONFIG_MAPPING_NAMES` 是一个有序字典, key 是配置名字, 也就是 model_type, value 是配置类的字符串形式.

再来看看 `_LazyConfigMapping` 是什么, 以及是有哪些作用.

```python
class _LazyConfigMapping(OrderedDict):
    """
    A dictionary that lazily load its values when they are requested.
    """

    def __init__(self, mapping):
        self._mapping = mapping
        self._extra_content = {}
        self._modules = {}

    def __getitem__(self, key):
        if key in self._extra_content:
            return self._extra_content[key]
        if key not in self._mapping:
            raise KeyError(key)
        value = self._mapping[key]
        module_name = model_type_to_module_name(key)
        if module_name not in self._modules:
            self._modules[module_name] = importlib.import_module(f".{module_name}", "transformers.models")
        if hasattr(self._modules[module_name], value):
            return getattr(self._modules[module_name], value)

        # Some of the mappings have entries model_type -> config of another model type. In that case we try to grab the
        # object at the top level.
        transformers_module = importlib.import_module("transformers")
        return getattr(transformers_module, value)

    def keys(self):
        return list(self._mapping.keys()) + list(self._extra_content.keys())

    def values(self):
        return [self[k] for k in self._mapping.keys()] + list(self._extra_content.values())

    def items(self):
        return [(k, self[k]) for k in self._mapping.keys()] + list(self._extra_content.items())

    def __iter__(self):
        return iter(list(self._mapping.keys()) + list(self._extra_content.keys()))

    def __contains__(self, item):
        return item in self._mapping or item in self._extra_content

    def register(self, key, value):
        """
        Register a new configuration in this mapping.
        """
        if key in self._mapping.keys():
            raise ValueError(f"'{key}' is already used by a Transformers config, pick another name.")
        self._extra_content[key] = value
```

`_LazyConfigMapping` 里面主要有三个变量, `_mapping` 保存 transformers 原生的配置类.
自己注册的配置类放在 `_extra_content`, 也就是对应 `register` 方法.
`_modules` 用于缓存已经加载过的模块.

核心方法就是 `__getitem__`, 优先从 `_extra_content` 中获取, `_extra_content` 里保存的不是字符串, 而是已经加载的配置类.
`_mapping` 中保存的是字符串, 所以需要一系列的转换和加载模块, 然后再从模块中获取配置类.
`_modules` 就是缓存 `_mapping` 中已经加载过的配置类.

# BertConfig

最后, 还是来具体看看一个模型的配置类吧. 以后的大部分模型, 也是会基于 Bert 讲解.

```python
class BertConfig(PretrainedConfig):
    model_type = "bert"

    def __init__(
        self,
        vocab_size=30522,
        hidden_size=768,
        num_hidden_layers=12,
        num_attention_heads=12,
        intermediate_size=3072,
        hidden_act="gelu",
        hidden_dropout_prob=0.1,
        attention_probs_dropout_prob=0.1,
        max_position_embeddings=512,
        type_vocab_size=2,
        initializer_range=0.02,
        layer_norm_eps=1e-12,
        pad_token_id=0,
        position_embedding_type="absolute",
        use_cache=True,
        classifier_dropout=None,
        **kwargs
    ):
        super().__init__(pad_token_id=pad_token_id, **kwargs)

        self.vocab_size = vocab_size
        self.hidden_size = hidden_size
        self.num_hidden_layers = num_hidden_layers
        self.num_attention_heads = num_attention_heads
        self.hidden_act = hidden_act
        self.intermediate_size = intermediate_size
        self.hidden_dropout_prob = hidden_dropout_prob
        self.attention_probs_dropout_prob = attention_probs_dropout_prob
        self.max_position_embeddings = max_position_embeddings
        self.type_vocab_size = type_vocab_size
        self.initializer_range = initializer_range
        self.layer_norm_eps = layer_norm_eps
        self.position_embedding_type = position_embedding_type
        self.use_cache = use_cache
        self.classifier_dropout = classifier_dropout
```

看起来挺无趣的吧, 只是声明了类变量 `model_type` 为 `bert`. 然后加了一大推参数, 扩充了 `__init__` 方法罢了.

# 小结

如此, 我们就看完了两个核心的配置类了. `PretrainedConfig` 是基础的配置类, 所有的模块都会继承自它, 然而因为模型很多,
为了方便, 又诞生了 `AutoConfig` 类, 可以基于 `model_type` 自动获取对应的配置类.

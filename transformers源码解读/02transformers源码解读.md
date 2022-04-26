[TOC]

# 简介

前面刚看了 `__init__` 文件, 继续从基础看起, 先研究基础配置和通用模块.

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

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

最后看一下如何保存模型和加载模型.

```python
def save_pretrained(
    self,
    save_directory: Union[str, os.PathLike],
    save_config: bool = True,
    state_dict: Optional[dict] = None,
    save_function: Callable = torch.save,
    push_to_hub: bool = False,
    max_shard_size: Union[int, str] = "10GB",
    **kwargs,
):
    if os.path.isfile(save_directory):
        logger.error(f"Provided path ({save_directory}) should be a directory, not a file")
        return

    if push_to_hub:
        commit_message = kwargs.pop("commit_message", None)
        repo = self._create_or_get_repo(save_directory, **kwargs)

    os.makedirs(save_directory, exist_ok=True)

    model_to_save = unwrap_model(self)

    dtype = get_parameter_dtype(model_to_save)
    model_to_save.config.torch_dtype = str(dtype).split(".")[1]

    model_to_save.config.architectures = [model_to_save.__class__.__name__]

    if self._auto_class is not None:
        custom_object_save(self, save_directory, config=self.config)

    if save_config:
        model_to_save.config.save_pretrained(save_directory)

    if state_dict is None:
        state_dict = model_to_save.state_dict()

    if self._keys_to_ignore_on_save is not None:
        for ignore_key in self._keys_to_ignore_on_save:
            if ignore_key in state_dict.keys():
                del state_dict[ignore_key]

    shards, index = shard_checkpoint(state_dict, max_shard_size=max_shard_size)

    for filename in os.listdir(save_directory):
        full_filename = os.path.join(save_directory, filename)
        if filename.startswith(WEIGHTS_NAME[:-4]) and os.path.isfile(full_filename):
            os.remove(full_filename)

    for shard_file, shard in shards.items():
        save_function(shard, os.path.join(save_directory, shard_file))

    if index is None:
        logger.info(f"Model weights saved in {os.path.join(save_directory, WEIGHTS_NAME)}")
    else:
        save_index_file = os.path.join(save_directory, WEIGHTS_INDEX_NAME)
        with open(save_index_file, "w", encoding="utf-8") as f:
            content = json.dumps(index, indent=2, sort_keys=True) + "\n"
            f.write(content)
        logger.info(
            f"The model is bigger than the maximum size per checkpoint ({max_shard_size}) and is going to be "
            f"split in {len(shards)} checkpoint shards. You can find where each parameters has been saved in the "
            f"index located at {save_index_file}."
        )

    if push_to_hub:
        url = self._push_to_hub(repo, commit_message=commit_message)
        logger.info(f"Model pushed to the hub in this commit: {url}")
```

`save_pretrained` 方法是用来保存模型的.

保存模型的第一步是要找到最终要保存的模型, 也就是 `model_to_save = unwrap_model(self)` 这行代码所在的工作.

```python
def unwrap_model(model: nn.Module) -> nn.Module:
    if hasattr(model, "module"):
        return unwrap_model(model.module)
    else:
        return model
```

`unwrap_model` 的代码很简单, 就是一个递归操作, 不断的获取 `model.module` 属性. 这是因为现在的模型经常被嵌套在分布式框架中,
所以需要不断取出最内核的模型.

```python
if self._auto_class is not None:
    custom_object_save(self, save_directory, config=self.config)
```

这两行处理了自定义的模型, 实际上就是将定义自定义模型的文件复制到保存目录下, 同时如果有依赖的相对导入的文件也会一并复制.
另外也会更新 `self.config`.

```python
if save_config:
    model_to_save.config.save_pretrained(save_directory)
```

然后当需要保存配置文件时, 就调用配置类的 `save_pretrained` 方法保存配置.

```python
# Save the model
if state_dict is None:
    state_dict = model_to_save.state_dict()
```

`state_dict` 是状态字典, 也就是最终要保存的模型的内容. 如果指定了 `self._keys_to_ignore_on_save`, 会过滤掉不需要保存的 key.

```python
shards, index = shard_checkpoint(state_dict, max_shard_size=max_shard_size)
```

如果单个模型的体积太大的话, 会进行切割, 分成多个文件保存. 同时也会先移除以前保存的文件, 如果在当前目录存在的话.

```python
for shard_file, shard in shards.items():
    save_function(shard, os.path.join(save_directory, shard_file))
```

最后就是用 `save_function` 函数来保存模型了, `save_function` 默认是 `torch.save` 函数.

看完保存后, 再来看看如何加载模型.

```python
@classmethod
def from_pretrained(cls, pretrained_model_name_or_path: Optional[Union[str, os.PathLike]], *model_args, **kwargs):
    # 从 kwargs 中获取一大堆参数
    config = kwargs.pop("config", None)
    state_dict = kwargs.pop("state_dict", None)
    cache_dir = kwargs.pop("cache_dir", None)
    from_tf = kwargs.pop("from_tf", False)
    from_flax = kwargs.pop("from_flax", False)
    ignore_mismatched_sizes = kwargs.pop("ignore_mismatched_sizes", False)
    force_download = kwargs.pop("force_download", False)
    resume_download = kwargs.pop("resume_download", False)
    proxies = kwargs.pop("proxies", None)
    output_loading_info = kwargs.pop("output_loading_info", False)
    local_files_only = kwargs.pop("local_files_only", False)
    use_auth_token = kwargs.pop("use_auth_token", None)
    revision = kwargs.pop("revision", None)
    mirror = kwargs.pop("mirror", None)
    from_pipeline = kwargs.pop("_from_pipeline", None)
    from_auto_class = kwargs.pop("_from_auto", False)
    _fast_init = kwargs.pop("_fast_init", True)
    torch_dtype = kwargs.pop("torch_dtype", None)
    low_cpu_mem_usage = kwargs.pop("low_cpu_mem_usage", False)

    from_pt = not (from_tf | from_flax)

    user_agent = {"file_type": "model", "framework": "pytorch", "from_auto_class": from_auto_class}
    if from_pipeline is not None:
        user_agent["using_pipeline"] = from_pipeline

    if is_offline_mode() and not local_files_only:
        logger.info("Offline mode: forcing local_files_only=True")
        local_files_only = True

    # Load config if we don't provide a configuration
    if not isinstance(config, PretrainedConfig):
        config_path = config if config is not None else pretrained_model_name_or_path
        config, model_kwargs = cls.config_class.from_pretrained(
            config_path,
            cache_dir=cache_dir,
            return_unused_kwargs=True,
            force_download=force_download,
            resume_download=resume_download,
            proxies=proxies,
            local_files_only=local_files_only,
            use_auth_token=use_auth_token,
            revision=revision,
            _from_auto=from_auto_class,
            _from_pipeline=from_pipeline,
            **kwargs,
        )
    else:
        model_kwargs = kwargs

    # This variable will flag if we're loading a sharded checkpoint. In this case the archive file is just the
    # index of the files.
    is_sharded = False
    sharded_metadata = None
    # Load model
    if pretrained_model_name_or_path is not None:
        pretrained_model_name_or_path = str(pretrained_model_name_or_path)
        if os.path.isdir(pretrained_model_name_or_path):
            if from_tf and os.path.isfile(os.path.join(pretrained_model_name_or_path, TF_WEIGHTS_NAME + ".index")):
                # Load from a TF 1.0 checkpoint in priority if from_tf
                archive_file = os.path.join(pretrained_model_name_or_path, TF_WEIGHTS_NAME + ".index")
            elif from_tf and os.path.isfile(os.path.join(pretrained_model_name_or_path, TF2_WEIGHTS_NAME)):
                # Load from a TF 2.0 checkpoint in priority if from_tf
                archive_file = os.path.join(pretrained_model_name_or_path, TF2_WEIGHTS_NAME)
            elif from_flax and os.path.isfile(os.path.join(pretrained_model_name_or_path, FLAX_WEIGHTS_NAME)):
                # Load from a Flax checkpoint in priority if from_flax
                archive_file = os.path.join(pretrained_model_name_or_path, FLAX_WEIGHTS_NAME)
            elif os.path.isfile(os.path.join(pretrained_model_name_or_path, WEIGHTS_NAME)):
                # Load from a PyTorch checkpoint
                archive_file = os.path.join(pretrained_model_name_or_path, WEIGHTS_NAME)
            elif os.path.isfile(os.path.join(pretrained_model_name_or_path, WEIGHTS_INDEX_NAME)):
                # Load from a sharded PyTorch checkpoint
                archive_file = os.path.join(pretrained_model_name_or_path, WEIGHTS_INDEX_NAME)
                is_sharded = True
            # At this stage we don't have a weight file so we will raise an error.
            elif os.path.isfile(
                os.path.join(pretrained_model_name_or_path, TF_WEIGHTS_NAME + ".index")
            ) or os.path.isfile(os.path.join(pretrained_model_name_or_path, TF2_WEIGHTS_NAME)):
                raise EnvironmentError(
                    f"Error no file named {WEIGHTS_NAME} found in directory {pretrained_model_name_or_path} but "
                    "there is a file for TensorFlow weights. Use `from_tf=True` to load this model from those "
                    "weights."
                )
            elif os.path.join(pretrained_model_name_or_path, FLAX_WEIGHTS_NAME):
                raise EnvironmentError(
                    f"Error no file named {WEIGHTS_NAME} found in directory {pretrained_model_name_or_path} but "
                    "there is a file for Flax weights. Use `from_flax=True` to load this model from those "
                    "weights."
                )
            else:
                raise EnvironmentError(
                    f"Error no file named {WEIGHTS_NAME}, {TF2_WEIGHTS_NAME}, {TF_WEIGHTS_NAME + '.index'} or "
                    f"{FLAX_WEIGHTS_NAME} found in directory {pretrained_model_name_or_path}."
                )
        elif os.path.isfile(pretrained_model_name_or_path) or is_remote_url(pretrained_model_name_or_path):
            archive_file = pretrained_model_name_or_path
        elif os.path.isfile(pretrained_model_name_or_path + ".index"):
            if not from_tf:
                raise ValueError(
                    f"We found a TensorFlow checkpoint at {pretrained_model_name_or_path + '.index'}, please set "
                    "from_tf to True to load from this checkpoint."
                )
            archive_file = pretrained_model_name_or_path + ".index"
        else:
            # set correct filename
            if from_tf:
                filename = TF2_WEIGHTS_NAME
            elif from_flax:
                filename = FLAX_WEIGHTS_NAME
            else:
                filename = WEIGHTS_NAME

            archive_file = hf_bucket_url(
                pretrained_model_name_or_path,
                filename=filename,
                revision=revision,
                mirror=mirror,
            )

        try:
            # Load from URL or cache if already cached
            resolved_archive_file = cached_path(
                archive_file,
                cache_dir=cache_dir,
                force_download=force_download,
                proxies=proxies,
                resume_download=resume_download,
                local_files_only=local_files_only,
                use_auth_token=use_auth_token,
                user_agent=user_agent,
            )

        except RepositoryNotFoundError:
            raise EnvironmentError(
                f"{pretrained_model_name_or_path} is not a local folder and is not a valid model identifier "
                "listed on 'https://huggingface.co/models'\nIf this is a private repository, make sure to pass a "
                "token having permission to this repo with `use_auth_token` or log in with `huggingface-cli "
                "login` and pass `use_auth_token=True`."
            )
        except RevisionNotFoundError:
            raise EnvironmentError(
                f"{revision} is not a valid git identifier (branch name, tag name or commit id) that exists for "
                "this model name. Check the model page at "
                f"'https://huggingface.co/{pretrained_model_name_or_path}' for available revisions."
            )
        except EntryNotFoundError:
            if filename == WEIGHTS_NAME:
                try:
                    # Maybe the checkpoint is sharded, we try to grab the index name in this case.
                    archive_file = hf_bucket_url(
                        pretrained_model_name_or_path,
                        filename=WEIGHTS_INDEX_NAME,
                        revision=revision,
                        mirror=mirror,
                    )
                    resolved_archive_file = cached_path(
                        archive_file,
                        cache_dir=cache_dir,
                        force_download=force_download,
                        proxies=proxies,
                        resume_download=resume_download,
                        local_files_only=local_files_only,
                        use_auth_token=use_auth_token,
                        user_agent=user_agent,
                    )
                    is_sharded = True
                except EntryNotFoundError:
                    # Otherwise, maybe there is a TF or Flax model file.  We try those to give a helpful error
                    # message.
                    has_file_kwargs = {
                        "revision": revision,
                        "mirror": mirror,
                        "proxies": proxies,
                        "use_auth_token": use_auth_token,
                    }
                    if has_file(pretrained_model_name_or_path, TF2_WEIGHTS_NAME, **has_file_kwargs):
                        raise EnvironmentError(
                            f"{pretrained_model_name_or_path} does not appear to have a file named {WEIGHTS_NAME} but "
                            "there is a file for TensorFlow weights. Use `from_tf=True` to load this model from those "
                            "weights."
                        )
                    elif has_file(pretrained_model_name_or_path, FLAX_WEIGHTS_NAME, **has_file_kwargs):
                        raise EnvironmentError(
                            f"{pretrained_model_name_or_path} does not appear to have a file named {WEIGHTS_NAME} but "
                            "there is a file for Flax weights. Use `from_flax=True` to load this model from those "
                            "weights."
                        )
                    else:
                        raise EnvironmentError(
                            f"{pretrained_model_name_or_path} does not appear to have a file named {WEIGHTS_NAME}, "
                            f"{TF2_WEIGHTS_NAME}, {TF_WEIGHTS_NAME} or {FLAX_WEIGHTS_NAME}."
                        )
            else:
                raise EnvironmentError(
                    f"{pretrained_model_name_or_path} does not appear to have a file named {filename}."
                )
        except HTTPError as err:
            raise EnvironmentError(
                f"There was a specific connection error when trying to load {pretrained_model_name_or_path}:\n"
                f"{err}"
            )
        except ValueError:
            raise EnvironmentError(
                f"We couldn't connect to '{HUGGINGFACE_CO_RESOLVE_ENDPOINT}' to load this model, couldn't find it in the cached "
                f"files and it looks like {pretrained_model_name_or_path} is not the path to a directory "
                f"containing a file named {WEIGHTS_NAME}, {TF2_WEIGHTS_NAME}, {TF_WEIGHTS_NAME} or "
                f"{FLAX_WEIGHTS_NAME}.\n"
                "Checkout your internet connection or see how to run the library in offline mode at "
                "'https://huggingface.co/docs/transformers/installation#offline-mode'."
            )
        except EnvironmentError:
            raise EnvironmentError(
                f"Can't load the model for '{pretrained_model_name_or_path}'. If you were trying to load it from "
                "'https://huggingface.co/models', make sure you don't have a local directory with the same name. "
                f"Otherwise, make sure '{pretrained_model_name_or_path}' is the correct path to a directory "
                f"containing a file named {WEIGHTS_NAME}, {TF2_WEIGHTS_NAME}, {TF_WEIGHTS_NAME} or "
                f"{FLAX_WEIGHTS_NAME}."
            )

        if resolved_archive_file == archive_file:
            logger.info(f"loading weights file {archive_file}")
        else:
            logger.info(f"loading weights file {archive_file} from cache at {resolved_archive_file}")
    else:
        resolved_archive_file = None

    # We'll need to download and cache each checkpoint shard if the checkpoint is sharded.
    if is_sharded:
        # resolved_archive_file becomes a list of files that point to the different checkpoint shards in this case.
        resolved_archive_file, sharded_metadata = get_checkpoint_shard_files(
            pretrained_model_name_or_path,
            resolved_archive_file,
            cache_dir=cache_dir,
            force_download=force_download,
            proxies=proxies,
            resume_download=resume_download,
            local_files_only=local_files_only,
            use_auth_token=use_auth_token,
            user_agent=user_agent,
            revision=revision,
            mirror=mirror,
        )

    # load pt weights early so that we know which dtype to init the model under
    if from_pt:
        if not is_sharded and state_dict is None:
            # Time to load the checkpoint
            state_dict = load_state_dict(resolved_archive_file)

        # set dtype to instantiate the model under:
        # 1. If torch_dtype is not None, we use that dtype
        # 2. If torch_dtype is "auto", we auto-detect dtype from the loaded state_dict, by checking its first
        #    weights entry - we assume all weights are of the same dtype
        # we also may have config.torch_dtype available, but we won't rely on it till v5
        dtype_orig = None
        if torch_dtype is not None:
            if isinstance(torch_dtype, str):
                if torch_dtype == "auto":
                    if is_sharded and "dtype" in sharded_metadata:
                        torch_dtype = sharded_metadata["dtype"]
                    elif not is_sharded:
                        torch_dtype = next(iter(state_dict.values())).dtype
                    else:
                        one_state_dict = load_state_dict(resolved_archive_file)
                        torch_dtype = next(iter(one_state_dict.values())).dtype
                        del one_state_dict  # free CPU memory
                else:
                    raise ValueError(
                        f"`torch_dtype` can be either a `torch.dtype` or `auto`, but received {torch_dtype}"
                    )
            dtype_orig = cls._set_default_torch_dtype(torch_dtype)

        if is_sharded:
            loaded_state_dict_keys = sharded_metadata["all_checkpoint_keys"]
        else:
            loaded_state_dict_keys = [k for k in state_dict.keys()]
        if low_cpu_mem_usage:
            state_dict = None

    config.name_or_path = pretrained_model_name_or_path

    # Instantiate model.
    if is_deepspeed_zero3_enabled():
        import deepspeed

        logger.info("Detected DeepSpeed ZeRO-3: activating zero.init() for this model")
        # this immediately partitions the model across all gpus, to avoid the overhead in time
        # and memory copying it on CPU or each GPU first
        with deepspeed.zero.Init(config_dict_or_path=deepspeed_config()):
            with no_init_weights(_enable=_fast_init):
                model = cls(config, *model_args, **model_kwargs)
    else:
        with no_init_weights(_enable=_fast_init):
            model = cls(config, *model_args, **model_kwargs)

    if from_tf:
        if resolved_archive_file.endswith(".index"):
            # Load from a TensorFlow 1.X checkpoint - provided by original authors
            model = cls.load_tf_weights(model, config, resolved_archive_file[:-6])  # Remove the '.index'
        else:
            # Load from our TensorFlow 2.0 checkpoints
            try:
                from .modeling_tf_pytorch_utils import load_tf2_checkpoint_in_pytorch_model

                model = load_tf2_checkpoint_in_pytorch_model(model, resolved_archive_file, allow_missing_keys=True)
            except ImportError:
                logger.error(
                    "Loading a TensorFlow model in PyTorch, requires both PyTorch and TensorFlow to be installed. Please see "
                    "https://pytorch.org/ and https://www.tensorflow.org/install/ for installation instructions."
                )
                raise
    elif from_flax:
        try:
            from .modeling_flax_pytorch_utils import load_flax_checkpoint_in_pytorch_model

            model = load_flax_checkpoint_in_pytorch_model(model, resolved_archive_file)
        except ImportError:
            logger.error(
                "Loading a Flax model in PyTorch, requires both PyTorch and Flax to be installed. Please see "
                "https://pytorch.org/ and https://flax.readthedocs.io/en/latest/installation.html for installation instructions."
            )
            raise
    elif from_pt:

        # restore default dtype
        if dtype_orig is not None:
            torch.set_default_dtype(dtype_orig)

        model, missing_keys, unexpected_keys, mismatched_keys, error_msgs = cls._load_pretrained_model(
            model,
            state_dict,
            loaded_state_dict_keys,  # XXX: rename?
            resolved_archive_file,
            pretrained_model_name_or_path,
            ignore_mismatched_sizes=ignore_mismatched_sizes,
            sharded_metadata=sharded_metadata,
            _fast_init=_fast_init,
            low_cpu_mem_usage=low_cpu_mem_usage,
        )

    # make sure token embedding weights are still tied if needed
    model.tie_weights()

    # Set model in evaluation mode to deactivate DropOut modules by default
    model.eval()

    if output_loading_info:
        loading_info = {
            "missing_keys": missing_keys,
            "unexpected_keys": unexpected_keys,
            "mismatched_keys": mismatched_keys,
            "error_msgs": error_msgs,
        }
        return model, loading_info

    return model
```

大致上来讲, 第一步是获取 `resolved_archive_file`, 不管是本地目录获取, 还是从 hf 的代码仓库中获取.
然后使用 `model = cls(config, *model_args, **model_kwargs)` 初始化模型实例.
接着针对不同的框架, 加载模型的权重.

tf1 版本是 `model = cls.load_tf_weights(model, config, resolved_archive_file[:-6])`.
tf2 版本是 `model = load_tf2_checkpoint_in_pytorch_model(model, resolved_archive_file, allow_missing_keys=True)`.
torch 版本是

```python
model, missing_keys, unexpected_keys, mismatched_keys, error_msgs = cls._load_pretrained_model(
    model,
    state_dict,
    loaded_state_dict_keys,  # XXX: rename?
    resolved_archive_file,
    pretrained_model_name_or_path,
    ignore_mismatched_sizes=ignore_mismatched_sizes,
    sharded_metadata=sharded_metadata,
    _fast_init=_fast_init,
    low_cpu_mem_usage=low_cpu_mem_usage,
)
```

# 小结

到这里, `PreTrainedModel` 模型就大致看完了. 下节会找个具体的模型来看看是怎么实现的.

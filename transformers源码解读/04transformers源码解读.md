[TOC]

# 简介

上节已经讲过了 `PreTrainedTokenizerBase` 和 `PreTrainedTokenizerFast`.
这次继续来看分词器, 看看 `PreTrainedTokenizer`.

# PreTrainedTokenizer

先看下 `PreTrainedTokenizer` 的初始化方法.

```python
def __init__(self, **kwargs):
    super().__init__(**kwargs)

    # Added tokens - We store this for both slow and fast tokenizers
    # until the serialization of Fast tokenizers is updated
    self.added_tokens_encoder: Dict[str, int] = {}
    self.added_tokens_decoder: Dict[int, str] = {}
    self.unique_no_split_tokens: List[str] = []
    self.tokens_trie = Trie()

    self._decode_use_source_tokenizer = False
```

值得注意的是初始化了 trie 树, `tokens_trie`.

> 在计算机科学中，trie，又称前缀树或字典树，是一种有序树，用于保存关联数组，其中的键通常是字符串。与二叉查找树不同，键不是直接保存在节点中，而是由节点在树中的位置决定。一个节点的所有子孙都有相同的前缀，也就是这个节点对应的字符串，而根节点对应空字符串。一般情况下，不是所有的节点都有对应的值，只有叶子节点和部分内部节点所对应的键才有相关的值。

引用下维基百科的解释, trie 树是个非常有用的数据结构, 正好来看看它的实现方式.

```python
class Trie:
    def __init__(self):
        self.data = {}

    def add(self, word: str):
        if not word:
            # Prevent empty string
            return
        ref = self.data
        for char in word:
            ref[char] = char in ref and ref[char] or {}
            ref = ref[char]
        ref[""] = 1
```

初始化方法很简单, 就是使用 `self.data` 来保存内部的结构.

`add` 方法将一个单词添加进 `self.data` 中. 首先跳过空字符串. 然后使用递归的方式, 每次将一个字符添加进去.
`char in ref and ref[char] or {}` 的方式看起来很巧妙, 简单来说就是存在 `ref[data]` 就使用或者初始化一个新的 `{}`.
在最后, 使用终止符 `""`, 并将值设置为 1.

代码里也有注释, 可供参数.

```python
>>> trie = Trie()
>>> trie.add("Hello 友達")
>>> trie.data
{"H": {"e": {"l": {"l": {"o": {" ": {"友": {"達": {"": 1}}}}}}}}}

>>> trie.add("Hello")
>>> trie.data
{"H": {"e": {"l": {"l": {"o": {"": 1, " ": {"友": {"達": {"": 1}}}}}}}}}
```

接下了看看最关键的 `split` 方法, 在使用 `add` 在 trie 树中补充词汇表后, 如何切割文本.

```python
def split(self, text: str) -> List[str]:
    states = OrderedDict()

    offsets = [0]

    skip = 0
    for current, current_char in enumerate(text):
        if skip and current < skip:
            continue

        to_remove = set()
        reset = False

        for start, trie_pointer in states.items():
            if "" in trie_pointer:
                for lookstart, looktrie_pointer in states.items():
                    if lookstart > start:
                        break
                    elif lookstart < start:
                        lookahead_index = current + 1
                        end = current + 1
                    else:
                        lookahead_index = current
                        end = current
                    next_char = text[lookahead_index] if lookahead_index < len(text) else None
                    if "" in looktrie_pointer:
                        start = lookstart
                        end = lookahead_index
                        skip = lookahead_index

                    while next_char in looktrie_pointer:
                        looktrie_pointer = looktrie_pointer[next_char]
                        lookahead_index += 1
                        if "" in looktrie_pointer:
                            start = lookstart
                            end = lookahead_index
                            skip = lookahead_index

                        if lookahead_index == len(text):
                            break
                        next_char = text[lookahead_index]

                offsets.append(start)
                offsets.append(end)
                reset = True
                break
            elif current_char in trie_pointer:
                trie_pointer = trie_pointer[current_char]

                states[start] = trie_pointer
            else:
                to_remove.add(start)

        if reset:
            states = {}
        else:
            for start in to_remove:
                del states[start]

        if current >= skip and current_char in self.data:
            states[current] = self.data[current_char]

    for start, trie_pointer in states.items():
        if "" in trie_pointer:
            end = len(text)
            offsets.append(start)
            offsets.append(end)
            break

    return self.cut_text(text, offsets)
```

代码不多, 但是原文的注释很多, 这里都去掉了. 主要就是构建 `offsets`, 然后据此切分成单词列表.

```python
def cut_text(self, text, offsets):
    offsets.append(len(text))
    tokens = []
    start = 0
    for end in offsets:
        if start > end:
            logger.error(
                "There was a bug in Trie algorithm in tokenization. Attempting to recover. Please report it anyway."
            )
            continue
        elif start == end:
            continue
        tokens.append(text[start:end])
        start = end

    return tokens
```

还是稍微展开讲一讲吧. 主体是一个大的 for 循环, 即 `for current, current_char in enumerate(text):`,
按照文本中的每个字符开始遍历.

当第一次循环的时候, 因为 `states` 是空的, 所以可以直接跳转到 for 循环的最后, 也就是这两行:

```python
if current >= skip and current_char in self.data:
    states[current] = self.data[current_char]
```

这两行的作用, 先忽略第一个判断, 后一个判断是说当前字符是否在 trie 树的开头字符中.
trie 树构建起来后, `self.data` 的第一层就是所有单词的第一个字母. 当匹配上的话,
就填充 `states`, key 是当前的索引位置, value 是后续的可能性, 也就是一个小 trie 树,
表示这个字符后面的所有可能性.

如果运气好, 第一个字符就匹配上了. 如果运气太坏, 整个文本, 即 `text` 都没匹配上, 那最终就是把整个文本当成是一个单词了.

现在假设第一个字符已经匹配上了, 就可以看下一个字符了, 这是 `states` 中已经有值了.
所以我们可以进入内部的 for 循环, 即 `for start, trie_pointer in states.items():` 中一探究竟了.

这个 for 循环中, 主要就是描述了三大可能性.

1. 当前位置已经匹配到结尾符了, 即 `if "" in trie_pointer:`, 可以构建一个单词了.
2. 当前位置没有结尾符, 但是当前字符在后续的可能性中, 即 `elif current_char in trie_pointer:`, 可以继续匹配了.
3. 当前位置不在后续的可能性中, 匹配无法继续了.

代码最多的部分就是第一种可能性了, 这表示已经匹配到一个单词了, 但是还有别的情况, 也是当前匹配到的单词不是最长匹配单词.\
比如 `extra_id_1` 和 `extra_id_100` 都在词汇表中, 当前的文本是 `extra_id_100 vs extra_id_1`.
那么虽然最开始匹配的 `extra_id_1` 已经到了结尾符了, 但是向前看, 还可以匹配到 `extra_id_100`.
所有这一步也就是前向匹配, 即 lookahead.

第一种可能性也是唯一填充 `offsets` 的方式. 当发现一个最长匹配的单词后, 就可以将它添加进 `offsets` 列表中了.

```python
offsets.append(start)
offsets.append(end)
reset = True
break
```

在 `split` 方法的最后, 就调用了 `cut_text` 方法, 根据 `offsets` 将文本切成了单词的列表.

这就是 Trie 类的所有方法了. 主要就是涉及到添加词汇表和切分文本两个功能.

接下来看看 `tokenize` 方法.

```python
def tokenize(self, text: TextInput, **kwargs) -> List[str]:
    all_special_tokens_extended = dict(
        (str(t), t) for t in self.all_special_tokens_extended if isinstance(t, AddedToken)
    )

    text, kwargs = self.prepare_for_tokenization(text, **kwargs)

    if kwargs:
        logger.warning(f"Keyword arguments {kwargs} not recognized.")

    if hasattr(self, "do_lower_case") and self.do_lower_case:
        escaped_special_toks = [
            re.escape(s_tok) for s_tok in (self.unique_no_split_tokens + self.all_special_tokens)
        ]
        pattern = r"(" + r"|".join(escaped_special_toks) + r")|" + r"(.+?)"
        text = re.sub(pattern, lambda m: m.groups()[0] or m.groups()[1].lower(), text)

    no_split_token = set(self.unique_no_split_tokens)
    tokens = self.tokens_trie.split(text)
    for i, token in enumerate(tokens):
        if token in no_split_token:
            tok_extended = all_special_tokens_extended.get(token, None)
            left = tokens[i - 1] if i > 0 else None
            right = tokens[i + 1] if i < len(tokens) - 1 else None
            if isinstance(tok_extended, AddedToken):
                if tok_extended.rstrip and right:
                    tokens[i + 1] = right.lstrip()
                if tok_extended.lstrip and left:
                    tokens[i - 1] = left.rstrip()  # Opposite here
            else:
                if right:
                    tokens[i + 1] = right.lstrip()
                if left:
                    tokens[i - 1] = left.rstrip()
    tokenized_text = []
    for token in tokens:
        if not token:
            continue
        if token in no_split_token:
            tokenized_text.append(token)
        else:
            tokenized_text.extend(self._tokenize(token))
    return tokenized_text
```

主要是用 `self.tokens_trie.split(text)` 分词了一下, 然后别的单词用 `tokenized_text.extend(self._tokenize(token))` 分词.
`self._tokenize` 当然又是没有实现的.

剩余的方法中已经没有什么特殊的了, 就不再继续看了.

# BertTokenizerFast

看完了基础类后, 以 bert 为例, 看看具体的模型相关的 tokenizer 类的实现.

相关的类有两个, `BertTokenizerFast` 是快速版的实现, `BertTokenizer` 是慢速版.

快速版的代码少些, 先来看看初始化方法.

```python
class BertTokenizerFast(PreTrainedTokenizerFast):
    vocab_files_names = VOCAB_FILES_NAMES
    pretrained_vocab_files_map = PRETRAINED_VOCAB_FILES_MAP
    pretrained_init_configuration = PRETRAINED_INIT_CONFIGURATION
    max_model_input_sizes = PRETRAINED_POSITIONAL_EMBEDDINGS_SIZES
    slow_tokenizer_class = BertTokenizer

    def __init__(
        self,
        vocab_file=None,
        tokenizer_file=None,
        do_lower_case=True,
        unk_token="[UNK]",
        sep_token="[SEP]",
        pad_token="[PAD]",
        cls_token="[CLS]",
        mask_token="[MASK]",
        tokenize_chinese_chars=True,
        strip_accents=None,
        **kwargs
    ):
        super().__init__(
            vocab_file,
            tokenizer_file=tokenizer_file,
            do_lower_case=do_lower_case,
            unk_token=unk_token,
            sep_token=sep_token,
            pad_token=pad_token,
            cls_token=cls_token,
            mask_token=mask_token,
            tokenize_chinese_chars=tokenize_chinese_chars,
            strip_accents=strip_accents,
            **kwargs,
        )

        normalizer_state = json.loads(self.backend_tokenizer.normalizer.__getstate__())
        if (
            normalizer_state.get("lowercase", do_lower_case) != do_lower_case
            or normalizer_state.get("strip_accents", strip_accents) != strip_accents
            or normalizer_state.get("handle_chinese_chars", tokenize_chinese_chars) != tokenize_chinese_chars
        ):
            normalizer_class = getattr(normalizers, normalizer_state.pop("type"))
            normalizer_state["lowercase"] = do_lower_case
            normalizer_state["strip_accents"] = strip_accents
            normalizer_state["handle_chinese_chars"] = tokenize_chinese_chars
            self.backend_tokenizer.normalizer = normalizer_class(**normalizer_state)

        self.do_lower_case = do_lower_case
```

快速版的自然是继承自 `PreTrainedTokenizerFast`. 一开始定义了一些类属性, 基本都是些字典信息.
同时指定了慢速版的类是 `slow_tokenizer_class = BertTokenizer`.

在 `__init__` 方法中, 首先是调用了父类的初始化, 然后加载了 `normalizer_state`.
加载后, 判断是否和当前参数不同, 以传入的参数为准, 重新初始化了 `self.backend_tokenizer.normalizer`.

```python
def build_inputs_with_special_tokens(self, token_ids_0, token_ids_1=None):
    output = [self.cls_token_id] + token_ids_0 + [self.sep_token_id]

    if token_ids_1:
        output += token_ids_1 + [self.sep_token_id]

    return output
```

`build_inputs_with_special_tokens` 方法主要是对 output 做了修改, 添加了首尾的特殊字符.

```python
def create_token_type_ids_from_sequences(
    self, token_ids_0: List[int], token_ids_1: Optional[List[int]] = None
) -> List[int]:
    sep = [self.sep_token_id]
    cls = [self.cls_token_id]
    if token_ids_1 is None:
        return len(cls + token_ids_0 + sep) * [0]
    return len(cls + token_ids_0 + sep) * [0] + len(token_ids_1 + sep) * [1]
```

`create_token_type_ids_from_sequences` 方法主要是创建 `token_type_ids`, 就是用来区分序列的.
简单来说就是如果只有一个序列, 就全部返回 0, 如果有两个序列, 第一个序列是 0, 第二个序列是 1.
每个序列都是以 `self.sep_token_id` 结尾的.

```python
def save_vocabulary(self, save_directory: str, filename_prefix: Optional[str] = None) -> Tuple[str]:
    files = self._tokenizer.model.save(save_directory, name=filename_prefix)
    return tuple(files)
```

最后一个函数是用来保存词汇表的.

# BertTokenizer

```python
class BertTokenizer(PreTrainedTokenizer):
    vocab_files_names = VOCAB_FILES_NAMES
    pretrained_vocab_files_map = PRETRAINED_VOCAB_FILES_MAP
    pretrained_init_configuration = PRETRAINED_INIT_CONFIGURATION
    max_model_input_sizes = PRETRAINED_POSITIONAL_EMBEDDINGS_SIZES

    def __init__(
        self,
        vocab_file,
        do_lower_case=True,
        do_basic_tokenize=True,
        never_split=None,
        unk_token="[UNK]",
        sep_token="[SEP]",
        pad_token="[PAD]",
        cls_token="[CLS]",
        mask_token="[MASK]",
        tokenize_chinese_chars=True,
        strip_accents=None,
        **kwargs
    ):
        super().__init__(
            do_lower_case=do_lower_case,
            do_basic_tokenize=do_basic_tokenize,
            never_split=never_split,
            unk_token=unk_token,
            sep_token=sep_token,
            pad_token=pad_token,
            cls_token=cls_token,
            mask_token=mask_token,
            tokenize_chinese_chars=tokenize_chinese_chars,
            strip_accents=strip_accents,
            **kwargs,
        )

        if not os.path.isfile(vocab_file):
            raise ValueError(
                f"Can't find a vocabulary file at path '{vocab_file}'. To load the vocabulary from a Google pretrained "
                "model use `tokenizer = BertTokenizer.from_pretrained(PRETRAINED_MODEL_NAME)`"
            )
        # 加载词汇表
        self.vocab = load_vocab(vocab_file)
        self.ids_to_tokens = collections.OrderedDict([(ids, tok) for tok, ids in self.vocab.items()])
        self.do_basic_tokenize = do_basic_tokenize
        # 两个分词器
        if do_basic_tokenize:
            self.basic_tokenizer = BasicTokenizer(
                do_lower_case=do_lower_case,
                never_split=never_split,
                tokenize_chinese_chars=tokenize_chinese_chars,
                strip_accents=strip_accents,
            )
        self.wordpiece_tokenizer = WordpieceTokenizer(vocab=self.vocab, unk_token=self.unk_token)
```

`BertTokenizer` 继承自 `PreTrainedTokenizer`, 初始化的时候必须提供 `vocab_file`.
词汇表是从词汇文件中加载的, 然后定义了两个分词器, 分别是 `BasicTokenizer` 和 `WordpieceTokenizer`.

那么就先来看看这两个分词器的实现.

主要是 `tokenize` 的实现部分.

```python
class BasicTokenizer(object):
    def __init__(self, do_lower_case=True, never_split=None, tokenize_chinese_chars=True, strip_accents=None):
        if never_split is None:
            never_split = []
        self.do_lower_case = do_lower_case
        self.never_split = set(never_split)
        self.tokenize_chinese_chars = tokenize_chinese_chars
        # strip 重音符号
        self.strip_accents = strip_accents

    def tokenize(self, text, never_split=None):
        """
        Basic Tokenization of a piece of text. Split on "white spaces" only, for sub-word tokenization, see
        WordPieceTokenizer.

        Args:
            never_split (`List[str]`, *optional*)
                Kept for backward compatibility purposes. Now implemented directly at the base class level (see
                [`PreTrainedTokenizer.tokenize`]) List of token not to split.
        """
        # union() returns a new set by concatenating the two sets.
        never_split = self.never_split.union(set(never_split)) if never_split else self.never_split
        text = self._clean_text(text)

        # This was added on November 1st, 2018 for the multilingual and Chinese
        # models. This is also applied to the English models now, but it doesn't
        # matter since the English models were not trained on any Chinese data
        # and generally don't have any Chinese data in them (there are Chinese
        # characters in the vocabulary because Wikipedia does have some Chinese
        # words in the English Wikipedia.).
        if self.tokenize_chinese_chars:
            text = self._tokenize_chinese_chars(text)
        # 使用的是空白分词器
        orig_tokens = whitespace_tokenize(text)
        split_tokens = []
        for token in orig_tokens:
            if token not in never_split:
                if self.do_lower_case:
                    token = token.lower()
                    # 是否需要去除重音符号
                    if self.strip_accents is not False:
                        token = self._run_strip_accents(token)
                elif self.strip_accents:
                    token = self._run_strip_accents(token)
            # 加入列表
            split_tokens.extend(self._run_split_on_punc(token, never_split))

        # 再来一遍
        output_tokens = whitespace_tokenize(" ".join(split_tokens))
        return output_tokens
```

`BasicTokenizer` 本质上还是使用了空白符作为分割, 也就是调用 `whitespace_tokenize` 的函数的那两行.
其中夹杂着对中文字符, 重音符和标点符号的处理.

```python
class WordpieceTokenizer(object):
    def __init__(self, vocab, unk_token, max_input_chars_per_word=100):
        self.vocab = vocab
        self.unk_token = unk_token
        self.max_input_chars_per_word = max_input_chars_per_word

    def tokenize(self, text):
        output_tokens = []
        for token in whitespace_tokenize(text):
            chars = list(token)
            if len(chars) > self.max_input_chars_per_word:
                output_tokens.append(self.unk_token)
                continue

            is_bad = False
            start = 0
            sub_tokens = []
            while start < len(chars):
                end = len(chars)
                cur_substr = None
                while start < end:
                    substr = "".join(chars[start:end])
                    if start > 0:
                        substr = "##" + substr
                    if substr in self.vocab:
                        cur_substr = substr
                        break
                    end -= 1
                if cur_substr is None:
                    is_bad = True
                    break
                sub_tokens.append(cur_substr)
                start = end

            if is_bad:
                output_tokens.append(self.unk_token)
            else:
                output_tokens.extend(sub_tokens)
        return output_tokens
```

再来看 `WordpieceTokenizer`. 同样是用了 `whitespace_tokenize`, 不同之处在于对于未知的单词的处理.
`WordpieceTokenizer` 是要求有词汇表的, 对于未知的单词, 可能会尝试拆分, 就在这个 `for token in whitespace_tokenize(text):` 中.
比如 `helloworld` 这个单词不在词汇表中, 但是 `hello` 和 `##world` 都在, 就会拆分成这两个单词.

继续回到 `BasicTokenizer`. `_tokenize` 这一块就是根据不同的情况使用 `self.basic_tokenizer` 和 `self.wordpiece_tokenizer`.
`basic_tokenizer` 是可选的, 即使在使用 `basic_tokenizer` 之后, 也要使用 `wordpiece_tokenizer` 对 token 再次进行分割.

```python
def _tokenize(self, text):
    split_tokens = []
    if self.do_basic_tokenize:
        for token in self.basic_tokenizer.tokenize(text, never_split=self.all_special_tokens):

            # If the token is part of the never_split set
            if token in self.basic_tokenizer.never_split:
                split_tokens.append(token)
            else:
                split_tokens += self.wordpiece_tokenizer.tokenize(token)
    else:
        split_tokens = self.wordpiece_tokenizer.tokenize(text)
    return split_tokens
```

最后看下如何保存词汇表.

```python
def save_vocabulary(self, save_directory: str, filename_prefix: Optional[str] = None) -> Tuple[str]:
    index = 0
    if os.path.isdir(save_directory):
        vocab_file = os.path.join(
            save_directory, (filename_prefix + "-" if filename_prefix else "") + VOCAB_FILES_NAMES["vocab_file"]
        )
    else:
        vocab_file = (filename_prefix + "-" if filename_prefix else "") + save_directory
    with open(vocab_file, "w", encoding="utf-8") as writer:
        for token, token_index in sorted(self.vocab.items(), key=lambda kv: kv[1]):
            if index != token_index:
                logger.warning(
                    f"Saving vocabulary to {vocab_file}: vocabulary indices are not consecutive."
                    " Please check that the vocabulary is not corrupted!"
                )
                index = token_index
            writer.write(token + "\n")
            index += 1
    return (vocab_file,)
```

保存的时候, 会判断下 index 和 token_index 是否保持一致. 如果不一致, 就可能是 `self.vocab` 被修改了, 会警告下.

# 小结

到这里为此, tokenizer 部分就已经看的差不多了. 接下来会专注到别的部分.

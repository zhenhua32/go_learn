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

<!-- TOC -->

- [简介](#简介)
- [HttpRouter 实现](#httprouter-实现)
- [数据结构](#数据结构)
- [添加路由](#添加路由)
  - [addRoute](#addroute)
  - [insertChild](#insertchild)
- [获取数据](#获取数据)
- [总结](#总结)

<!-- /TOC -->

## 简介

Gin 源码解读, 基于 [v1.5.0 ](https://github.com/gin-gonic/gin/tree/v1.5.0) 版本.

## HttpRouter 实现

添加路由主要是由 `addRoute` 完成:

```go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
	assert1(path[0] == '/', "path must begin with '/'")
	assert1(method != "", "HTTP method can not be empty")
	assert1(len(handlers) > 0, "there must be at least one handler")

	debugPrintRoute(method, path, handlers)
	root := engine.trees.get(method)
	if root == nil {
		root = new(node)
		root.fullPath = "/"
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
	root.addRoute(path, handlers)
}
```

Gin 的路由是通过 [httprouter](https://github.com/julienschmidt/httprouter) 实现的, 来深入了解下它的源代码.

## 数据结构

github 的文档解释了实现原理, 具体可以参考 [How does it work?](https://github.com/julienschmidt/httprouter#how-does-it-work).

HttpRouter 内部使用了 Radix 树, 是前缀树的紧凑版变种.

![radix tree](./img/page/radix_tree.png)

上图来自维基百科, 显示了 Radix 树的结构. 相比普通前缀树, Radix 树的边上能存储多个字符, 极大的压缩了树的深度.

看一下数据结构的定义:

```go
// Param is a single URL parameter, consisting of a key and a value.
type Param struct {
	Key   string
	Value string
}

// Params is a Param-slice, as returned by the router.
// The slice is ordered, the first URL parameter is also the first slice value.
// It is therefore safe to read values by the index.
type Params []Param

type methodTree struct {
	method string
	root   *node
}

type methodTrees []methodTree
```

`Engine.trees` 的类型就是 `methodTrees`, 初始化语句是 `trees: make(methodTrees, 0, 9),`.

```go
func (trees methodTrees) get(method string) *node {
	for _, tree := range trees {
		if tree.method == method {
			return tree.root
		}
	}
	return nil
}
```

前面添加路由的代码中第一步是找到 root, 即 `root := engine.trees.get(method)`, 结合 `get` 代码,
我们可以发现 `methodTrees` 实际上根据 HTTP 方法分类的, 每种方法都对应一颗树.

如果当前该类型的 HTTP 方法不存在, 就新建一棵树 `methodTree`:

```go
root = new(node)
root.fullPath = "/"
engine.trees = append(engine.trees, methodTree{method: method, root: root})
```

再看一下树的节点是如何定义的:

```go
type nodeType uint8

const (
	static nodeType = iota // default
	root
	param
	catchAll
)

type node struct {
	path      string
	indices   string
	children  []*node
	handlers  HandlersChain
	priority  uint32
	nType     nodeType
	maxParams uint8
	wildChild bool
	fullPath  string
}
```

## 添加路由

数据结构已经了解了, 看一下路由到底是如何添加的, 即 `root.addRoute(path, handlers)`.

```go
// addRoute adds a node with the given handle to the path.
// Not concurrency-safe!
func (n *node) addRoute(path string, handlers HandlersChain) {
	fullPath := path
	n.priority++
	numParams := countParams(path)

	parentFullPathIndex := 0

	// non-empty tree
	if len(n.path) > 0 || len(n.children) > 0 {
	walk:
		for {
			// Update maxParams of the current node
			if numParams > n.maxParams {
				n.maxParams = numParams
			}

			// Find the longest common prefix.
			// This also implies that the common prefix contains no ':' or '*'
			// since the existing key can't contain those chars.
			i := 0
			max := min(len(path), len(n.path))
			for i < max && path[i] == n.path[i] {
				i++
			}

			// Split edge
			if i < len(n.path) {
				child := node{
					path:      n.path[i:],
					wildChild: n.wildChild,
					indices:   n.indices,
					children:  n.children,
					handlers:  n.handlers,
					priority:  n.priority - 1,
					fullPath:  n.fullPath,
				}

				// Update maxParams (max of all children)
				for i := range child.children {
					if child.children[i].maxParams > child.maxParams {
						child.maxParams = child.children[i].maxParams
					}
				}

				n.children = []*node{&child}
				// []byte for proper unicode char conversion, see #65
				n.indices = string([]byte{n.path[i]})
				n.path = path[:i]
				n.handlers = nil
				n.wildChild = false
				n.fullPath = fullPath[:parentFullPathIndex+i]
			}

			// Make new node a child of this node
			if i < len(path) {
				path = path[i:]

				if n.wildChild {
					parentFullPathIndex += len(n.path)
					n = n.children[0]
					n.priority++

					// Update maxParams of the child node
					if numParams > n.maxParams {
						n.maxParams = numParams
					}
					numParams--

					// Check if the wildcard matches
					if len(path) >= len(n.path) && n.path == path[:len(n.path)] {
						// check for longer wildcard, e.g. :name and :names
						if len(n.path) >= len(path) || path[len(n.path)] == '/' {
							continue walk
						}
					}

					pathSeg := path
					if n.nType != catchAll {
						pathSeg = strings.SplitN(path, "/", 2)[0]
					}
					prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
					panic("'" + pathSeg +
						"' in new path '" + fullPath +
						"' conflicts with existing wildcard '" + n.path +
						"' in existing prefix '" + prefix +
						"'")
				}

				c := path[0]

				// slash after param
				if n.nType == param && c == '/' && len(n.children) == 1 {
					parentFullPathIndex += len(n.path)
					n = n.children[0]
					n.priority++
					continue walk
				}

				// Check if a child with the next path byte exists
				for i := 0; i < len(n.indices); i++ {
					if c == n.indices[i] {
						parentFullPathIndex += len(n.path)
						i = n.incrementChildPrio(i)
						n = n.children[i]
						continue walk
					}
				}

				// Otherwise insert it
				if c != ':' && c != '*' {
					// []byte for proper unicode char conversion, see #65
					n.indices += string([]byte{c})
					child := &node{
						maxParams: numParams,
						fullPath:  fullPath,
					}
					n.children = append(n.children, child)
					n.incrementChildPrio(len(n.indices) - 1)
					n = child
				}
				n.insertChild(numParams, path, fullPath, handlers)
				return

			} else if i == len(path) { // Make node a (in-path) leaf
				if n.handlers != nil {
					panic("handlers are already registered for path '" + fullPath + "'")
				}
				n.handlers = handlers
			}
			return
		}
	} else { // Empty tree
		n.insertChild(numParams, path, fullPath, handlers)
		n.nType = root
	}
}
```

### addRoute

代码有点长, 先根据 if 语句分为两种情况, 一种是初始化的时候(即树是空的), 另一种是树是非空的.

```go
n.insertChild(numParams, path, fullPath, handlers)
n.nType = root
```

树是空的情况下, 即 n.path 是空字符串(初始值) 且 n.children 是空切片.
这个时候, 只是通过 `insertChild` 插入节点, 然后将节点的类型设置为 `root`.
`insertChild` 的代码也有点长, 等下再来看.

当树是非空的, 进入到了一个 for 循环中, 先跟着注释看一下 for 大体上是做什么的.

```go
// Update maxParams of the current node
if numParams > n.maxParams {
  n.maxParams = numParams
}

// Find the longest common prefix.
// This also implies that the common prefix contains no ':' or '*'
// since the existing key can't contain those chars.
i := 0
max := min(len(path), len(n.path))
for i < max && path[i] == n.path[i] {
  i++
}

// Split edge

// Make new node a child of this node
```

前面两个步骤, 更新 maxParams 和计算最长前缀的长度, 非常简单, 直接看代码就行.

看一下节点是如何分裂的, 即第三步:

```go
// Split edge
if i < len(n.path) {
  child := node{
    path:      n.path[i:],
    wildChild: n.wildChild,
    indices:   n.indices,
    children:  n.children,
    handlers:  n.handlers,
    priority:  n.priority - 1,
    fullPath:  n.fullPath,
  }

  // Update maxParams (max of all children)
  for i := range child.children {
    if child.children[i].maxParams > child.maxParams {
      child.maxParams = child.children[i].maxParams
    }
  }

  n.children = []*node{&child}
  // []byte for proper unicode char conversion, see #65
  n.indices = string([]byte{n.path[i]})
  n.path = path[:i]
  n.handlers = nil
  n.wildChild = false
  n.fullPath = fullPath[:parentFullPathIndex+i]
}
```

当公共前缀的长度小于 `n.path` 时, 当前节点就会分裂出一个子节点.

比如, 当前节点 `node.path = "/ping"`, 遇到 `path = "/pong"` 时就会分裂,
公共前缀的长度 `i=2`, 因此节点会分裂为 `node.path = "/p"` 和 `node.path = "ing"`.
分裂出来的后一个节点会占据当前节点的大部分属性.

接着看第四步, 如何为当前节点添加一个子节点, 是 `root.addRoute(path, handlers)` 的核心代码.

这也是一个 if 判断, 让我们先看一下后半部分, 即可能出错时的情况:

```go
else if i == len(path) { // Make node a (in-path) leaf
  if n.handlers != nil {
    panic("handlers are already registered for path '" + fullPath + "'")
  }
  n.handlers = handlers
}
```

如果 handlers 不为空, 就会发生错误, 这说明 handlers 只被允许注册一次.

看一下 if 的前半部分, 即 `if i < len(path)` 时的情况:

```go
// Make new node a child of this node
if i < len(path) {
  path = path[i:]

  if n.wildChild {
    parentFullPathIndex += len(n.path)
    n = n.children[0]
    n.priority++

    // Update maxParams of the child node
    if numParams > n.maxParams {
      n.maxParams = numParams
    }
    numParams--

    // Check if the wildcard matches
    if len(path) >= len(n.path) && n.path == path[:len(n.path)] {
      // check for longer wildcard, e.g. :name and :names
      if len(n.path) >= len(path) || path[len(n.path)] == '/' {
        continue walk
      }
    }

    pathSeg := path
    if n.nType != catchAll {
      pathSeg = strings.SplitN(path, "/", 2)[0]
    }
    prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
    panic("'" + pathSeg +
      "' in new path '" + fullPath +
      "' conflicts with existing wildcard '" + n.path +
      "' in existing prefix '" + prefix +
      "'")
  }

  c := path[0]

  // slash after param
  if n.nType == param && c == '/' && len(n.children) == 1 {
    parentFullPathIndex += len(n.path)
    n = n.children[0]
    n.priority++
    continue walk
  }

  // Check if a child with the next path byte exists
  for i := 0; i < len(n.indices); i++ {
    if c == n.indices[i] {
      parentFullPathIndex += len(n.path)
      i = n.incrementChildPrio(i)
      n = n.children[i]
      continue walk
    }
  }

  // Otherwise insert it
  if c != ':' && c != '*' {
    // []byte for proper unicode char conversion, see #65
    n.indices += string([]byte{c})
    child := &node{
      maxParams: numParams,
      fullPath:  fullPath,
    }
    n.children = append(n.children, child)
    n.incrementChildPrio(len(n.indices) - 1)
    n = child
  }
  n.insertChild(numParams, path, fullPath, handlers)
  return

}
```

这一部分也是有点长, 也需要一步步拆解来看.

首先根据 `path = path[i:]`, 发现 path 已经去除了公共前缀部分了.

先看一下第一个判断, `if n.wildChild`, 即存在通配符子节点:

```go
if n.wildChild {
  parentFullPathIndex += len(n.path)
  n = n.children[0]
  n.priority++

  // Update maxParams of the child node
  if numParams > n.maxParams {
    n.maxParams = numParams
  }
  numParams--

  // Check if the wildcard matches
  if len(path) >= len(n.path) && n.path == path[:len(n.path)] {
    // check for longer wildcard, e.g. :name and :names
    if len(n.path) >= len(path) || path[len(n.path)] == '/' {
      continue walk
    }
  }

  pathSeg := path
  if n.nType != catchAll {
    pathSeg = strings.SplitN(path, "/", 2)[0]
  }
  prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
  panic("'" + pathSeg +
    "' in new path '" + fullPath +
    "' conflicts with existing wildcard '" + n.path +
    "' in existing prefix '" + prefix +
    "'")
}
```

通配符的判断中, 一般都是触发通配符冲突错误的, 除非前面通配符部分一样, 后面有 `/`.

```go
c := path[0]

// slash after param
if n.nType == param && c == '/' && len(n.children) == 1 {
  parentFullPathIndex += len(n.path)
  n = n.children[0]
  n.priority++
  continue walk
}
```

当节点是 `:` 通配符且 path 开头为 `/` 后, 进入到新一轮的循环中.

```go
// Check if a child with the next path byte exists
for i := 0; i < len(n.indices); i++ {
  if c == n.indices[i] {
    parentFullPathIndex += len(n.path)
    i = n.incrementChildPrio(i)
    n = n.children[i]
    continue walk
  }
}
```

检查是否存在一个孩子节点, 如果有的话就直接跳到那个节点, 然后进入新一轮的循环中.
前面节点分裂的时候, 设置了 `n.indices = string([]byte{n.path[i]})`.

```go
// Otherwise insert it
if c != ':' && c != '*' {
  // []byte for proper unicode char conversion, see #65
  n.indices += string([]byte{c})
  child := &node{
    maxParams: numParams,
    fullPath:  fullPath,
  }
  n.children = append(n.children, child)
  n.incrementChildPrio(len(n.indices) - 1)
  n = child
}
```

经过了前面的判断之后, 走到这里, 如果 c 不是 `:` 或 `*`, 就会插入一个节点, 并替换当前节点为这个节点.

```go
n.insertChild(numParams, path, fullPath, handlers)
return
```

最后依旧是调用 `insertChild`. 然后终于可以使用 return 跳出循环, 结束整个方法了.

### insertChild

上面在两个地方调用了 `n.insertChild(numParams, path, fullPath, handlers)`, 看一下它的实现.

```go
func (n *node) insertChild(numParams uint8, path string, fullPath string, handlers HandlersChain) {
	var offset int // already handled bytes of the path

	// find prefix until first wildcard (beginning with ':' or '*')
	for i, max := 0, len(path); numParams > 0; i++ {
		c := path[i]
		if c != ':' && c != '*' {
			continue
		}

		// find wildcard end (either '/' or path end)
		end := i + 1
		for end < max && path[end] != '/' {
			switch path[end] {
			// the wildcard name must not contain ':' and '*'
			case ':', '*':
				panic("only one wildcard per path segment is allowed, has: '" +
					path[i:] + "' in path '" + fullPath + "'")
			default:
				end++
			}
		}

		// check if this Node existing children which would be
		// unreachable if we insert the wildcard here
		if len(n.children) > 0 {
			panic("wildcard route '" + path[i:end] +
				"' conflicts with existing children in path '" + fullPath + "'")
		}

		// check if the wildcard has a name
		if end-i < 2 {
			panic("wildcards must be named with a non-empty name in path '" + fullPath + "'")
		}

		if c == ':' { // param
			// split path at the beginning of the wildcard
			if i > 0 {
				n.path = path[offset:i]
				offset = i
			}

			child := &node{
				nType:     param,
				maxParams: numParams,
				fullPath:  fullPath,
			}
			n.children = []*node{child}
			n.wildChild = true
			n = child
			n.priority++
			numParams--

			// if the path doesn't end with the wildcard, then there
			// will be another non-wildcard subpath starting with '/'
			if end < max {
				n.path = path[offset:end]
				offset = end

				child := &node{
					maxParams: numParams,
					priority:  1,
					fullPath:  fullPath,
				}
				n.children = []*node{child}
				n = child
			}

		} else { // catchAll
			if end != max || numParams > 1 {
				panic("catch-all routes are only allowed at the end of the path in path '" + fullPath + "'")
			}

			if len(n.path) > 0 && n.path[len(n.path)-1] == '/' {
				panic("catch-all conflicts with existing handle for the path segment root in path '" + fullPath + "'")
			}

			// currently fixed width 1 for '/'
			i--
			if path[i] != '/' {
				panic("no / before catch-all in path '" + fullPath + "'")
			}

			n.path = path[offset:i]

			// first node: catchAll node with empty path
			child := &node{
				wildChild: true,
				nType:     catchAll,
				maxParams: 1,
				fullPath:  fullPath,
			}
			n.children = []*node{child}
			n.indices = string(path[i])
			n = child
			n.priority++

			// second node: node holding the variable
			child = &node{
				path:      path[i:],
				nType:     catchAll,
				maxParams: 1,
				handlers:  handlers,
				priority:  1,
				fullPath:  fullPath,
			}
			n.children = []*node{child}

			return
		}
	}

	// insert remaining path part and handle to the leaf
	n.path = path[offset:]
	n.handlers = handlers
	n.fullPath = fullPath
}
```

折叠一下代码, 主要是两部分, 一个 for 循环, 以及一些更新属性的语句.

```go
// insert remaining path part and handle to the leaf
n.path = path[offset:]
n.handlers = handlers
n.fullPath = fullPath
```

主要看一下 for 循环:

```go
// find prefix until first wildcard (beginning with ':' or '*')
for i, max := 0, len(path); numParams > 0; i++ {
  c := path[i]
  if c != ':' && c != '*' {
    continue
  }
```

这几行判断, 如同注释说明的那般, 直到遇到通配符字符 `':' or '*'` 才开始真正处理.
注意判断条件是 `numParams`, 这个参数指明了有几个通配符参数.

```go
// find wildcard end (either '/' or path end)
end := i + 1
for end < max && path[end] != '/' {
  switch path[end] {
  // the wildcard name must not contain ':' and '*'
  case ':', '*':
    panic("only one wildcard per path segment is allowed, has: '" +
      path[i:] + "' in path '" + fullPath + "'")
  default:
    end++
  }
		}
```

这也是个判断, 用于验证通配符名字中不能出现多个 `':' and '*'`.

```go
// check if this Node existing children which would be
// unreachable if we insert the wildcard here
if len(n.children) > 0 {
  panic("wildcard route '" + path[i:end] +
    "' conflicts with existing children in path '" + fullPath + "'")
}

// check if the wildcard has a name
if end-i < 2 {
  panic("wildcards must be named with a non-empty name in path '" + fullPath + "'")
}
```

又是两个判断, 第一个用于验证当前 node 不能存储子节点, 否则通配符节点就冲突了.
第二个用于验证通配符节点的名字必须有长度, 至少要有一个字符.

最后是根据通配符的不同分别构造, 先看一下 `c == ':'` 时的代码:

```go
if c == ':' { // param
  // split path at the beginning of the wildcard
  if i > 0 {
    n.path = path[offset:i]
    offset = i
  }

  child := &node{
    nType:     param,
    maxParams: numParams,
    fullPath:  fullPath,
  }
  n.children = []*node{child}
  n.wildChild = true
  n = child
  n.priority++
  numParams--

  // if the path doesn't end with the wildcard, then there
  // will be another non-wildcard subpath starting with '/'
  if end < max {
    n.path = path[offset:end]
    offset = end

    child := &node{
      maxParams: numParams,
      priority:  1,
      fullPath:  fullPath,
    }
    n.children = []*node{child}
    n = child
  }

}
```

然后是 `c == '*'` 时的代码, 也就是 else 部分:

```go
else { // catchAll
  if end != max || numParams > 1 {
    panic("catch-all routes are only allowed at the end of the path in path '" + fullPath + "'")
  }

  if len(n.path) > 0 && n.path[len(n.path)-1] == '/' {
    panic("catch-all conflicts with existing handle for the path segment root in path '" + fullPath + "'")
  }

  // currently fixed width 1 for '/'
  i--
  if path[i] != '/' {
    panic("no / before catch-all in path '" + fullPath + "'")
  }

  n.path = path[offset:i]

  // first node: catchAll node with empty path
  child := &node{
    wildChild: true,
    nType:     catchAll,
    maxParams: 1,
    fullPath:  fullPath,
  }
  n.children = []*node{child}
  n.indices = string(path[i])
  n = child
  n.priority++

  // second node: node holding the variable
  child = &node{
    path:      path[i:],
    nType:     catchAll,
    maxParams: 1,
    handlers:  handlers,
    priority:  1,
    fullPath:  fullPath,
  }
  n.children = []*node{child}

  return
}
```

`catchAll` 通配符有点特殊, 这个通配符后面是不允许出现其他通配符参数的, 所以前几行都在判断要求是否符合.
这个过程中会创建两个类型为 `catchAll` 的节点, 第一个节点指示存储通配符子节点, 即`wildChild=true`,
第二个节点会占有具体的内容.

添加路由的过程基本上就是这样, 接下来看一下如何读取数据.

## 获取数据

从树中获取数据, 主要发生在 `func (engine *Engine) handleHTTPRequest(c *Context)` 中.
看一下代码片段:

```go
root := t[i].root
// Find route in tree
value := root.getValue(rPath, c.Params, unescape)
if value.handlers != nil {
  c.handlers = value.handlers
  c.Params = value.params
  c.fullPath = value.fullPath
  c.Next()
  c.writermem.WriteHeaderNow()
  return
}
```

主要是通过 `getValue` 方法获取数据的, 完整代码如下:

```go
// getValue returns the handle registered with the given path (key). The values of
// wildcards are saved to a map.
// If no handle can be found, a TSR (trailing slash redirect) recommendation is
// made if a handle exists with an extra (without the) trailing slash for the
// given path.
func (n *node) getValue(path string, po Params, unescape bool) (value nodeValue) {
	value.params = po
walk: // Outer loop for walking the tree
	for {
		if len(path) > len(n.path) {
			if path[:len(n.path)] == n.path {
				path = path[len(n.path):]
				// If this node does not have a wildcard (param or catchAll)
				// child,  we can just look up the next child node and continue
				// to walk down the tree
				if !n.wildChild {
					c := path[0]
					for i := 0; i < len(n.indices); i++ {
						if c == n.indices[i] {
							n = n.children[i]
							continue walk
						}
					}

					// Nothing found.
					// We can recommend to redirect to the same URL without a
					// trailing slash if a leaf exists for that path.
					value.tsr = path == "/" && n.handlers != nil
					return
				}

				// handle wildcard child
				n = n.children[0]
				switch n.nType {
				case param:
					// find param end (either '/' or path end)
					end := 0
					for end < len(path) && path[end] != '/' {
						end++
					}

					// save param value
					if cap(value.params) < int(n.maxParams) {
						value.params = make(Params, 0, n.maxParams)
					}
					i := len(value.params)
					value.params = value.params[:i+1] // expand slice within preallocated capacity
					value.params[i].Key = n.path[1:]
					val := path[:end]
					if unescape {
						var err error
						if value.params[i].Value, err = url.QueryUnescape(val); err != nil {
							value.params[i].Value = val // fallback, in case of error
						}
					} else {
						value.params[i].Value = val
					}

					// we need to go deeper!
					if end < len(path) {
						if len(n.children) > 0 {
							path = path[end:]
							n = n.children[0]
							continue walk
						}

						// ... but we can't
						value.tsr = len(path) == end+1
						return
					}

					if value.handlers = n.handlers; value.handlers != nil {
						value.fullPath = n.fullPath
						return
					}
					if len(n.children) == 1 {
						// No handle found. Check if a handle for this path + a
						// trailing slash exists for TSR recommendation
						n = n.children[0]
						value.tsr = n.path == "/" && n.handlers != nil
					}

					return

				case catchAll:
					// save param value
					if cap(value.params) < int(n.maxParams) {
						value.params = make(Params, 0, n.maxParams)
					}
					i := len(value.params)
					value.params = value.params[:i+1] // expand slice within preallocated capacity
					value.params[i].Key = n.path[2:]
					if unescape {
						var err error
						if value.params[i].Value, err = url.QueryUnescape(path); err != nil {
							value.params[i].Value = path // fallback, in case of error
						}
					} else {
						value.params[i].Value = path
					}

					value.handlers = n.handlers
					value.fullPath = n.fullPath
					return

				default:
					panic("invalid node type")
				}
			}
		} else if path == n.path {
			// We should have reached the node containing the handle.
			// Check if this node has a handle registered.
			if value.handlers = n.handlers; value.handlers != nil {
				value.fullPath = n.fullPath
				return
			}

			if path == "/" && n.wildChild && n.nType != root {
				value.tsr = true
				return
			}

			// No handle found. Check if a handle for this path + a
			// trailing slash exists for trailing slash recommendation
			for i := 0; i < len(n.indices); i++ {
				if n.indices[i] == '/' {
					n = n.children[i]
					value.tsr = (len(n.path) == 1 && n.handlers != nil) ||
						(n.nType == catchAll && n.children[0].handlers != nil)
					return
				}
			}

			return
		}

		// Nothing found. We can recommend to redirect to the same URL with an
		// extra trailing slash if a leaf exists for that path
		value.tsr = (path == "/") ||
			(len(n.path) == len(path)+1 && n.path[len(path)] == '/' &&
				path == n.path[:len(n.path)-1] && n.handlers != nil)
		return
	}
}
```

代码有点长, 先读一下注释. 主要是根据路径和参数, 获取注册在上面的 handlers.

```go
// nodeValue holds return values of (*Node).getValue method
type nodeValue struct {
	handlers HandlersChain
	params   Params
	tsr      bool
	fullPath string
}

// Param is a single URL parameter, consisting of a key and a value.
type Param struct {
	Key   string
	Value string
}
```

里面用到的结构体如上. 方法的主体部分是一个 for 循环.

for 循环里面, 前半部分是一个判断, 先看一下后半部分.

```go
// Nothing found. We can recommend to redirect to the same URL with an
// extra trailing slash if a leaf exists for that path
value.tsr = (path == "/") ||
  (len(n.path) == len(path)+1 && n.path[len(path)] == '/' &&
    path == n.path[:len(n.path)-1] && n.handlers != nil)
return
```

如果没有找到对应的匹配, 会设置一个叫做 `tsr` 的标识, 用于判断是否符合 `TSR (trailing slash redirect)`, 即尾部斜杆重定向. 比如 `/path` 可以重定向到 `/path/`.

回到 if 判断上来, 先看第一个判断部分, 即 `if len(path) > len(n.path)`.

```go
if len(path) > len(n.path) {
  if path[:len(n.path)] == n.path {
    path = path[len(n.path):]
    // If this node does not have a wildcard (param or catchAll)
    // child,  we can just look up the next child node and continue
    // to walk down the tree
    if !n.wildChild {
      c := path[0]
      for i := 0; i < len(n.indices); i++ {
        if c == n.indices[i] {
          n = n.children[i]
          continue walk
        }
      }

      // Nothing found.
      // We can recommend to redirect to the same URL without a
      // trailing slash if a leaf exists for that path.
      value.tsr = path == "/" && n.handlers != nil
      return
    }

    // handle wildcard child
    n = n.children[0]
    switch n.nType {
    case param:
      // find param end (either '/' or path end)
      end := 0
      for end < len(path) && path[end] != '/' {
        end++
      }

      // save param value
      if cap(value.params) < int(n.maxParams) {
        value.params = make(Params, 0, n.maxParams)
      }
      i := len(value.params)
      value.params = value.params[:i+1] // expand slice within preallocated capacity
      value.params[i].Key = n.path[1:]
      val := path[:end]
      if unescape {
        var err error
        if value.params[i].Value, err = url.QueryUnescape(val); err != nil {
          value.params[i].Value = val // fallback, in case of error
        }
      } else {
        value.params[i].Value = val
      }

      // we need to go deeper!
      if end < len(path) {
        if len(n.children) > 0 {
          path = path[end:]
          n = n.children[0]
          continue walk
        }

        // ... but we can't
        value.tsr = len(path) == end+1
        return
      }

      if value.handlers = n.handlers; value.handlers != nil {
        value.fullPath = n.fullPath
        return
      }
      if len(n.children) == 1 {
        // No handle found. Check if a handle for this path + a
        // trailing slash exists for TSR recommendation
        n = n.children[0]
        value.tsr = n.path == "/" && n.handlers != nil
      }

      return

    case catchAll:
      // save param value
      if cap(value.params) < int(n.maxParams) {
        value.params = make(Params, 0, n.maxParams)
      }
      i := len(value.params)
      value.params = value.params[:i+1] // expand slice within preallocated capacity
      value.params[i].Key = n.path[2:]
      if unescape {
        var err error
        if value.params[i].Value, err = url.QueryUnescape(path); err != nil {
          value.params[i].Value = path // fallback, in case of error
        }
      } else {
        value.params[i].Value = path
      }

      value.handlers = n.handlers
      value.fullPath = n.fullPath
      return

    default:
      panic("invalid node type")
    }
  }
}
```

这部分的判断里嵌套了一个 if 判断, 用于判断路径的前缀和当前节点的 path 相符, 如果不相等就直接跳过.

然后是根据 `n.wildChild` 判断, 即基于是否有通配符子节点.

如果没有通配符子节点, 会继续查找下一个子节点, 然后进行新一轮的 for 循环.
如果找不到子节点, 就直接返回了. 是否存在子节点是根据 `n.indices` 判断的,
`n.indices` 是个字符串, 保存了所有子节点路径的第一个字符.
比如, 当前注册了两个路径, `/ping` 和 `/pong`, 那么当前的节点就是 `/p` 公共前缀,
然后它的 `n.indices="io"`.

如果存在通配符子节点, 就会根据 `n.nType` 的类型进行选择处理.

如果类型是 `param`, 即使用 `:` 命名的变量, 就会先保存那个变量的值.
如果长度还有剩余 `if end < len(path) {`, 就会进入到新一个 for 循环中;
否认就认为是结束了, 将 `handlers` 和 `fullPath` 复制一下就行了.

如果类型是 `catchAll`, 即使用 `*` 命令的任意匹配变量, 处理就比较简单了,
因为不用考虑后面还有路径的问题, `*` 会匹配所有剩余的 path 路径.
直接保存变量值, 然后将 `handlers` 和 `fullPath` 复制一下就行了.

如果类型不符合上述的两种类型, 就会触发 panic.

接着看另一个判断, 即 `else if path == n.path`.

```go
else if path == n.path {
  // We should have reached the node containing the handle.
  // Check if this node has a handle registered.
  if value.handlers = n.handlers; value.handlers != nil {
    value.fullPath = n.fullPath
    return
  }

  if path == "/" && n.wildChild && n.nType != root {
    value.tsr = true
    return
  }

  // No handle found. Check if a handle for this path + a
  // trailing slash exists for trailing slash recommendation
  for i := 0; i < len(n.indices); i++ {
    if n.indices[i] == '/' {
      n = n.children[i]
      value.tsr = (len(n.path) == 1 && n.handlers != nil) ||
        (n.nType == catchAll && n.children[0].handlers != nil)
      return
    }
  }

  return
}
```

这部分的处理也是比较简单的, 和前面的逻辑类似, 主要是看路径上是否有 handler 注册.
如果没有 handler 注册, 就会检查 `value.tsr` 的值, 是否属于尾部斜杆重定向.

由此, 从树中获取数据的过程也已经看完了.

## 总结

优秀的代码还是要多读读的, 即有助于理解原理, 又能开阔自己的视野.
另外一点, 读代码的时候调试器真的是非常有用, 尤其是观察数据结构是怎么存储的.

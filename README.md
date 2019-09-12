# unist-sitter
**This repo is not yet a thing**, but I hope that it will be... soon.

What the heck does "unist-sitter" mean? It's a marriage of two awesome pieces of software:

- [tree-sitter] is GitHub's modern "parser generator tool and incremental parsing library" that can be used to parse [dozens of languages and grammars](https://tree-sitter.github.io/tree-sitter/#available-parsers) into syntax tree data structures.
- [unist] is a "universal syntax tree" spec that allows you to navigate (and manipulate!) abstract syntax trees with [dozens of awesome and well-tested utilities](https://github.com/syntax-tree/unist#list-of-utilities).

Basically, it should be possible to do this:

```js
const TreeSitter = require('tree-sitter')
const Ruby = require('tree-sitter-ruby')
const {UnistNode} = require('unist-sitter')
const {select} = require('unist-util-select')

const parser = new TreeSitter()
parser.setLanguage(Ruby)

const tree = await parser.parse(`puts "yo #{ENV['DOG']}"`)
const unistTree = UnistNode(tree.rootNode)
console.log(select(unistTree, 'interpolation'))
```

## Syntax nodes
The syntax node API would need only be a single function that either:

1. Converts a tree-sitter [SyntaxNode] object into a [unist Node]. For instance:

    ```js
    function UnistNode(from) {
      const {type, children = []} = from
      return {
        type,
        position: createUnistPosition(from),
        children: children.map(sitterNodeToUnist)
      }
    }
    ```
    
    If references to the "original" nodes weren't preserved, the resulting object model could be smaller in memory and more easily serializable.

2. Creates a wrapper (proxy) for the SyntaxNode, exposing the [unist Node] interface (a la [unist-util-parents](https://github.com/syntax-tree/unist-util-parents)). This might be more memory-hungry, but it would also allow for editing of the underlying tree-sitter nodes while preserving the ability to navigate the tree with unist utilities.

    Wrapping offers more opportunities for dynamic getters that make it easier to form CSS-like selectors that match syntax nodes, e.g. by identifier:
    
    ```js
    class UnistNode {
      // instead of: select('method_call:has(:scope > identifier[text=foo])'
      // you can do: select('method_call[identifier=foo]')
      get identifier() {
        const node = select(this, ':has(:scope > identifier')
        return node ? node.text : undefined
      }
    }
    ```

## File objects
At the simplest level, this would map all of the information necessary to create a unist-sitter parse tree from a single file to unified's [vfile] object model. Something like:

```js
const {FileNode} = require('unist-sitter')

const foo = FileNode({path: 'foo.rb'})
```

This would makes it possible to use [lots of cool utilities](https://github.com/vfile/vfile#utilities), like [vfile-find-down](https://github.com/vfile/vfile-find-down) to traverse the file system.

Presumably, they would also be easy to read from the filesystem and parse with a given tree-sitter grammar:

```js
const Ruby = require('tree-sitter-ruby')
await foo.read() // sets .contents
await foo.parse(Ruby) // sets .syntax ?
```

## Directory objects
This is where things get kind of wild. Now imagine a unist tree structure that represents _both the file system **and** file-level syntax trees_. Suddenly, searching for an ambiguously named method call becomes a whole lot easier:

```js
const {DirectoryNode} = require('unist-sitter')
const visit = require('unist-util-visit')

const root = DirectoryNode(process.cwd())
visit(root, 'file[lang=ruby] method_call:has(:host > identifier[text=get])', node => {
})
```

What sorcery is this? Well, if a directory node lazily evaluates its `children` property...

```js
const {readdirSync} = require('fs')
const {join} = require('path')

class DirectoryNode extends VFile {
  get type() { return 'directory' }
  
  get children() {
    return readdirSync(this.path, {withFileTypes: true})
      .map(dirent => {
        const path = join(this.path, dirent.name)
        return dirent.isDirectory()
          ? DirectoryNode({path})
          : dirent.isFile() ? FileNode({path}) : null
      })
      .filter(Boolean)
  }
}
```

and a file node does the same (assuming some knowledge of how filename extensions map to parser grammars)...

```js
class FileNode extends VFile {
  get type() { return 'file' }
  
  parse() {
    const parser = getParser(this.lang)
    const tree = parser.parse(this.contents)
    return UnistNode(tree.rootNode)
  }

  // returns "rb" if the filename ends with ".rb", etc.
  get lang() {
    return this.extname.replace(/^\./, '')
  }  

  get children() {
    return [this.parse()]
  }  
}
```

Implementing custom node types for different languages would super-charge the unist utilities and allow you to do things like parse ERB as _both_ HTML and Ruby:

```js
const ERB = getParser('embedded-template')
const HTML = getParser('html')
const Ruby = getParser('ruby')

class ERBNode extends FileNode {
  get lang() {
    return 'erb'
  }
  
  parse() {
    const tree = ERB.parse(this.contents)
    const html = selectAll(tree, 'output').map(node => node.text).join('')
    const ruby = selectAll(tree, 'directive').map(erbDirectiveToRuby).join('\n')
    return {
      html: HTML.parse(html),
      ruby: Ruby.parse(ruby)
    }
  }
  
  get children() {
    const {html, ruby} = this.parse()
    return [html, ruby]
  }
}
```

Anyway, more soon! :rocket:


[tree-sitter]: https://tree-sitter.github.io/tree-sitter/
[unist]: https://github.com/syntax-tree/unist
[SyntaxNode]: https://github.com/tree-sitter/node-tree-sitter/blob/32df455d7b257428b02b60a0af9857451892c801/tree-sitter.d.ts#L41-L85
[unist Node]: https://github.com/syntax-tree/unist#node
[vfile]: https://github.com/vfile/vfile#intro

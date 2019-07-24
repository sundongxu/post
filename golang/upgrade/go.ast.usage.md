---
title: "初步了解使用 go AST 知识"
date: 2019-07-24T23:18:33+08:00
categories:
    - Golang
tags: 
    - Golang
    - ast
---

最近遇到一个需求需要解析 Golang 项目中的 interface 并且自动生成 gomock 相关的文件。

第一个版本简单粗暴，但是我这只菜鸡不会写 shell :(

```
sed -n 's/^type \([A-Z][^[:space:]]*\) interface .*/\1/p' "interfaces.go"
```

第二个版本: 使用 go AST 解析 go file 自动生成，轮子可以看 [代码](https://github.com/zhaolion/gengo/tree/master/cmd/gomock)

下面进入正题，什么是 AST，这个包使用的背景和基本知识有哪些，请先看看以下文档:

* [go/ast - goc](https://godoc.org/go/ast)
* [AST wiki](https://en.wikipedia.org/wiki/Abstract_syntax_tree)

## AST 是什么？

AST 是 抽象语法树（Abstract Syntax Tree）缩写，在计算机科学中，是源代码语法结构的一种抽象表示。它以树状的形式表现编程语言的语法结构，树上的每个节点都表示源代码中的一种结构。之所以说语法是“抽象”的，是因为这里的语法并不会表示出真实语法中出现的每个细节。比如，嵌套括号被隐含在树的结构中，并没有以节点的形式呈现；而类似于 if-condition-then 这样的条件跳转语句，可以使用带有两个分支的节点来表示。

当我们对源代码语法分析时，其实我们是在程序设计语言的语法规则指导下才能进行的。我们使用语法规则描述该语言的各种语法成分的是如何组成的，通常用前后文无关文法或与之等价的Backus-Naur 范式(BNF) 将相应程序设计语言的语法规则准确的描述出来。

了解 Go 对应的语法规则，就能对 Go 代码进行语法分析，当然完成最上面的需求就很简单了

## Go 的 AST 是什么？
一般来说，需要词法分析，然后进行语法分析。Go 的 AST library 能够将 `Go 文件` 或者 `包含 Go 文件目录` 作为输入，使用 Scaner 分析生成 token，然后将 token 送进 Parser 变成语法树

在 Go 中，每个 token 都以它所处的位置，类型和原始字面量来表示

看下官网的 demo:

code:

```
// src is the input for which we want to print the AST.
src := `
package main
func main() {
	println("Hello, World!")
}
`

// Create the AST by parsing src.
fset := token.NewFileSet() // positions are relative to fset
f, err := parser.ParseFile(fset, "", src, 0)
if err != nil {
    panic(err)
}

// Print the AST.
ast.Print(fset, f)
```

Output:

```
     0  *ast.File {
     1  .  Package: 2:1
     2  .  Name: *ast.Ident {
     3  .  .  NamePos: 2:9
     4  .  .  Name: "main"
     5  .  }
     6  .  Decls: []ast.Decl (len = 1) {
     7  .  .  0: *ast.FuncDecl {
     8  .  .  .  Name: *ast.Ident {
     9  .  .  .  .  NamePos: 3:6
    10  .  .  .  .  Name: "main"
    11  .  .  .  .  Obj: *ast.Object {
    12  .  .  .  .  .  Kind: func
    13  .  .  .  .  .  Name: "main"
    14  .  .  .  .  .  Decl: *(obj @ 7)
    15  .  .  .  .  }
    16  .  .  .  }
    17  .  .  .  Type: *ast.FuncType {
    18  .  .  .  .  Func: 3:1
    19  .  .  .  .  Params: *ast.FieldList {
    20  .  .  .  .  .  Opening: 3:10
    21  .  .  .  .  .  Closing: 3:11
    22  .  .  .  .  }
    23  .  .  .  }
    24  .  .  .  Body: *ast.BlockStmt {
    25  .  .  .  .  Lbrace: 3:13
    26  .  .  .  .  List: []ast.Stmt (len = 1) {
    27  .  .  .  .  .  0: *ast.ExprStmt {
    28  .  .  .  .  .  .  X: *ast.CallExpr {
    29  .  .  .  .  .  .  .  Fun: *ast.Ident {
    30  .  .  .  .  .  .  .  .  NamePos: 4:2
    31  .  .  .  .  .  .  .  .  Name: "println"
    32  .  .  .  .  .  .  .  }
    33  .  .  .  .  .  .  .  Lparen: 4:9
    34  .  .  .  .  .  .  .  Args: []ast.Expr (len = 1) {
    35  .  .  .  .  .  .  .  .  0: *ast.BasicLit {
    36  .  .  .  .  .  .  .  .  .  ValuePos: 4:10
    37  .  .  .  .  .  .  .  .  .  Kind: STRING
    38  .  .  .  .  .  .  .  .  .  Value: "\"Hello, World!\""
    39  .  .  .  .  .  .  .  .  }
    40  .  .  .  .  .  .  .  }
    41  .  .  .  .  .  .  .  Ellipsis: -
    42  .  .  .  .  .  .  .  Rparen: 4:25
    43  .  .  .  .  .  .  }
    44  .  .  .  .  .  }
    45  .  .  .  .  }
    46  .  .  .  .  Rbrace: 5:1
    47  .  .  .  }
    48  .  .  }
    49  .  }
    50  .  Scope: *ast.Scope {
    51  .  .  Objects: map[string]*ast.Object (len = 1) {
    52  .  .  .  "main": *(obj @ 11)
    53  .  .  }
    54  .  }
    55  .  Unresolved: []*ast.Ident (len = 1) {
    56  .  .  0: *(obj @ 29)
    57  .  }
    58  }
```

上面结果可以使用 [goast-viewer](https://yuroyoro.github.io/goast-viewer/index.html) 自己查看

最后用图来更好的理解:

![ast.tree.demo](https://i.imgur.com/C2wPSiL.png)

总结一下，Go 将所有可以识别的 token 抽象成 Node，通过 interface 方式组织在一起，它们之间的关系如下图示意

![golang_ast_relation](https://i.imgur.com/i1a9ojJ.png)

## 通过 AST 找出所有的 interface

大概理解完 AST 组织结构，所以想要找出所有的 interface，可以顺着下面结构找

FileSet -> Package -> Files -> File.Scope.Objects -> obj.Decl -> ast.TypeSpec.Type -> ast.InterfaceType

核心代码

```go
func (ff *lookup) find(base, dir string) {
	fs := token.NewFileSet()
	pkgs, err := parser.ParseDir(fs, dir, func(info os.FileInfo) bool {
		return true
	}, parser.AllErrors)
	if err != nil {
		panic(err)
	}

	interfaces := make([]*ityp, 0, 16)
	for p, pkg := range pkgs {
		for fileName, file := range pkg.Files {
			for _, obj := range file.Scope.Objects {
				typ, ok := obj.Decl.(*ast.TypeSpec)
				if !ok {
					continue
				}

				_, ok = typ.Type.(*ast.InterfaceType)
				if !ok {
					continue
				}

				if strings.ToLower(string(obj.Name[0])) != string(obj.Name[0]) {
					interfaces = append(interfaces, &ityp{
						packageName: p,
						importPath:  filepath.Join(base, filepath.Dir(fileName)),
						name:        obj.Name,
					})
				}
			}
		}
	}

	ff.append(interfaces)
}
```
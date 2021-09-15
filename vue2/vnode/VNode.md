# VNode格式的设计

真正编译的地方

```js
var createCompiler = createCompilerCreator(function baseCompile (
    template,
    options
) {
    var ast = parse(template.trim(), options);
    if (options.optimize !== false) {
      optimize(ast, options);
    }
    var code = generate(ast, options);
    return {
      ast: ast,
      render: code.render,
      staticRenderFns: code.staticRenderFns
    }
  });
```

这个过程三步:
+ 1.parse生成ast(先不管)
+ 2.optimize优化vnode: optimize 我们把整个 AST 树中的每一个 AST 元素节点标记了 static 和 staticRoot (静态节点与静态根节点)
+ 3.generate生成代码
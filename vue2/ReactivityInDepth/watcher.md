## watcher类分析

- 渲染Watcher：变量修改时，负责通知HTML里的重新渲染
- computed Watcher：变量修改时，负责通知computed里依赖此变量的computed属性变量的修改
- user Watcher：变量修改时，负责通知watch属性里所对应的变量函数的执行



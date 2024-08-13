## asan 的几个主要功能
1. 内存泄漏检查
2. use after free
3. ...

## 使用asan 排查use after free

gcc  指定fsanitize option 即可打开asan 

```
gcc -fsanitize=address -g your_program.c -o your_program
```

## asan 的结果分析

# cpp 学习比较

## 1. 右值和右值引用的理解
1. 什么是右值？ 能取地址的是左值，不能取地址的是右值，典型的右值是临时变量，表达式，常量，函数返回值
2. 什么时候用右值引用？常见的解决返回局部对象拷贝问题，string dst = move(src.to_string()),  将函数返回值直接赋值给新对象
3. 移动构造函数长啥样？参数是&&
4. 什么时候触发移动构造函数？函数内部调用move return，或者对表达式调用move 的时候触发移动构造函数
   

参考资料：https://www.cnblogs.com/david-china/p/17080072.html

## 2. 智能指针的理解
1. unique 指针
2. share 指针
3. weak 指针

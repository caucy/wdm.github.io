# nginx编译成静态库写单测

## nginx 编译成静态库的步骤

1. replace main 函数

2. 编译所有.c 文件
./configure & make 将生成所有.o 文件到obj/src 下

3. 将.o 文件合并成.a 文件
ar -rc libnginx.a `find objs -name *.o`

ranlib libfoo.a  # 在更新静态库成员之后，对静态库的成员“__.SYMDEF”进行更新（更新库的符号索引表)

4. 准备头文件，使用静态库
cp `find src -name *.h` /usr/include/nginx/

## 使用nginx 静态库写功能函数

## 使用gtest 写测试用例

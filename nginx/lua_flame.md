1. 安装 systemtap
stap 在 debian 安装略麻烦

安装是否成功
stap -e 'probe begin{printf("Hello, World"); exit();}'
安装合适的 systemtap
从 debian 官方看 systemtap 稳定版本是 4.0， debian 的版本是 9.11 stretch

apt 安装是 3.x

源码安装
step1 卸载原来的 systemtap
apt --purge remove systemtap / systemtap-common / systemtap-runtime / systemtap-std-dev

step2 下载源码
wget xxxx_4.0.tar.gz

step3 编译
apt install libdw-dev

./configure && make && make install

 

step4 检查内盒 debuginfo
dpkg -l|grep debug

关注 linux-image-xxxx, crash, kdump-tools, libc6-dbg libdw1 等

 

2. 编译 luajit
注意开 ccdebug=1，make ccdebug=1 

如果不开 debug，stapxx 将抓不到栈信息

3. 抓火焰图
下载各工具包
# git clone https://github.com/openresty/stapxx.git --depth=1 /opt/stapxx
# export STAP_PLUS_HOME=/opt/stapxx
# export PATH=$STAP_PLUS_HOME:$STAP_PLUS_HOME/samples:$PATH

# stap++ -e 'probe begin { println("hello") exit() }'

hello


# git clone https://github.com/openresty/openresty-systemtap-toolkit.git --depth=1 /opt/openresty-systemtap-toolkit

# git clone https://github.com/brendangregg/FlameGraph.git --depth=1 /opt/FlameGraph
 

绘制火焰图
# /opt/stapxx/samples/lj-lua-stacks.sxx --arg time=120 --skip-badvars -x `ps --no-headers -fC nginx|awk '/worker/  {print$2}'| shuf | head -n 1` > /tmp/tmp.bt （-x 是要抓的进程的 pid， 探测结果输出到 tmp.bt）
# /opt/openresty-systemtap-toolkit/fix-lua-bt tmp.bt > /tmp/flame.bt  (处理 lj-lua-stacks.sxx 的输出，使其可读性更佳)
# /opt/FlameGraph/stackcollapse-stap.pl /tmp/flame.bt > /tmp/flame.cbt
# /opt/FlameGraph/flamegraph.pl /tmp/flame.cbt > /tmp/flame.svg
 

为了突出效果，建议在运行 stap++ 的时候，使用压测工具，以便采集足够的样本

# ab -n 10000 -c 100 -k http://localhost/

4. 如何判断 so 是否有 debuginfo
file xxx.so xxxxxx not stripped 为有 debuginfo

readelf -S xxx |grep debug 也能看出来

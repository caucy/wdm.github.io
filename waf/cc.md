# 防cc 的思路

## 1. 古老手段
1. 源ip 限速，+ js 挑战
2. 区域封禁
3. 七层黑名单，明显特征怎么找出来（人肉找？）
去四层找tls 的cipher，加密套件等，抓包找差别？（肉鸡底层漏洞相同）

## 2. 智能防cc
1. 源ip 限速，怎么发现cc 攻击
2. 源ip 限频，怎么找阈值
3. 怎么判断误封？要有个正常流量基线
4. 清洗流量送检
5. 分析流量，流式计算+监控+报警
6. 建设封禁平台，自动执行规则，给lb 下发403


# PmHub智能项目管理系统

## 项目介绍
基于SpringCloud + SpringCloud Alibaba + Flowable + TTL + Vue技术栈开发的智能项目管理系统。该系统支持自定义项目、上线项目发布、任务分派与执行跟踪、自定义工作流程、企业微信通知集成等功能。管理者可通过ECharts图表实时监控项目和任务进度，实现项目管理流程化和智能化。

## 项目亮点
- 配置Nacos为配置中心和服务注册中心，持久化配置到MySql数据库中，避免了由服务器宕机或重启造成的配置丢失。
- 使用Redis分布式锁，保证流程更新时的顺序执行，避免了由网络或性能原因影响执行顺序。
- 通过AOP特性结合自定义注解，实现了接口鉴权和内部认证。
- 为了保证缓存一致性，采用Cache Aside模式，更新数据后及时清除缓存。
- 使用Sentinel + Gateway实现网关限流，大幅提升单点服务可用性至99%以上，保证高并发情况下的服务可靠性。

## 项目架构



![0c1ca675-e862-4c82-ac1e-c1ad570c257f](C:\Users\ASUS\OneDrive\桌面\0c1ca675-e862-4c82-ac1e-c1ad570c257f.png)

```
1    com.laigeoffer.pmhub
2    |— pmhub-ui                  // 前端框架 [1024]
3    |— pmhub-gateway             // 网关模块 [6880]
4    |— pmhub-auth                // 认证中心 [6800]
5    |— pmhub-api                 // 接口模块
6    |    |— pmhub-api-system                    // 系统接口
7    |    └— pmhub-api-workflow                  // 流程接口
8    |— pmhub-base                // 通用模块
9    |    |— pmhub-base-core                     // 核心模块组件
10   |    |— pmhub-base-datasource               // 多数据源组件
11   |    |— pmhub-base-seata                    // 分布式事务组件
12   |    |— pmhub-base-security                 // 安全模块组件
13   |    |— pmhub-base-swagger                  // 系统接口组件
14   |    └— pmhub-base-notice                   // 消息组件组件
15   |— pmhub-modules             // 业务模块
16   |    |— pmhub-system                        // 系统模块 [6801]
17   |    |— pmhub-gen                           // 代码生成 [6802]
18   |    |— pmhub-job                           // 定时任务 [6803]
19   |    |— pmhub-project                       // 项目服务 [6806]
20   |    └— pmhub-workflow                      // 流程服务 [6808]
21   |— pmhub-monitor             // 监控中心 [6888]
22   └—pom.xml                                   // 公共依赖
```
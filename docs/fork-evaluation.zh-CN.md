# go-tileproxy 代码质量与采用建议

审查日期：2026-10-25。上游 go-tileproxy 基线 7655266ba1d37e6bbe73a1b6eb0de66151be8d08。

结论：算法和功能可供参考，但不建议将完整依赖链作为当前小范围瓦片服务的长期基础。建议实现一个独立、范围明确的 Go 子服务：图源配置、瓦片获取、GCJ-02 与 WGS84 对齐、图像重采样、缓存、XYZ 输出及健康状态。JS 后台继续管理地点、集合、项目；RustGIS 读取本机 XYZ 地址。此次仅做审查、fork 和补丁回馈，没有开始重写，也没有部署正式服务。

## 已确认的问题

1. 嵌套缓存有真实初始化缺陷。service.go 同一次 Go map 遍历内创建和解析缓存；来源尚未创建时会被丢弃。setting/factory.go 用可选 Name 字段重新查 manager，名称省略时可空指针。新回归测试用本机 HTTP PNG 源、24 次构建、三级缓存、实际像素验证；上游原始代码在独立基线中崩溃，修复后通过。
2. 错误被隐藏。cache/layer.go 在 LoadTileCoords 和 getSource 两处忽略 error，service/tile.go 将 LoadTileCoord 的错误转换为普通空白图。该行为使空覆盖、坐标配置错误和上游故障难以区分。试接时遇到空白瓦片后需要额外诊断日志才能定位，不能把 HTTP 200 当作出图成功。
3. 缓存写入不够稳妥。cache/local_cache.go 的 StoreTile/StoreTiles 直接 os.WriteFile 最终路径；不是临时文件写完后原子重命名。中断或并发读可能看到不完整内容，这是代码层面风险，本次未模拟进程断电或证明已发生损坏。
4. 打包和依赖维护成本高。go.mod 含八个相邻本地仓库 replace；GEOS、PROJ、GeographicLib、XSLT 等原生静态依赖使跨平台构建复杂。本次遇到三处原生包构建参数问题，必须修改头文件目录、库目录或全角空格。它们并非单纯换一个 Go 编译版本就能解决。
5. 配置语义有陷阱。NW/UL 枚举存在，但 grid.go 中 FlippedYAxis 只对 UL 生效；试验使用 ul 规避。已有 GCJ02MC 明确有旧的多项式数值约定，不能简单宣称为错误公式并覆盖；正式补丁改成新增 GCJ02:3857，保留旧语义。

## 可取之处

分包、缓存策略、热重载、请求超时和服务管理已有一定设计；图像变形具有按误差自适应细分网格的实现，并非固定平移瓦片。根包、setting、cache、service 的相关测试通过，client 和 cache 的竞态检测通过。修补后本机 CLI 可以构建，独立 RustGIS CLI 对北京、上海、广州、武汉四个公开地标实际渲染检查成功。不能据此推断全国范围、卫星图、所有缩放层级或并发负载均已验收。

## 验证与边界

所有源码、编译和验证在独立目录中进行。正式 Geo Service 未停止、重启或写入测试数据。

- go-tileproxy：根包、setting、cache、service 测试通过；client/cache 的 -race 测试通过；CLI 构建通过。
- go-geo：投影相关测试、新增四地参考值/逆转换/极区测试及原有 gcj02 包通过。全量仍有四个网格测试失败，在恢复上游 proj.go/srs.go 的独立基线中同样复现。
- go-geoid、go-xslt：全量测试通过。
- go-geos：TestVersion 通过。TestVersionConsts 的 JTS 版本断言与 bundled 库报告值不符；没有更改该测试或预编译库，不宣称全量通过。
- go-tileproxy 全量检查遇到 TestGetGeotiff 外网超时，以及缺少 data/test_import.mbtiles；后者在上游原始基线也复现。这些不是本次根包定向测试失败。
- 第一轮新版 RustGIS 导出为空白，随后在代理已确认可用、独立新缓存的第二轮完成四地渲染和逐图目视检查；第一轮不计作验收成功。

## fork、提交与上游反馈

所有 PR 使用中文，状态已回读为 OPEN，未合并。fork 分支保留本次改动；没有将这些改动部署到正式服务。

- go-geo：`816c014352aaac2a4e9d43e3f2c7429ecce7d975`；[上游 PR](https://github.com/flywave/go-geo/pull/1)；[fork 分支](https://github.com/codepiano/go-geo/tree/codex/amap-xyz-support)。
- go-tileproxy：`c80cbe0ca7b1f26e71a61b6059c6e533b2f13b9d`；[上游 PR](https://github.com/flywave/go-tileproxy/pull/2)；[fork 分支](https://github.com/codepiano/go-tileproxy/tree/codex/amap-xyz-support)。
- go-geoid：`4a97bb9cfc5e2afcbf117bb1c127d0ce2b7bc24f`；[上游 PR](https://github.com/flywave/go-geoid/pull/1)；[fork 分支](https://github.com/codepiano/go-geoid/tree/codex/fix-cgo-paths)。
- go-geos：`5b5fdc03afe5846ec5640c488f47c009117fd2df`；[上游 PR](https://github.com/flywave/go-geos/pull/1)；[fork 分支](https://github.com/codepiano/go-geos/tree/codex/fix-cgo-paths)。
- go-xslt：`07156316aa57cb5d7f2e0e0ab985a048d80ed132`；[上游 PR](https://github.com/flywave/go-xslt/pull/1)；[fork 分支](https://github.com/codepiano/go-xslt/tree/codex/fix-cgo-paths)。

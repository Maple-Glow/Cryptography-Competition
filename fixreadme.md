配置与文档

configs/local.yaml
整个本地复现的“单一事实来源”。集中管理路径（数据、索引、preamble 输出）、分片数（emb/url shards）、端口、并发、模型/维度等参数。离线流水线、编排脚本、启动脚本都从这里读取，避免各处硬编码。

docs/ARCHITECTURE.md
项目“地图”：在线/离线两条链路、数据流（corpus → embedding → 聚类/PCA → 打包 → preamble → 在线检索）、关键耦合点（packing、protocol、存储抽象），以及与 tiptoe 原目录的对应关系。

docs/DEPLOY_LOCAL.md
本地运行指南：准备依赖→生成小语料→一键建立索引→生成 preamble→拉起服务→发起查询→查看日志/指标。面向新人按步骤走完。

docs/EXTEND.md
扩展说明：如何替换向量模型、改聚类/PCA参数、扩/缩分片、换存储后端（LocalFS/MinIO）、在容器或多机场景下平滑迁移。

docs/TROUBLESHOOTING.md
常见坑与诊断：端口占用、内存不够、preamble 校验失败、分片不一致、客户端/服务端维度不匹配、HE 库安装等问题的定位思路。

容器与启动

docker/Dockerfile
构建统一开发/运行镜像：Go（构建 search/*）、Python（embedding/cluster 等）、原生依赖（如 SEAL/HEXL 的系统包）。让“裸机只装 Docker”也能跑通。

docker/compose.yml
一键起停本地多进程拓扑（coordinator、若干 emb-server/url-server、可选 MinIO），并挂载 data/、logs/ 与 configs/。便于团队对齐运行环境。

cmd/local-run.sh
面向使用者的“一个脚本”入口：读取 configs/local.yaml，调用编排器拉起/关闭全部在线服务与客户端自测；提供基本的诊断输出与日志定位。

进程编排与健康检查

orchestration/local_launcher.py
本地“编排器”：按分片数并行启动 search/ 里的 emb/url 服务与 coordinator，做健康探测（/healthz 或端口探针）、失败重启、日志路径规范化，支持前台/后台运行。

存储抽象（去 AWS 化的关键）

storage/base.py
存储接口定义：get/put/exists/ls 等最小集合，屏蔽“文件系统 vs 对象存储”的差异，让流水线与评测脚本只面向抽象编程。

storage/localfs.py
本地文件系统实现：把路径当 key 使用。默认实现，适配离线管线与预处理产物的读写。

storage/minio.py（可选）
S3 兼容实现（走 MinIO 或任意 S3 端点），为后续扩展到对象存储留好接口，但不会强制依赖 AWS。

离线流水线（索引构建）

pipelines/local_index.py
一键离线管线驱动器：串起 embedding/ 向量化 → cluster/ KMeans 分配/后处理 → dim-reduce/ PCA → search/packing 打包（构建 emb/url 分片文件）。输出供 preamble 构建和在线服务使用。所有路径/参数取自 configs/local.yaml 与 storage.*。

预处理与辅助脚本

scripts/make_small_corpus.py
生成/下载最小可运行语料（如公共数据采样或内置小 TSV），标准化到统一 schema（doc_id、text、url 可选），体量可控，便于 CI 和 demo。

scripts/build_preamble.py
把“服务启动时的预处理”前置为独立步骤：从打包好的 embedding/url 分片构建各服务所需的 preamble/数据库/索引状态，并做基本校验（尺寸、checksum、元数据一致性）。

scripts/kill_all.sh
清理辅助：一键回收由编排器或手工启动的所有 tiptoe 相关进程，避免端口/僵尸进程占用。

scripts/healthcheck.py
统一健康检查：对 coordinator 与各分片做端口或 HTTP 探测，输出人读得懂的状态摘要，供 CI 或本地排障使用。

测试与评测

tests/fake_corpus_test.sh
极小端到端自测：用内置超小语料，构建索引 → 生成 preamble → 拉起最小分片拓扑 → 跑几条固定查询并断言返回数量/顺序/解密成功。用于快速回归。

tests/real_corpus_local.sh
真实公开语料的小规模复现：面向开发/评测，把“最小 demo”扩大到更现实的体量，仍不依赖 S3。便于观察性能/内存曲线。

基准与可视化（本地版）

perf-eval/local_bench.py
本地压测工具：按 QPS/并发/查询集合对 coordinator 施压，采集 p50/p95/p99、吞吐、错误率，以及可选 CPU/RSS（采样）。产出结构化 JSONL，供绘图使用。

quality-eval/local_quality.py
质量评测：加载小规模标注查询（如 MS MARCO dev 子集），计算 MRR@k / nDCG@k 等指标；与 tf-idf/ 的基线结果并排输出，验证“功能正确+效果合理”。

plots/local_plot.py
从 local_bench.py 与 local_quality.py 的结果文件生成延迟曲线、吞吐 vs 分片、质量对比等图表，方便复现实验报告的核心图。

CI 与工程化

.github/workflows/local.yml
CI 工作流：lint → 构建（Go/Py）→ 运行最小端到端用例 →（可选）轻量压测。缓存依赖与构建产物（如 preamble），产出日志与图表为 Artifacts。

Makefile
统一命令入口：build / demo / e2e / bench / clean 等，把容器、流水线、预处理、编排、测试串起来，降低上手成本。

杂项与占位

.env.example
环境变量样例：数据根目录、默认端口、并发/分片、是否启用调试等。便于在不同机器快速切换配置。

.gitignore
忽略大文件与生成物：data/、preamble/、logs/、虚拟环境、缓存、临时图表等，保持仓库整洁。

data/.gitkeep
数据目录占位；离线语料、向量、聚类结果、preamble 等默认都放这里（或其子目录），方便统一挂载与清理。

logs/.gitkeep
日志目录占位；编排器与各服务的标准输出/错误输出集中到这里，便于定位问题。

这些新增件与原仓库的关系（一句话总览）

离线侧：pipelines/local_index.py 驱动原有 embedding/、cluster/、dim-reduce/ 与 search/packing/，读写通过 storage/ 抽象，参数统一在 configs/local.yaml。

预处理：scripts/build_preamble.py 产出服务启动所需状态，避免依赖作者的 S3 预处理数据。

在线侧：orchestration/local_launcher.py 与 cmd/local-run.sh 拉起原有 search/ 的 emb/url/coordinator/client 服务，scripts/healthcheck.py 负责探测。

评测与展示：perf-eval/local_bench.py、quality-eval/local_quality.py、plots/local_plot.py 构成本地可复现实验闭环。

工程化：Docker + CI + Makefile + 文档，保证“新人按文档”即可跑通端到端。
# 本地服务发现目录（svcdir）

一个运行在 Flask + SQLite 上的小型服务发现目录：运维人员可以在网页上修改
「服务名 → 目标地址」的映射、创建命名备份、并从一个备份恢复。另有 `/resolve`
查询 API 供客户端使用。

## 启动

### Docker Compose（推荐）

```bash
docker compose up --build
```

- 网页：<http://localhost:8000/>
- API：`GET http://localhost:8000/resolve?name=orders-api`
- 健康检查：`GET /healthz`
- SQLite 数据与备份都位于命名卷 `svcdir-data` 的 `/data/directory.sqlite3`
  （备份是同一数据库里的版本快照，所以备份与恢复天然原子）。

### 本地运行

```bash
python3 -m pip install -r requirements.txt
SVCDIR_DB=./data/directory.sqlite3 python3 -m svcdir   # 开发用内置服务器
# 或： gunicorn --bind 0.0.0.0:8000 --workers 1 --threads 8 svcdir.__main__:app
```

环境变量：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `SVCDIR_DB` | `data/directory.sqlite3` | SQLite 文件路径 |
| `SVCDIR_RESTORE_LEASE_SECONDS` | `30` | 恢复租约时长（进程异常死亡后的兜底回收上限） |
| `PORT` | `8000` | 仅用于 `python -m svcdir` |

## 页面与 API

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/` | 网页：目录、修改表单、备份列表、恢复按钮 |
| GET | `/resolve?name=<服务名>` | 返回单个服务的地址；地址被隔离时 503 |
| GET | `/resolve` | 返回整个当前目录（排除指向隔离地址的服务） |
| GET | `/api/state` | 当前版本、完整目录、隔离清单、恢复状态、备份列表 |
| GET | `/api/backups` | 备份列表 |
| POST | `/api/services` | `{"service": ..., "address": ..., "note": ...}` 新增/修改 |
| DELETE | `/api/services/<名>` | 删除 |
| POST | `/api/quarantines` | `{"address": ..., "reason": ...}` 隔离目标地址 |
| DELETE | `/api/quarantines/<地址>` | 解除隔离 |
| GET | `/api/quarantines` | 隔离清单及受影响服务 |
| POST | `/api/backups` | `{"name": ...}` 创建命名备份 |
| GET | `/api/backups/<名>/validate` | 校验备份（不修改目录） |
| POST | `/api/backups/<名>/restore` | 从备份恢复 |

浏览器表单走 `POST /services`、`POST /quarantines`、
`POST /quarantines/<地址>/release`、`/backups`、`/backups/<名>/restore`，
成功后 303 跳回页面并显示结果或错误。

查询响应示例：

```json
{"ok": true, "version": 6, "restore_in_progress": false,
 "name": "orders-api", "address": "10.0.0.12:8080", "available": true}
```

指向已隔离地址的服务：

```json
{"ok": false, "error": {"code": "address_quarantined",
 "message": "服务 orders-api 当前不可用：目标地址 10.0.0.12:8080 已被隔离",
 "service": "orders-api", "address": "10.0.0.12:8080", "available": false}}
```

HTTP 状态码为 `503`。全目录查询 `GET /resolve` 不会列出这些服务，
并附带 `quarantined_count`。

## 一致性与恢复语义

实现要点（对应需求逐条）：

- **备份是同一提交点的完整目录。** 目录从不原地修改：每次修改都写入一个
  *完整的新版本*（先复制当前版本的全部行，再改一行），最后一条
  `UPDATE meta SET value=<新版本>` 才是发布。备份行只固定「版本号」，因此
  创建备份期间持续写入不会让备份变成半个目录。
- **恢复前校验。** 恢复分为三步：①在一个写事务里登记「恢复中」租约；②只读
  事务里校验备份（条目数、每行服务名/地址合法性、重复服务名、SHA-256 摘要与
  备份记录比对）；③一个原子写事务创建完整的新版本并切换 `current_version`。
  校验失败 → 直接返回 422，目录一个字节都不动。
- **恢复期间查询仍返回当前完整目录。** `current_version` 只在最后一步切换，
  查询永远读它；整个查询在读事务快照里完成，因此不会读到混排的数据。
- **新修改明确拒绝，已受理的修改先结算。** 登记恢复意图需要写锁，所有已受理
  的修改必然先提交完成；之后到达的修改看到有效租约，返回 `409
  restore_in_progress`（网页也会明确提示）。
- **恢复后新查询只见备份地址；先前受理的查询不混用两版。** 切换后新查询用新
  版本；一个在切换前开始的查询持有自己的读快照（`store.read_view()`），
  即使内部重复读取也仍是同一版本。
- **切换中断后重启只呈现原版或完整备份版。** 新版本与版本号切换在同一事务内
  提交；崩在提交前 → 仍是原版，崩在提交后 → 是完整备份版，旧版本的行永不删除。
  恢复者用租约 + 操作系统文件锁（`<db>.restore.lock`）标记存活：进程死亡时
  内核释放锁，重启后立即回收被遗弃的「恢复中」标记，不必等租约到期。
- **并发恢复。** 文件锁 + 数据库内的 `restore_state` 双重互斥，第二个恢复请求
  得到 409。

## 地址隔离语义

隔离针对**目标地址**而不是服务名：运维员判定某个地址故障后，所有当前指向它
的服务立即停止解析，直到显式解除。实现要点：

- **隔离清单独立于目录版本。** 隔离记录在 `quarantines` 表，不在按版本复制的
  `entries` 里；命名备份只固定目录行。因此从任何备份恢复都不会删除隔离记录，
  也不会重新暴露故障地址。隔离记录持久化在 SQLite 文件中，重启后仍生效。
- **隔离状态变化推进目录版本。** 隔离 / 解除隔离都在同一个写事务里写入
  `quarantines` 并创建一个内容相同的新版本（`kind=quarantine|unquarantine`），
  随后发布 `current_version`。查询在单个读事务里同时读取版本行与隔离集合，
  所以新查询立即看到变化，而已开始的查询仍在其旧快照上完整结束（不会混用新
  隔离集合与旧目录，或反之）。
- **按当前地址判定。** 服务是否被阻断由「当前版本里该服务的地址」与「隔离
  集合」在读取时求交：服务改址到正常地址后即不再受旧地址隔离影响；改回隔离
  地址或新增指向隔离地址的服务则立即不可用。
- **解除隔离只恢复仍指向该地址的服务。** 响应里的 `restored` 是解除时实际
  指向该地址的服务列表；期间已改址的服务不受影响。
- **与恢复的互斥。** 隔离 / 解除隔离与普通修改一样，在恢复进行中被拒绝
  （409）并先于恢复意图结算，所以不会出现「恢复把目录换掉、隔离却落在另一
  版目录上」的混排；隔离失败（校验或写入失败）时事务整体回滚，原状态不变。
- **重启。** 隔离表是普通表，随数据库文件持久化；schema 为 v2，旧 v1 数据库
  在启动时自动补建空的 `quarantines` 表（不改变已有目录）。

## 测试

```bash
python3 -m pip install -r requirements-dev.txt
python3 -m pytest -q
```

关键交错场景覆盖在 `tests/test_svcdir.py`：

- `test_quarantine_blocks_single_and_whole_directory`
- `test_quarantine_publishes_new_version`
- `test_unquarantine_restores_only_currently_matching_services`
- `test_repointing_a_service_escapes_the_old_address_quarantine`
- `test_new_service_pointing_at_quarantined_address_is_blocked`
- `test_restore_does_not_resurrect_a_quarantine`
- `test_failed_validation_keeps_quarantine_state`
- `test_quarantine_survives_restart`
- `test_quarantine_rejected_while_restore_in_progress`
- `test_query_started_before_quarantine_uses_one_state`
- `test_interleaved_quarantine_and_restore_never_mixes`
- `test_write_failure_in_quarantine_leaves_state_intact`
- `test_http_quarantine_flow` / `test_html_quarantine_forms`
- `test_quarantine_input_validation` / `test_schema_v1_database_is_migrated`

- `test_writes_rejected_while_restore_in_progress_and_settled_before`
- `test_query_accepted_before_switch_keeps_one_version`
- `test_concurrent_writers_during_restore_never_mix`
- `test_cross_process_readers_and_writer_during_restore`（独立进程 + WAL 快照）
- `test_corrupt_backup_is_rejected_and_directory_unchanged`
- `test_backup_with_missing_rows_is_rejected`
- `test_interrupted_switch_presents_original_or_complete_backup`
- `test_killed_restorer_leaves_original_and_restart_can_retry`（SIGKILL 恢复进程）
- `test_expired_restore_lease_is_reclaimed_after_restart`
- `test_dead_restorer_is_reclaimed_immediately_on_restart`
- `test_restore_can_run_after_prior_abandoned_marker`
- `test_live_restore_lease_survives_sibling_process_initialization`
- `test_backup_row_pins_one_version_during_continuous_writes`
- `test_http_rejects_writes_and_serves_reads_during_restore`
- `test_api_flow` / `test_html_page_and_forms` / `test_html_escapes_hostile_names`
- `test_api_validation_errors_are_json` / `test_edge_case_inputs_are_rejected_cleanly`

## 布局

```
svcdir/errors.py     错误类型 → HTTP 状态码
svcdir/store.py      目录版本、命名备份、校验、恢复、地址隔离（核心逻辑）
svcdir/web.py        Flask 路由：网页 + API
svcdir/templates/    页面模板
tests/test_svcdir.py 自测（含并发交错）
Dockerfile / docker-compose.yml
```

## 已知限制与未完成项

- **未在 Docker 中实际构建/运行。** 本环境没有 Docker（`docker` 命令不存在），
  因此 Dockerfile 与 compose 文件只做了静态检查。应用本身已用
  `python -m svcdir` 与 gunicorn 在真实 HTTP 上跑通。
- 单容器部署，恢复互斥依赖同一文件系统上的 `flock`；若把数据目录放到不支持
  `flock` 的网络文件系统，会退化为「租约到期才算失效」。
- 备份保存在同一 SQLite 数据库中，可防止误改目录，但不能防磁盘/文件损坏；
  需要异地容灾应另行导出备份文件。
- 未做用户认证与 HTTPS——按需求这是本地运维工具，请只在内网使用。
- 恢复的「已受理查询」保证依赖单个查询请求只在一个读事务里读完（当前实现如此）；
  没有为客户端提供跨多个独立 HTTP 请求的版本固定参数。

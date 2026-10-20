# 部署记录：4.3.2-optimize（2026-10-20）

主机：`f`（root@70.40.186.45），服务：`sol.service`

## 版本

| 组件 | 仓库 | 分支 / 提交 |
|---|---|---|
| agave-validator | `~/agave` | `4.3.2-optimize` / `471d23b1a7` |
| libyellowstone_grpc_geyser.so | `~/yellowstone-grpc` | `4.3.2-optimize` / `4c7d0b8` |

agave 包含：sendTransaction 默认跳过 preflight 与默认重试 0、deshred 服务门控、finalized 签名订阅跳过、每 2000 slot 清空签名订阅。
yellowstone 包含：移除 vote payload 丢弃（`drop_vote_payloads`）。

## 构建

与 `manifest` 约定一致：

```
# geyser
cd ~/yellowstone-grpc
CARGO_TARGET_DIR=target-abi-4.3 RUSTFLAGS='-Ctarget-cpu=x86-64-v2' \
  cargo build --release --locked --offline --package yellowstone-grpc-geyser -j 4

# agave
cd ~/agave
cargo build --release --locked --offline --bin agave-validator -j 6
```

产物 sha256：

- `agave-validator` `5b5744c74fa5adf5e49f3d450fcf835e51adece00996ac55f49527c1d8c96094`
- `libyellowstone_grpc_geyser.so` `9b70c93547b50dfd4ae16bf0eec83cff5ead06993c54a54f0e2952896687be33`

构建耗时：geyser 约 3 分 10 秒，agave 约 6 分 51 秒。

## 远端布局（以实际为准）

- `sol.service` → `ExecStart=/root/validator.sh`
- `/root/validator.sh` 执行二进制与 geyser 配置。**这是线上真实脚本，参数与发布目录里的 `validator.sh` 不同，不能整体覆盖。**
- 发布目录：`/root/agave-releases/<版本名>/`（旧版：`4.3.0-optimize-20260920`，`grpc-a6ad340-agave4.3.0`）
- `/root/deploy_all.sh` 是 trader 的部署脚本，**与本服务无关**。

## 部署步骤

1. 在本地构建两个产物（见上）。
2. 远端创建 `/root/agave-releases/4.3.2-optimize-20261020-2306/`，上传二进制与 `.so`，核对 sha256。
3. 远端 `agave-validator --version` 确认为 `4.3.2 (src:471d23b1)`。
4. 生成该目录下的 `yellowstone_config.json`（`libpath` 指向新 `.so`）。
5. 备份并修改线上 `/root/validator.sh`，**只替换两处**：
   - `/root/agave-releases/4.3.0-optimize-20260920/agave-validator` → 新目录
   - `--geyser-plugin-config /root/yellowstone_config.json` → 新目录下的配置
   用 `diff` 确认仅这两处变化。
6. `systemctl stop sol.service` → `systemctl start sol.service`。
7. 验证：进程路径、geyser 日志（`loading plugin: ...-g4c7d0b8`）、gRPC 端口 60003 监听。

## 回滚

```
systemctl stop sol.service
cp /root/validator.sh.bak-pre-4.3.2-optimize-20261020-2306 /root/validator.sh
cp /root/yellowstone_config.json.bak-pre-4.3.2-optimize-20261020-2306 /root/yellowstone_config.json
systemctl start sol.service
```

旧二进制仍在 `/root/agave-releases/4.3.0-optimize-20260920/`，可直接指回。

## 遇到的问题

1. **脚本不一致**：发布目录的 `validator.sh` 与线上 `/root/validator.sh` 参数不同（`--accounts`、`--snapshots`、重试参数等）。直接复制会改变线上行为，因此只做最小替换。
2. **废弃配置键**：线上 `yellowstone_config.json` 含 `drop_vote_payloads: true`，该选项已从 grpc 代码删除，新插件会拒绝。新配置已移除该键。
3. **废弃 CLI 参数（仅告警）**：`--accounts-db-access-storages-method`、`--accounts-db-cache-limit-mb`、`--disable-banking-trace`、`--enable-accounts-disk-index`、`--limit-ledger-size` 已废弃，应在后续版本替换为 `--accounts-db-write-cache-limit`、`--accounts-index-limit`、`--limit-blockstore-size` 等。
4. **RPC 初始化耗时**：服务启动后 RPC 端口（60001）需要约 20–30 分钟才会绑定与响应，这是正常的 bootstrap 过程，不是故障。启动期间 `getHealth`/`getSlot` 返回连接拒绝或空结果属于预期。
5. **启动期告警**：`Failed to get RPC node version`、`Rayon global thread pool already initialized`、`voting disabled`（`--no-voting` 预期）为正常启动输出。

## 待确认

- RPC 恢复后，需要确认 `getHealth` 为 `ok`、slot 追赶到网络、gRPC 下游（60003）客户端能正常接收数据。
- 上一次部署验证的中断，意味着 RPC 恢复时间还未观测到，需要人工确认。

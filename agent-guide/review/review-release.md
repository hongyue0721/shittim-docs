# review-release — 发布门审查

> 审查一次发布是否满足全部硬门，是否可以安全提交/推送/同步镜像。**在 push 之前执行**。

## 目标

确认：统一门全绿、无 generated 漂移、FILE_MANIFEST 正确、身份合规、双仓闭环可验证。

## 逐项清单

### A. 门禁证据

- [ ] `./scripts/check-schema.sh` 实际运行且全绿（Node 硬门 → generate×2 → check → fmt → clippy → workspace test → generated 漂移 → FILE_MANIFEST）；**审查者不替开发方跑，但必须核对其输出**；
- [ ] generated 无漂移：`git diff --exit-code -- rust/crates/kernel-contracts/src/generated` 干净；
- [ ] `cargo fmt --check` / `cargo clippy -D warnings` 证据；
- [ ] 聚焦测试 + 全 workspace 测试证据（测试名/数量来自真实输出）。

### B. 提交内容

- [ ] 提交信息中文、按功能域；一个功能域一个提交；
- [ ] 无无关文件混入（`git show --stat <sha>`）；
- [ ] 无敏感文件（凭据、密钥、本地路径）；无构建产物/依赖被提交。

### C. 身份与远端

- [ ] local `user.name`/`user.email` 与 HEAD author/committer 均为 `小岳` / `2933634892@qq.com`；
- [ ] 远端为主仓 `https://github.com/hongyue0721/shittim.git`（`git remote -v`）；
- [ ] 推送后 `git ls-remote origin master` == `git rev-parse HEAD`。

### D. 文档镜像

- [ ] `pnpm run sync:docs-repository` 成功；`pnpm run check:docs-repository` 通过；
- [ ] 镜像闭集正确：只含 tracked `*.md` + `LICENSE` + 固定 `.gitignore`；
- [ ] 镜像提交消息含来源主仓完整 SHA；两仓 `FILE_MANIFEST.md` 一致；
- [ ] 无 force push 痕迹；无平行 commit。

### E. 完成度诚实性

- [ ] 本次发布宣称的完成度与 MATRIX/PROGRESS/代码一致；
- [ ] 无「文档完成但代码没有」的切片；
- [ ] 未实现的五方法/child materializer/真实远程验签等没有被写成已支持；
- [ ] `V2InitialBuildActive` 谓词未全闭合前，无「可启动 server」宣称。

## 输出

按 `review-overview.md` §5 报告格式；结论 **GO**（可 push）或 **NO-GO**（列出全部阻断项）。NO-GO 时 push 不得执行，由开发方修根因后重审。

# Workshop 9 - Container Security - Docker Scout

用 Docker Scout 扫描 [docker/scout-demo-service](https://github.com/docker/scout-demo-service)(Node.js/Express 应用),完成漏洞修复与合规整改。

## 镜像版本演进(rentalover/scout-demo-grp1)

| 版本 | 变更 | 扫描结果 |
|---|---|---|
| v1 | 原始代码(express 4.17.1) | **3 CVE**:CVE-2022-24999 (HIGH)、CVE-2024-29041 (MEDIUM)、CVE-2024-43796 (LOW) |
| v2 | express → 4.21.2(`npm i`) | **0 CVE** ✓ |
| v3 | Dockerfile 添加 `USER appuser`;启用 containerd-snapshotter;`--provenance --sbom` 构建 | 合规 1/7 → 3/7(非 root ✓、attestation ✓) |
| v4 | 重写 Dockerfile(alpine:latest + `apk upgrade` + `npm audit fix --force` → express 4.22.2) | **健康评分 A (94%),6/7 策略通过** |

## 最终 quickview(v4)

```text
Policy status  FAILED  (6/7 policies met)
Health score  A  (94%)

 ✓  Default non-root user
 !  Copyleft licensed packages found        17 packages
 ✓  No fixable critical or high vulnerabilities   0C 0H 0M 0L
 ✓  No high-profile vulnerabilities
 ✓  No outdated base images
 ✓  No unapproved base images
 ✓  Supply chain attestations
```

## 已知残留风险(2026-08-20)

1. **CVE-2026-58055(nghttp2-libs 1.69.0-r0,severity UNSPECIFIED)**:由 alpine 的 `nodejs` 包引入,修复版 1.70.0-r0 尚未进入任何 stable Alpine 分支(v3.23/v3.24 均只有 1.69.0-r0),属上游未修复。Dockerfile 已含 `RUN apk update && apk upgrade`,Alpine 发布修复包后重建即自动修复。应用本身为 HTTP/1.1(Express),不启用 HTTP/2,实际暴露面有限。
2. **Copyleft 许可证包(17 个)**:来自 npm 依赖树的许可证策略提示,需法务评估后决定是否替换依赖,无法通过技术手段"修复"。
3. **devDependencies 审计残留(6 项)**:`npm audit fix` 无法在不做破坏性升级的前提下清除,且 devDependencies 不进入镜像(`NODE_ENV=production` 构建时跳过)。

## 执行命令记录

```bash
# 构建各版本
docker build --push -t rentalover/scout-demo-grp1:v1 .
docker build --push -t rentalover/scout-demo-grp1:v2 .
docker build --provenance=true --sbom=true --push -t rentalover/scout-demo-grp1:v3 .
docker build --provenance=true --sbom=true --push -t rentalover/scout-demo-grp1:v4 .

# 扫描
docker scout cves --only-package express rentalover/scout-demo-grp1:v1
docker scout cves --only-package express rentalover/scout-demo-grp1:v2
docker scout quickview rentalover/scout-demo-grp1:v4
docker scout recommendations rentalover/scout-demo-grp1:v1
```

## 环境

- DigitalOcean droplet `scout`:Ubuntu 24.04,1 vCPU / 1GB(sgp1)
- Docker Engine 29.7.2(buildx + containerd-snapshotter 已启用)
- Docker Scout CLI v1.24.0(`~/.docker/cli-plugins/docker-scout`)

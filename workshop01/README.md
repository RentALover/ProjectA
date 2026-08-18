# Workshop 4 - Terraform

用 Terraform HCL 在 DigitalOcean 上自动化部署的容器应用栈:

- Docker 网络 `bgg-net`
- MySQL 容器 `bgg-database`(镜像 `chukmunnlee/bgg-database:v3.1`,数据卷挂载 `/var/lib/mysql`,暴露 3306)
- 3 个 `bgg-backend` Node.js 容器(镜像 `chukmunnlee/bgg-backend:v3`,环境变量 `BGG_DB_HOST` 指向数据库容器,内部端口 3000)
- 独立 droplet 上的 Nginx 反向代理,`nginx.conf` 由 `sample.nginx.conf.tftpl` 根据后端容器的外部端口自动生成

## 文件

| 文件 | 作用 |
|---|---|
| `provider.tf` | 声明 docker / digitalocean / local 三个 provider |
| `variables.tf` | 变量定义(docker 主机、DO 凭据、版本号、实例数等) |
| `resources.tf` | 镜像、网络、数据卷、容器、droplet、Nginx 配置生成与输出 |
| `sample.nginx.conf.tftpl` | Nginx 反向代理配置模板 |

## 执行命令(在装有 docker-machine 的工作站上)

```bash
terraform init

terraform plan \
  -var "do_token=${DO_PAT}" \
  -var "ssh_private_key=/root/.ssh/id_rsa" \
  -var "docker_host=<docker host ip>" \
  -var "docker_cert_path=/root/.docker/machine/machines/docker-nginx"

terraform apply -auto-approve \
  -var "do_token=${DO_PAT}" \
  -var "ssh_private_key=/root/.ssh/id_rsa" \
  -var "docker_host=<docker host ip>" \
  -var "docker_cert_path=/root/.docker/machine/machines/docker-nginx"
```

## 产出

- 输出 `nginx_ip`(反向代理公网 IP)与 `backend_ports`(3 个后端外部端口)
- 空文件 `root@<nginx_ip>`
- 浏览器访问 `http://<nginx_ip>` 验证应用

## 本次部署结果(2026-08-18)

```text
Apply complete! Resources: 11 added, 0 changed, 0 destroyed.

Outputs:
backend_ports = [ 32770, 32768, 32769 ]
nginx_ip      = "143.198.86.94"
```

- 反向代理:`http://143.198.86.94`(HTTP 200)
- 后端端点:`139.59.255.63:32770`、`139.59.255.63:32768`、`139.59.255.63:32769`
- 生成的空文件:`root@143.198.86.94`(terraform apply 后出现在工作目录)


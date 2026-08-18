# Workshop 5 - Ansible

用 Ansible 自动化在 DigitalOcean 服务器上安装 code-server(浏览器版 VS Code)+ Nginx 反向代理。

## 架构

- **Terraform**:创建 codeserver droplet,生成 Ansible 的 `inventory.yaml` 主机清单和空文件 `root@<ip>`
- **Ansible playbook**:下载安装 code-server 4.9.1、安装 nginx、用 Jinja2 模板渲染 systemd 服务和 nginx 配置、`daemon_reload` 并重启服务

## 文件

| 文件 | 作用 |
|---|---|
| `provider.tf` / `variables.tf` / `resources.tf` | Terraform 创建 droplet + 生成 inventory |
| `inventory.yaml.tftpl` | Ansible 主机清单模板(含 SSH 私钥路径、域名、密码变量) |
| `code-server.service.j2` | systemd 服务模板(`Environment=PASSWORD={{codeserver_password}}`) |
| `code-server.conf.j2` | Nginx 配置模板(`server_name {{codeserver_domain}} {{ansible_host}}`) |
| `playbook.yaml` | Ansible 剧本 |

## 执行命令(工作站上)

```bash
terraform init

terraform plan -var "do_token=${DO_PAT}" \
  -var "ssh_private_key=/root/.ssh/id_rsa" \
  -var "codeserver_password=password123456"

terraform apply -auto-approve -var "do_token=${DO_PAT}" \
  -var "ssh_private_key=/root/.ssh/id_rsa" \
  -var "codeserver_password=password123456"

ansible-playbook playbook.yaml -i inventory.yaml
```

## 验证

浏览器访问 `http://code-<ipv4_address>.nip.io`(nip.io 自动解析到服务器 IP)或直接 `http://<ipv4_address>`,输入密码登录 code-server。

## 本次部署结果(2026-08-18)

```text
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
codeserver_ip = "129.212.236.230"

PLAY RECAP (ansible-playbook playbook.yaml -i inventory.yaml)
codeserver: ok=14  changed=4  unreachable=0  failed=0  skipped=1
```

- 访问地址:`http://code-129.212.236.230.nip.io`(登录页 HTTP 200,密码 `password123456`)
- 生成的空文件:`root@129.212.236.230`(terraform apply 后出现在工作目录)
- 验证点:
  - `/lib/systemd/system/code-server.service`:`Environment=PASSWORD=password123456`
  - `/etc/nginx/sites-available/code-server.conf`:`server_name code-129.212.236.230.nip.io 129.212.236.230;`
  - systemd:`code-server` / `nginx` 均 active + enabled,`daemon_reload: true`


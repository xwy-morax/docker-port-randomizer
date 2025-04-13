# Docker Port Randomizer

![版本](https://img.shields.io/badge/版本-2.0.0-blue)
![平台](https://img.shields.io/badge/平台-Linux-green)
![语言](https://img.shields.io/badge/语言-Bash-yellow)

一个高性能的Docker端口随机分配工具，用于解决Docker容器部署时的端口冲突和选择困难问题。当你需要部署多个服务但不想手动选择端口时，这个工具可以自动为你分配随机可用端口。

> 本项全部代码及本介绍目由 TIG AI 完全开发，官方网站：[https://phapi.furina.junmatec.cn/](https://phapi.furina.junmatec.cn/)

## 特性

- 🚀 **高性能**：采用"先随机后验证"策略，秒级完成端口分配
- 🔄 **自动检测**：自动排除已占用端口，确保分配的端口可用
- 🛡️ **安全可靠**：避免常用端口（如80、443）的冲突
- 🌐 **广泛兼容**：支持所有Linux发行版，特别优化用于Ubuntu
- 🔧 **灵活配置**：支持自定义端口范围、排除特定端口等
- 🇨🇳 **中文界面**：提供友好的中文交互体验

## 安装

### 方法1：直接下载

```bash
# 下载脚本
curl -o random-port.sh https://raw.githubusercontent.com/xwy-morax/docker-port-randomizer/main/random-port.sh

# 添加执行权限
chmod +x random-port.sh
```

### 方法2：克隆仓库

```bash
# 克隆仓库
git clone https://github.com/xwy-morax/docker-port-randomizer.git

# 进入目录
cd docker-port-randomizer

# 添加执行权限
chmod +x random-port.sh
```

### 全局安装（推荐）

将脚本安装到系统路径，使其在任何目录下都可用：

```bash
# 移动到系统PATH目录
sudo mv random-port.sh /usr/local/bin/random-port

# 确保它有执行权限
sudo chmod +x /usr/local/bin/random-port
```

## 使用方法

### 基本用法

```bash
# 基本用法
random-port "docker run -d -p 80:80 --name my-container my-image"

# 使用dry-run模式（不执行命令，只显示结果）
random-port --dry-run "docker run -d -p 80:80 --name my-container my-image"
```

### 高级选项

```bash
# 指定端口范围
random-port --range 8000-9000 "docker run -d -p 80:80 --name my-container my-image"

# 排除特定端口
random-port --exclude 80,443,3306,5432 "docker run -d -p 80:80 --name my-container my-image"

# 设置最大尝试次数
random-port --max-tries 100 "docker run -d -p 80:80 --name my-container my-image"
```

### 帮助信息

```bash
random-port --help
```

## 选项说明

| 选项 | 描述 |
|------|------|
| `-h, --help` | 显示帮助信息 |
| `-v, --version` | 显示版本信息 |
| `-r, --range` | 指定端口范围（默认：1024-65535） |
| `-e, --exclude` | 指定要排除的端口（默认：80,443） |
| `-d, --dry-run` | 仅显示修改后的命令，不执行 |
| `-m, --max-tries` | 最大尝试次数（默认：50） |

## 工作原理

Docker Port Randomizer 使用高效的"先随机后验证"策略：

1. 随机选择一个指定范围内的端口
2. 检查该端口是否被占用或在排除列表中
3. 如果端口可用，则使用该端口；否则重新选择
4. 将选定的端口替换到原始Docker命令中
5. 询问用户是否采纳并执行修改后的命令

这种方法比预先生成所有可用端口列表要快得多，特别是在处理大范围端口时。

## 支持的Docker命令格式

工具支持多种Docker端口映射格式：

- `-p 80:80`
- `-p=80:80`
- `--publish 80:80`
- `--publish=80:80`

## 常见问题

### Q: 工具支持Windows/macOS吗？
A: 目前仅支持Linux系统。Windows用户可以在WSL中使用。

### Q: 如何在CI/CD管道中使用？
A: 使用`--dry-run`选项获取修改后的命令，然后在CI/CD脚本中执行。

## 贡献

欢迎贡献代码、报告问题或提出改进建议！请遵循以下步骤：

1. Fork 这个仓库
2. 创建你的特性分支 (`git checkout -b feature/amazing-feature`)
3. 提交你的更改 (`git commit -m 'Add some amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 开启一个 Pull Request

## 致谢

- 感谢所有为这个项目提供反馈和建议的用户
- 特别感谢那些在部署多个Docker容器时遇到端口选择困难症的开发者们，这个工具就是为你们创建的！

## 关于开发者

本项目由 TIG 完全开发。如需了解更多信息或查看其他项目，请访问我们的官方网站：[https://phapi.furina.junmatec.cn/](https://phapi.furina.junmatec.cn/)


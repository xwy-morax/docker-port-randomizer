Role: Human
你说有没有一款是 适配Linux的端口随机生成工具，因为很多时候我要部署一个新的项目，但是项目原本给的那个运行示例代码，它总是用的80和433端口，我们常常需要把它改到其他端口 但是每一次部署项目 都需要确认一遍系统有哪些端口可用，然后选的端口实在是太选择困难症了，所以有没有这样一个东西，它作为一个直接可以运行的命令，可以通过一个文件复制到任何一个 Linux系统中，优先系统ubuntu
然后使用这个命令，加上一个运行容器的代码，比如
docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool
就可以自动检测服务器中可用的其他端口（默认排除80，443，以及已经被其他项目占用的端口，然后随机分配给这个项目一个端口，并以中文输出 询问是否采纳项目映射到这个宿主机端口

-------

Role: AI
```thinking
这个问题是关于在Linux系统(特别是Ubuntu)上创建一个端口随机生成工具的请求。让我分析一下需求和可能的解决方案。

首先，用户的需求是:
1. 一个可以在Linux(优先Ubuntu)上运行的工具
2. 能够检测系统中可用的端口
3. 自动排除80、443以及已被占用的端口
4. 随机选择一个可用端口
5. 能够与docker命令结合使用
6. 以中文输出并询问是否采纳该端口映射

这是一个很实用的需求。在部署多个服务时，端口分配确实是一个常见的小麻烦，特别是当你有选择困难症的时候。

我可以设计一个简单的shell脚本来实现这个功能。这个脚本需要:
1. 检查系统中已占用的端口
2. 排除80、443和其他已占用端口
3. 从剩余可用端口中随机选择一个
4. 接受一个docker命令作为参数
5. 修改该命令中的端口映射
6. 以中文输出并询问是否采纳

对于检查已占用端口，我们可以使用`netstat`、`ss`或`lsof`命令。`ss`命令在现代Linux系统中更为推荐，因为它比`netstat`更快。

对于随机选择端口，我们可以使用`shuf`命令或者`$RANDOM`变量。

让我设计这个脚本:

1. 首先，我们需要获取已占用的端口列表
2. 然后，生成一个可用端口列表(排除已占用的和特定的端口)
3. 从可用端口列表中随机选择一个
4. 替换docker命令中的端口映射
5. 输出修改后的命令并询问是否执行

我们可以将这个脚本命名为`random-port.sh`，并使其可执行。

让我开始编写这个脚本:

```bash
#!/bin/bash

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "请提供一个docker命令作为参数"
    echo "例如: ./random-port.sh \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    exit 1
fi

# 获取已占用的端口列表
OCCUPIED_PORTS=$(ss -tuln | grep LISTEN | awk '{print $5}' | cut -d: -f2 | sort -u)

# 定义要排除的特定端口
EXCLUDED_PORTS="80 443"

# 生成可用端口列表(1024-65535)
AVAILABLE_PORTS=""
for port in $(seq 1024 65535); do
    # 检查端口是否在已占用列表或排除列表中
    if ! echo "$OCCUPIED_PORTS $EXCLUDED_PORTS" | grep -q -w "$port"; then
        AVAILABLE_PORTS="$AVAILABLE_PORTS $port"
    fi
done

# 从可用端口列表中随机选择一个
RANDOM_PORT=$(echo $AVAILABLE_PORTS | tr ' ' '\n' | shuf -n 1)

# 获取原始docker命令
DOCKER_CMD="$1"

# 提取容器内部端口
CONTAINER_PORT=$(echo $DOCKER_CMD | grep -o -P '(?<=-p )\d+(?=:\d+)' || echo "80")

# 替换docker命令中的端口映射
NEW_DOCKER_CMD=$(echo $DOCKER_CMD | sed -E "s/-p [0-9]+:/-p $RANDOM_PORT:/")

# 输出修改后的命令并询问是否执行
echo "已为您随机选择端口: $RANDOM_PORT"
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"
echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
else
    echo "已取消执行。"
fi
```

等等，我注意到上面的脚本可能有一些问题:

1. 正则表达式部分可能不够健壮，特别是在提取容器内部端口时
2. 如果docker命令中有多个端口映射，这个脚本只会替换第一个
3. 我们应该考虑更多的边缘情况

让我改进一下这个脚本:

```bash
#!/bin/bash

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "请提供一个docker命令作为参数"
    echo "例如: ./random-port.sh \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    exit 1
fi

# 获取已占用的端口列表
OCCUPIED_PORTS=$(ss -tuln | grep LISTEN | awk '{print $5}' | cut -d: -f2 | sort -u)

# 定义要排除的特定端口
EXCLUDED_PORTS="80 443"

# 生成可用端口列表(1024-65535)
AVAILABLE_PORTS=()
for port in $(seq 1024 65535); do
    # 检查端口是否在已占用列表或排除列表中
    if ! echo "$OCCUPIED_PORTS $EXCLUDED_PORTS" | grep -q -w "$port"; then
        AVAILABLE_PORTS+=($port)
    fi
done

# 从可用端口列表中随机选择一个
RANDOM_PORT=${AVAILABLE_PORTS[$RANDOM % ${#AVAILABLE_PORTS[@]}]}

# 获取原始docker命令
DOCKER_CMD="$1"

# 提取端口映射部分
PORT_MAPPING=$(echo $DOCKER_CMD | grep -o -P '(?<=-p )[0-9]+:[0-9]+')

if [ -z "$PORT_MAPPING" ]; then
    echo "未找到端口映射，请确保docker命令包含类似 -p 80:80 的端口映射。"
    exit 1
fi

# 提取容器内部端口
CONTAINER_PORT=$(echo $PORT_MAPPING | cut -d: -f2)

# 替换docker命令中的端口映射
NEW_DOCKER_CMD=$(echo $DOCKER_CMD | sed -E "s/-p [0-9]+:$CONTAINER_PORT/-p $RANDOM_PORT:$CONTAINER_PORT/")

# 输出修改后的命令并询问是否执行
echo "已为您随机选择端口: $RANDOM_PORT"
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"
echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
else
    echo "已取消执行。"
fi
```

嗯，这个脚本还是有一些问题。特别是在处理多个端口映射时可能会出错。让我再次改进:

```bash
#!/bin/bash

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "请提供一个docker命令作为参数"
    echo "例如: ./random-port.sh \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    exit 1
fi

# 获取已占用的端口列表
OCCUPIED_PORTS=$(ss -tuln | grep LISTEN | awk '{print $5}' | cut -d: -f2 | sort -u)

# 定义要排除的特定端口
EXCLUDED_PORTS="80 443"

# 生成可用端口列表(1024-65535)
AVAILABLE_PORTS=()
for port in $(seq 1024 65535); do
    # 检查端口是否在已占用列表或排除列表中
    if ! echo "$OCCUPIED_PORTS $EXCLUDED_PORTS" | grep -q -w "$port"; then
        AVAILABLE_PORTS+=($port)
    fi
done

# 获取原始docker命令
DOCKER_CMD="$1"

# 提取所有端口映射
PORT_MAPPINGS=$(echo $DOCKER_CMD | grep -o -P '(?<=-p )[0-9]+:[0-9]+')

if [ -z "$PORT_MAPPINGS" ]; then
    echo "未找到端口映射，请确保docker命令包含类似 -p 80:80 的端口映射。"
    exit 1
fi

NEW_DOCKER_CMD="$DOCKER_CMD"

# 处理每个端口映射
echo "端口映射替换计划:"
while read -r mapping; do
    HOST_PORT=$(echo $mapping | cut -d: -f1)
    CONTAINER_PORT=$(echo $mapping | cut -d: -f2)
    
    # 从可用端口列表中随机选择一个
    RANDOM_INDEX=$((RANDOM % ${#AVAILABLE_PORTS[@]}))
    RANDOM_PORT=${AVAILABLE_PORTS[$RANDOM_INDEX]}
    
    # 从可用端口列表中移除已选择的端口
    unset AVAILABLE_PORTS[$RANDOM_INDEX]
    AVAILABLE_PORTS=(${AVAILABLE_PORTS[@]})
    
    echo "  原端口映射: $HOST_PORT:$CONTAINER_PORT -> 新端口映射: $RANDOM_PORT:$CONTAINER_PORT"
    
    # 替换docker命令中的端口映射
    NEW_DOCKER_CMD=$(echo "$NEW_DOCKER_CMD" | sed -E "s/-p $HOST_PORT:$CONTAINER_PORT/-p $RANDOM_PORT:$CONTAINER_PORT/")
done <<< "$PORT_MAPPINGS"

# 输出修改后的命令并询问是否执行
echo ""
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"
echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
else
    echo "已取消执行。"
fi
```

我发现这个脚本在处理多个端口映射时可能会有问题，因为我们使用了简单的字符串替换。让我再次改进，使用更健壮的方法:

```bash
#!/bin/bash

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "请提供一个docker命令作为参数"
    echo "例如: ./random-port.sh \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    exit 1
fi

# 获取已占用的端口列表
OCCUPIED_PORTS=$(ss -tuln | grep LISTEN | awk '{print $5}' | cut -d: -f2 | sort -u)

# 定义要排除的特定端口
EXCLUDED_PORTS="80 443"

# 生成可用端口列表(1024-65535)
AVAILABLE_PORTS=()
for port in $(seq 1024 65535); do
    # 检查端口是否在已占用列表或排除列表中
    if ! echo "$OCCUPIED_PORTS $EXCLUDED_PORTS" | grep -q -w "$port"; then
        AVAILABLE_PORTS+=($port)
    fi
done

# 获取原始docker命令
DOCKER_CMD="$@"

# 分割docker命令为数组
IFS=' ' read -r -a CMD_ARRAY <<< "$DOCKER_CMD"

# 查找所有端口映射参数的索引
PORT_INDICES=()
for i in "${!CMD_ARRAY[@]}"; do
    if [ "${CMD_ARRAY[$i]}" = "-p" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_INDICES+=($i)
    fi
done

# 如果没有找到端口映射
if [ ${#PORT_INDICES[@]} -eq 0 ]; then
    echo "未找到端口映射，请确保docker命令包含类似 -p 80:80 的端口映射。"
    exit 1
fi

# 处理每个端口映射
echo "端口映射替换计划:"
for idx in "${PORT_INDICES[@]}"; do
    MAPPING_IDX=$((idx+1))
    MAPPING="${CMD_ARRAY[$MAPPING_IDX]}"
    
    # 提取主机端口和容器端口
    HOST_PORT=$(echo $MAPPING | cut -d: -f1)
    CONTAINER_PORT=$(echo $MAPPING | cut -d: -f2)
    
    # 从可用端口列表中随机选择一个
    RANDOM_INDEX=$((RANDOM % ${#AVAILABLE_PORTS[@]}))
    RANDOM_PORT=${AVAILABLE_PORTS[$RANDOM_INDEX]}
    
    # 从可用端口列表中移除已选择的端口
    unset AVAILABLE_PORTS[$RANDOM_INDEX]
    AVAILABLE_PORTS=(${AVAILABLE_PORTS[@]})
    
    echo "  原端口映射: $HOST_PORT:$CONTAINER_PORT -> 新端口映射: $RANDOM_PORT:$CONTAINER_PORT"
    
    # 更新命令数组中的端口映射
    CMD_ARRAY[$MAPPING_IDX]="$RANDOM_PORT:$CONTAINER_PORT"
done

# 重建docker命令
NEW_DOCKER_CMD="${CMD_ARRAY[*]}"

# 输出修改后的命令并询问是否执行
echo ""
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"
echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
else
    echo "已取消执行。"
fi
```

我注意到这个脚本在处理端口映射时可能还有一些边缘情况没有考虑到，比如当端口映射格式为`-p=80:80`或`--publish 80:80`时。让我再次改进:

```bash
#!/bin/bash

# 设置语言环境为中文
export LANG=zh_CN.UTF-8

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "请提供一个docker命令作为参数"
    echo "例如: ./random-port.sh \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    exit 1
fi

# 获取已占用的端口列表
echo "正在检查系统已占用端口..."
OCCUPIED_PORTS=$(ss -tuln | grep LISTEN | awk '{print $5}' | cut -d: -f2 | sort -u)

# 定义要排除的特定端口
EXCLUDED_PORTS="80 443"

# 生成可用端口列表(1024-65535)
echo "正在生成可用端口列表..."
AVAILABLE_PORTS=()
for port in $(seq 1024 65535); do
    # 检查端口是否在已占用列表或排除列表中
    if ! echo "$OCCUPIED_PORTS $EXCLUDED_PORTS" | grep -q -w "$port"; then
        AVAILABLE_PORTS+=($port)
    fi
done

# 获取原始docker命令
DOCKER_CMD="$@"

# 使用正则表达式提取所有端口映射
PORT_MAPPINGS=$(echo "$DOCKER_CMD" | grep -o -E '(-p|--publish)[ =][0-9]+:[0-9]+')

if [ -z "$PORT_MAPPINGS" ]; then
    echo "未找到端口映射，请确保docker命令包含类似 -p 80:80 或 --publish 80:80 的端口映射。"
    exit 1
fi

NEW_DOCKER_CMD="$DOCKER_CMD"

# 处理每个端口映射
echo "端口映射替换计划:"
while read -r mapping; do
    # 提取参数类型(-p 或 --publish)和映射
    PARAM_TYPE=$(echo "$mapping" | grep -o -E '(-p|--publish)')
    PORT_MAP=$(echo "$mapping" | grep -o -E '[0-9]+:[0-9]+')
    
    # 提取主机端口和容器端口
    HOST_PORT=$(echo "$PORT_MAP" | cut -d: -f1)
    CONTAINER_PORT=$(echo "$PORT_MAP" | cut -d: -f2)
    
    # 从可用端口列表中随机选择一个
    RANDOM_INDEX=$((RANDOM % ${#AVAILABLE_PORTS[@]}))
    RANDOM_PORT=${AVAILABLE_PORTS[$RANDOM_INDEX]}
    
    # 从可用端口列表中移除已选择的端口
    unset AVAILABLE_PORTS[$RANDOM_INDEX]
    AVAILABLE_PORTS=(${AVAILABLE_PORTS[@]})
    
    echo "  原端口映射: $HOST_PORT:$CONTAINER_PORT -> 新端口映射: $RANDOM_PORT:$CONTAINER_PORT"
    
    # 替换docker命令中的端口映射
    NEW_DOCKER_CMD=$(echo "$NEW_DOCKER_CMD" | sed -E "s/$PARAM_TYPE[ =]$HOST_PORT:$CONTAINER_PORT/$PARAM_TYPE $RANDOM_PORT:$CONTAINER_PORT/")
done <<< "$PORT_MAPPINGS"

# 输出修改后的命令并询问是否执行
echo ""
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"
echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
else
    echo "已取消执行。"
fi
```

我觉得这个脚本还可以再优化一下，特别是在处理端口映射的正则表达式部分。让我再次改进:

```bash
#!/bin/bash

# 设置语言环境为中文
export LANG=zh_CN.UTF-8

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "请提供一个docker命令作为参数"
    echo "例如: ./random-port.sh \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    exit 1
fi

# 获取已占用的端口列表
echo "正在检查系统已占用端口..."
OCCUPIED_PORTS=$(ss -tuln | grep LISTEN | awk '{print $5}' | cut -d: -f2 | sort -u)

# 定义要排除的特定端口
EXCLUDED_PORTS="80 443"

# 生成可用端口列表(1024-65535)
echo "正在生成可用端口列表..."
AVAILABLE_PORTS=()
for port in $(seq 1024 65535); do
    # 检查端口是否在已占用列表或排除列表中
    if ! echo "$OCCUPIED_PORTS $EXCLUDED_PORTS" | grep -q -w "$port"; then
        AVAILABLE_PORTS+=($port)
    fi
done

# 获取原始docker命令
DOCKER_CMD="$@"

# 分割docker命令为数组
IFS=' ' read -r -a CMD_ARRAY <<< "$DOCKER_CMD"

# 查找所有端口映射
PORT_MAPPINGS=()
for i in "${!CMD_ARRAY[@]}"; do
    # 检查 -p 参数
    if [ "${CMD_ARRAY[$i]}" = "-p" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 -p=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^-p= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
    
    # 检查 --publish 参数
    if [ "${CMD_ARRAY[$i]}" = "--publish" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 --publish=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^--publish= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
done

if [ ${#PORT_MAPPINGS[@]} -eq 0 ]; then
    echo "未找到端口映射，请确保docker命令包含类似 -p 80:80 或 --publish 80:80 的端口映射。"
    exit 1
fi

NEW_DOCKER_CMD="$DOCKER_CMD"

# 处理每个端口映射
echo "端口映射替换计划:"
for mapping in "${PORT_MAPPINGS[@]}"; do
    # 提取端口映射部分
    if [[ "$mapping" =~ (-p|--publish)=([0-9]+):([0-9]+) ]]; then
        # 处理 -p=XXX:XXX 或 --publish=XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE}=${HOST_PORT}:${CONTAINER_PORT}"
    elif [[ "$mapping" =~ (-p|--publish)\ ([0-9]+):([0-9]+) ]]; then
        # 处理 -p XXX:XXX 或 --publish XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE} ${HOST_PORT}:${CONTAINER_PORT}"
    else
        echo "无法解析端口映射: $mapping"
        continue
    fi
    
    # 从可用端口列表中随机选择一个
    RANDOM_INDEX=$((RANDOM % ${#AVAILABLE_PORTS[@]}))
    RANDOM_PORT=${AVAILABLE_PORTS[$RANDOM_INDEX]}
    
    # 从可用端口列表中移除已选择的端口
    unset AVAILABLE_PORTS[$RANDOM_INDEX]
    AVAILABLE_PORTS=(${AVAILABLE_PORTS[@]})
    
    # 创建新的端口映射
    if [[ "$FULL_MAPPING" =~ = ]]; then
        NEW_MAPPING="${PARAM_TYPE}=${RANDOM_PORT}:${CONTAINER_PORT}"
    else
        NEW_MAPPING="${PARAM_TYPE} ${RANDOM_PORT}:${CONTAINER_PORT}"
    fi
    
    echo "  原端口映射: $FULL_MAPPING -> 新端口映射: $NEW_MAPPING"
    
    # 替换docker命令中的端口映射
    NEW_DOCKER_CMD="${NEW_DOCKER_CMD//$FULL_MAPPING/$NEW_MAPPING}"
done

# 输出修改后的命令并询问是否执行
echo ""
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"
echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
else
    echo "已取消执行。"
fi
```

我觉得这个脚本已经相当完善了，但还可以添加一些额外的功能和改进:

1. 添加帮助信息
2. 添加版本信息
3. 添加更多的错误处理
4. 添加日志记录
5. 添加更多的选项，如指定端口范围、排除特定端口等

让我再次改进:

```bash
#!/bin/bash

# 设置语言环境为中文
export LANG=zh_CN.UTF-8

VERSION="1.0.0"

# 显示帮助信息
show_help() {
    echo "随机端口分配工具 v$VERSION"
    echo "用法: $0 [选项] <docker命令>"
    echo ""
    echo "选项:"
    echo "  -h, --help        显示此帮助信息"
    echo "  -v, --version     显示版本信息"
    echo "  -r, --range       指定端口范围 (默认: 1024-65535)"
    echo "  -e, --exclude     指定要排除的端口 (默认: 80,443)"
    echo "  -d, --dry-run     仅显示修改后的命令，不执行"
    echo ""
    echo "示例:"
    echo "  $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "  $0 --range 8000-9000 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "  $0 --exclude 80,443,3306,5432 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo ""
}

# 显示版本信息
show_version() {
    echo "随机端口分配工具 v$VERSION"
}

# 默认值
PORT_RANGE_MIN=1024
PORT_RANGE_MAX=65535
EXCLUDED_PORTS="80 443"
DRY_RUN=false

# 解析命令行参数
while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)
            show_help
            exit 0
            ;;
        -v|--version)
            show_version
            exit 0
            ;;
        -r|--range)
            if [[ $2 =~ ^([0-9]+)-([0-9]+)$ ]]; then
                PORT_RANGE_MIN="${BASH_REMATCH[1]}"
                PORT_RANGE_MAX="${BASH_REMATCH[2]}"
                shift 2
            else
                echo "错误: 端口范围格式不正确，应为 MIN-MAX"
                exit 1
            fi
            ;;
        -e|--exclude)
            EXCLUDED_PORTS="${2//,/ }"
            shift 2
            ;;
        -d|--dry-run)
            DRY_RUN=true
            shift
            ;;
        *)
            # 假设剩余的参数是docker命令
            break
            ;;
    esac
done

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "错误: 未提供docker命令"
    echo "请提供一个docker命令作为参数"
    echo "例如: $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "使用 -h 或 --help 查看帮助信息"
    exit 1
fi

# 获取已占用的端口列表
echo "正在检查系统已占用端口..."
OCCUPIED_PORTS=$(ss -tuln | grep LISTEN | awk '{print $5}' | cut -d: -f2 | sort -u)

# 生成可用端口列表
echo "正在生成可用端口列表 (范围: $PORT_RANGE_MIN-$PORT_RANGE_MAX, 排除: $EXCLUDED_PORTS)..."
AVAILABLE_PORTS=()
for port in $(seq $PORT_RANGE_MIN $PORT_RANGE_MAX); do
    # 检查端口是否在已占用列表或排除列表中
    if ! echo "$OCCUPIED_PORTS $EXCLUDED_PORTS" | grep -q -w "$port"; then
        AVAILABLE_PORTS+=($port)
    fi
done

# 检查是否有可用端口
if [ ${#AVAILABLE_PORTS[@]} -eq 0 ]; then
    echo "错误: 在指定范围内没有可用端口"
    exit 1
fi

# 获取原始docker命令
DOCKER_CMD="$@"

# 分割docker命令为数组
IFS=' ' read -r -a CMD_ARRAY <<< "$DOCKER_CMD"

# 查找所有端口映射
PORT_MAPPINGS=()
for i in "${!CMD_ARRAY[@]}"; do
    # 检查 -p 参数
    if [ "${CMD_ARRAY[$i]}" = "-p" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 -p=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^-p= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
    
    # 检查 --publish 参数
    if [ "${CMD_ARRAY[$i]}" = "--publish" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 --publish=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^--publish= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
done

if [ ${#PORT_MAPPINGS[@]} -eq 0 ]; then
    echo "错误: 未找到端口映射"
    echo "请确保docker命令包含类似 -p 80:80 或 --publish 80:80 的端口映射"
    exit 1
fi

NEW_DOCKER_CMD="$DOCKER_CMD"

# 处理每个端口映射
echo "端口映射替换计划:"
for mapping in "${PORT_MAPPINGS[@]}"; do
    # 提取端口映射部分
    if [[ "$mapping" =~ (-p|--publish)=([0-9]+):([0-9]+) ]]; then
        # 处理 -p=XXX:XXX 或 --publish=XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE}=${HOST_PORT}:${CONTAINER_PORT}"
    elif [[ "$mapping" =~ (-p|--publish)\ ([0-9]+):([0-9]+) ]]; then
        # 处理 -p XXX:XXX 或 --publish XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE} ${HOST_PORT}:${CONTAINER_PORT}"
    else
        echo "警告: 无法解析端口映射: $mapping"
        continue
    fi
    
    # 从可用端口列表中随机选择一个
    RANDOM_INDEX=$((RANDOM % ${#AVAILABLE_PORTS[@]}))
    RANDOM_PORT=${AVAILABLE_PORTS[$RANDOM_INDEX]}
    
    # 从可用端口列表中移除已选择的端口
    unset AVAILABLE_PORTS[$RANDOM_INDEX]
    AVAILABLE_PORTS=(${AVAILABLE_PORTS[@]})
    
    # 创建新的端口映射
    if [[ "$FULL_MAPPING" =~ = ]]; then
        NEW_MAPPING="${PARAM_TYPE}=${RANDOM_PORT}:${CONTAINER_PORT}"
    else
        NEW_MAPPING="${PARAM_TYPE} ${RANDOM_PORT}:${CONTAINER_PORT}"
    fi
    
    echo "  原端口映射: $FULL_MAPPING -> 新端口映射: $NEW_MAPPING"
    
    # 替换docker命令中的端口映射
    NEW_DOCKER_CMD="${NEW_DOCKER_CMD//$FULL_MAPPING/$NEW_MAPPING}"
done

# 输出修改后的命令并询问是否执行
echo ""
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"

if [ "$DRY_RUN" = true ]; then
    echo "仅显示模式，不执行命令"
    exit 0
fi

echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
    
    # 检查命令执行状态
    if [ $? -eq 0 ]; then
        echo "命令执行成功!"
    else
        echo "命令执行失败，请检查错误信息"
    fi
else
    echo "已取消执行"
fi
```

这个脚本现在已经相当完善了，它可以:

1. 检测系统中已占用的端口
2. 排除特定的端口(默认是80和443)
3. 从可用端口中随机选择
4. 处理各种格式的docker端口映射
5. 提供帮助信息和版本信息
6. 支持指定端口范围
7. 支持排除特定端口
8. 支持仅显示模式(不执行命令)
9. 提供中文输出
10. 询问是否采纳端口映射并执行命令

用户可以将这个脚本保存为`random-port.sh`，然后使用`chmod +x random-port.sh`使其可执行，最后就可以使用它来随机分配端口了。

这个脚本应该能够满足用户的需求，它可以在任何Linux系统(特别是Ubuntu)上运行，只需要确保系统中安装了`ss`命令(大多数现代Linux发行版都默认安装了这个命令)。

总结一下，这个脚本的主要功能是:
1. 检测系统中已占用的端口
2. 排除特定的端口(默认是80和443)
3. 从可用端口中随机选择
4. 修改docker命令中的端口映射
5. 询问是否采纳端口映射并执行命令

这个脚本应该能够很好地解决用户的端口选择困难症问题。
```

# 随机端口分配工具

我可以为你创建一个适配Linux的端口随机生成工具，特别适合解决你提到的选择困难症问题。这个工具可以自动检测系统中可用的端口，排除80、443以及已被占用的端口，然后随机分配一个可用端口给你的Docker项目。

下面是一个完整的shell脚本，你可以将其保存为`random-port.sh`：

```bash
#!/bin/bash

# 设置语言环境为中文
export LANG=zh_CN.UTF-8

VERSION="1.0.0"

# 显示帮助信息
show_help() {
    echo "随机端口分配工具 v$VERSION"
    echo "用法: $0 [选项] <docker命令>"
    echo ""
    echo "选项:"
    echo "  -h, --help        显示此帮助信息"
    echo "  -v, --version     显示版本信息"
    echo "  -r, --range       指定端口范围 (默认: 1024-65535)"
    echo "  -e, --exclude     指定要排除的端口 (默认: 80,443)"
    echo "  -d, --dry-run     仅显示修改后的命令，不执行"
    echo ""
    echo "示例:"
    echo "  $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "  $0 --range 8000-9000 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "  $0 --exclude 80,443,3306,5432 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo ""
}

# 显示版本信息
show_version() {
    echo "随机端口分配工具 v$VERSION"
}

# 默认值
PORT_RANGE_MIN=1024
PORT_RANGE_MAX=65535
EXCLUDED_PORTS="80 443"
DRY_RUN=false

# 解析命令行参数
while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)
            show_help
            exit 0
            ;;
        -v|--version)
            show_version
            exit 0
            ;;
        -r|--range)
            if [[ $2 =~ ^([0-9]+)-([0-9]+)$ ]]; then
                PORT_RANGE_MIN="${BASH_REMATCH[1]}"
                PORT_RANGE_MAX="${BASH_REMATCH[2]}"
                shift 2
            else
                echo "错误: 端口范围格式不正确，应为 MIN-MAX"
                exit 1
            fi
            ;;
        -e|--exclude)
            EXCLUDED_PORTS="${2//,/ }"
            shift 2
            ;;
        -d|--dry-run)
            DRY_RUN=true
            shift
            ;;
        *)
            # 假设剩余的参数是docker命令
            break
            ;;
    esac
done

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "错误: 未提供docker命令"
    echo "请提供一个docker命令作为参数"
    echo "例如: $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "使用 -h 或 --help 查看帮助信息"
    exit 1
fi

# 获取已占用的端口列表
echo "正在检查系统已占用端口..."
OCCUPIED_PORTS=$(ss -tuln | grep LISTEN | awk '{print $5}' | cut -d: -f2 | sort -u)

# 生成可用端口列表
echo "正在生成可用端口列表 (范围: $PORT_RANGE_MIN-$PORT_RANGE_MAX, 排除: $EXCLUDED_PORTS)..."
AVAILABLE_PORTS=()
for port in $(seq $PORT_RANGE_MIN $PORT_RANGE_MAX); do
    # 检查端口是否在已占用列表或排除列表中
    if ! echo "$OCCUPIED_PORTS $EXCLUDED_PORTS" | grep -q -w "$port"; then
        AVAILABLE_PORTS+=($port)
    fi
done

# 检查是否有可用端口
if [ ${#AVAILABLE_PORTS[@]} -eq 0 ]; then
    echo "错误: 在指定范围内没有可用端口"
    exit 1
fi

# 获取原始docker命令
DOCKER_CMD="$@"

# 分割docker命令为数组
IFS=' ' read -r -a CMD_ARRAY <<< "$DOCKER_CMD"

# 查找所有端口映射
PORT_MAPPINGS=()
for i in "${!CMD_ARRAY[@]}"; do
    # 检查 -p 参数
    if [ "${CMD_ARRAY[$i]}" = "-p" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 -p=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^-p= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
    
    # 检查 --publish 参数
    if [ "${CMD_ARRAY[$i]}" = "--publish" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 --publish=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^--publish= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
done

if [ ${#PORT_MAPPINGS[@]} -eq 0 ]; then
    echo "错误: 未找到端口映射"
    echo "请确保docker命令包含类似 -p 80:80 或 --publish 80:80 的端口映射"
    exit 1
fi

NEW_DOCKER_CMD="$DOCKER_CMD"

# 处理每个端口映射
echo "端口映射替换计划:"
for mapping in "${PORT_MAPPINGS[@]}"; do
    # 提取端口映射部分
    if [[ "$mapping" =~ (-p|--publish)=([0-9]+):([0-9]+) ]]; then
        # 处理 -p=XXX:XXX 或 --publish=XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE}=${HOST_PORT}:${CONTAINER_PORT}"
    elif [[ "$mapping" =~ (-p|--publish)\ ([0-9]+):([0-9]+) ]]; then
        # 处理 -p XXX:XXX 或 --publish XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE} ${HOST_PORT}:${CONTAINER_PORT}"
    else
        echo "警告: 无法解析端口映射: $mapping"
        continue
    fi
    
    # 从可用端口列表中随机选择一个
    RANDOM_INDEX=$((RANDOM % ${#AVAILABLE_PORTS[@]}))
    RANDOM_PORT=${AVAILABLE_PORTS[$RANDOM_INDEX]}
    
    # 从可用端口列表中移除已选择的端口
    unset AVAILABLE_PORTS[$RANDOM_INDEX]
    AVAILABLE_PORTS=(${AVAILABLE_PORTS[@]})
    
    # 创建新的端口映射
    if [[ "$FULL_MAPPING" =~ = ]]; then
        NEW_MAPPING="${PARAM_TYPE}=${RANDOM_PORT}:${CONTAINER_PORT}"
    else
        NEW_MAPPING="${PARAM_TYPE} ${RANDOM_PORT}:${CONTAINER_PORT}"
    fi
    
    echo "  原端口映射: $FULL_MAPPING -> 新端口映射: $NEW_MAPPING"
    
    # 替换docker命令中的端口映射
    NEW_DOCKER_CMD="${NEW_DOCKER_CMD//$FULL_MAPPING/$NEW_MAPPING}"
done

# 输出修改后的命令并询问是否执行
echo ""
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"

if [ "$DRY_RUN" = true ]; then
    echo "仅显示模式，不执行命令"
    exit 0
fi

echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
    
    # 检查命令执行状态
    if [ $? -eq 0 ]; then
        echo "命令执行成功!"
    else
        echo "命令执行失败，请检查错误信息"
    fi
else
    echo "已取消执行"
fi
```

## 使用方法

1. 将上面的脚本保存为`random-port.sh`文件
2. 给脚本添加执行权限：
   ```bash
   chmod +x random-port.sh
   ```
3. 运行脚本，传入你的docker命令：
   ```bash
   ./random-port.sh "docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool"
   ```

## 功能特点

- 自动检测系统已占用端口
- 默认排除80和443端口
- 支持自定义端口范围和排除列表
- 支持多种端口映射格式（-p、--publish等）
- 中文交互界面
- 询问是否采纳随机分配的端口
- 支持仅显示模式（不执行命令）

## 高级选项

- 指定端口范围：
  ```bash
  ./random-port.sh --range 8000-9000 "docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool"
  ```

- 排除特定端口：
  ```bash
  ./random-port.sh --exclude 80,443,3306,5432 "docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool"
  ```

- 仅显示修改后的命令，不执行：
  ```bash
  ./random-port.sh --dry-run "docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool"
  ```

这个脚本应该能很好地解决你的端口选择困难症问题，让你在部署新项目时不再为端口选择而烦恼。

-------

Role: Human
root@ser249578744169:/data/neko-api-key-tool/neko-api-key-tool#docker build -t neko-api-key-tool .
[+] Building 160.0s (14/14) FINISHED                                                                                                                                                                                                                                                                                                            docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                                                                                                                                                                      0.0s
 => => transferring dockerfile: 365B                                                                                                                                                                                                                                                                                                                      0.0s
 => WARN: FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 1)                                                                                                                                                                                                                                                                            0.0s
 => [internal] load metadata for docker.io/library/nginx:1.19.0-alpine                                                                                                                                                                                                                                                                                    6.9s
 => [internal] load metadata for docker.io/library/node:16                                                                                                                                                                                                                                                                                                0.9s
 => [internal] load .dockerignore                                                                                                                                                                                                                                                                                                                         0.0s
 => => transferring context: 2B                                                                                                                                                                                                                                                                                                                           0.0s
 => [builder 1/6] FROM docker.io/library/node:16@sha256:f77a1aef2da8d83e45ec990f45df50f1a286c5fe8bbfb8c6e4246c6389705c0b                                                                                                                                                                                                                                 22.7s
 => => resolve docker.io/library/node:16@sha256:f77a1aef2da8d83e45ec990f45df50f1a286c5fe8bbfb8c6e4246c6389705c0b                                                                                                                                                                                                                                          0.0s
 => => sha256:1ddc7e4055fdb6f6bf31063b593befda814294f9f904b6ddfc21ab1513bafa8e 7.23kB / 7.23kB                                                                                                                                                                                                                                                            0.0s
 => => sha256:f77a1aef2da8d83e45ec990f45df50f1a286c5fe8bbfb8c6e4246c6389705c0b 776B / 776B                                                                                                                                                                                                                                                                0.0s
 => => sha256:c94b82f9827cab6e421b350965a9ef11b25b13ffbd1030536203d541f55dcbe2 2.00kB / 2.00kB                                                                                                                                                                                                                                                            0.0s
 => => sha256:311da6c465ea1576925360eba391bcd32dece9be95960a0bc9ffcb25fe712017 50.50MB / 50.50MB                                                                                                                                                                                                                                                          2.4s
 => => sha256:7e9bf114588c05b2df612b083b96582f3b8dbf51647aa6138a50d09d42df2454 17.58MB / 17.58MB                                                                                                                                                                                                                                                          1.0s
 => => sha256:ffd9397e94b74abcb54e514f1430e00f604328d1f895eadbd482f08cc02444e5 51.89MB / 51.89MB                                                                                                                                                                                                                                                          3.6s
 => => sha256:513d779256048c961239af5f500589330546b072775217272e19ffae1635e98e 191.90MB / 191.90MB                                                                                                                                                                                                                                                        6.4s
 => => extracting sha256:311da6c465ea1576925360eba391bcd32dece9be95960a0bc9ffcb25fe712017                                                                                                                                                                                                                                                                 3.8s
 => => sha256:ae3b95bbaa61ce24cefdd89e7c74d6fbd7713b2bcae93af47063d06bd7e02172 4.20kB / 4.20kB                                                                                                                                                                                                                                                            2.5s
 => => sha256:0e421f66aff42bb069dffc26af6d132194b22a1082b08c5ef7cd69c627783c04 34.79MB / 34.79MB                                                                                                                                                                                                                                                          3.6s
 => => sha256:ca266fd6192108b67fb57b74753a8c4ca5d8bd458baae3d4df7ce9f42dedcc1d 2.27MB / 2.27MB                                                                                                                                                                                                                                                            3.8s
 => => sha256:ee7d78be1eb92caf6ae84fc3af736b23eca018d5dedc967ae5bdee6d7082403b 450B / 450B                                                                                                                                                                                                                                                                3.8s
 => => extracting sha256:7e9bf114588c05b2df612b083b96582f3b8dbf51647aa6138a50d09d42df2454                                                                                                                                                                                                                                                                 0.6s
 => => extracting sha256:ffd9397e94b74abcb54e514f1430e00f604328d1f895eadbd482f08cc02444e5                                                                                                                                                                                                                                                                 3.2s
 => => extracting sha256:513d779256048c961239af5f500589330546b072775217272e19ffae1635e98e                                                                                                                                                                                                                                                                 9.6s
 => => extracting sha256:ae3b95bbaa61ce24cefdd89e7c74d6fbd7713b2bcae93af47063d06bd7e02172                                                                                                                                                                                                                                                                 0.0s
 => => extracting sha256:0e421f66aff42bb069dffc26af6d132194b22a1082b08c5ef7cd69c627783c04                                                                                                                                                                                                                                                                 2.0s
 => => extracting sha256:ca266fd6192108b67fb57b74753a8c4ca5d8bd458baae3d4df7ce9f42dedcc1d                                                                                                                                                                                                                                                                 0.1s
 => => extracting sha256:ee7d78be1eb92caf6ae84fc3af736b23eca018d5dedc967ae5bdee6d7082403b                                                                                                                                                                                                                                                                 0.0s
 => [stage-1 1/2] FROM docker.io/library/nginx:1.19.0-alpine@sha256:17ba9c1ca3dbea0b44f7c890862161c4976f66ebb33e8fdd4452417aebba8e64                                                                                                                                                                                                                      1.6s
 => => resolve docker.io/library/nginx:1.19.0-alpine@sha256:17ba9c1ca3dbea0b44f7c890862161c4976f66ebb33e8fdd4452417aebba8e64                                                                                                                                                                                                                              0.0s
 => => sha256:17ba9c1ca3dbea0b44f7c890862161c4976f66ebb33e8fdd4452417aebba8e64 1.65kB / 1.65kB                                                                                                                                                                                                                                                            0.0s
 => => sha256:ee5a9b68e8d4a4b8b48318ff08ad5489bd1ce52b357bf48c511968a302bc347b 1.36kB / 1.36kB                                                                                                                                                                                                                                                            0.0s
 => => sha256:7d0cdcc60a96a5124763fddf5d534d058ad7d0d8d4c3b8be2aefedf4267d0270 8.03kB / 8.03kB                                                                                                                                                                                                                                                            0.0s
 => => sha256:cbdbe7a5bc2a134ca8ec91be58565ec07d037386d1f1d8385412d224deafca08 2.81MB / 2.81MB                                                                                                                                                                                                                                                            0.3s
 => => sha256:10c113fb0c778963cb3069e94e8148a3770122f6763c94373e22f5342b503ab0 6.46MB / 6.46MB                                                                                                                                                                                                                                                            0.4s
 => => sha256:9ba64393807bf2549af97a1a074ca5fff1bce25ad115b0a7ced446cd1b4305d0 538B / 538B                                                                                                                                                                                                                                                                0.3s
 => => extracting sha256:cbdbe7a5bc2a134ca8ec91be58565ec07d037386d1f1d8385412d224deafca08                                                                                                                                                                                                                                                                 0.1s
 => => sha256:262f9908119d4529a370bcdf1f1306131ad556edf400413d5fa74008d7919931 899B / 899B                                                                                                                                                                                                                                                                0.4s
 => => sha256:c4a057508f96954546441044f0d2373303862a4d4accc163e68a4c30d0c88869 668B / 668B                                                                                                                                                                                                                                                                0.4s
 => => extracting sha256:10c113fb0c778963cb3069e94e8148a3770122f6763c94373e22f5342b503ab0                                                                                                                                                                                                                                                                 0.9s
 => => extracting sha256:9ba64393807bf2549af97a1a074ca5fff1bce25ad115b0a7ced446cd1b4305d0                                                                                                                                                                                                                                                                 0.0s
 => => extracting sha256:262f9908119d4529a370bcdf1f1306131ad556edf400413d5fa74008d7919931                                                                                                                                                                                                                                                                 0.0s
 => => extracting sha256:c4a057508f96954546441044f0d2373303862a4d4accc163e68a4c30d0c88869                                                                                                                                                                                                                                                                 0.0s
 => [internal] load build context                                                                                                                                                                                                                                                                                                                         0.1s
 => => transferring context: 1.43MB                                                                                                                                                                                                                                                                                                                       0.1s
 => [builder 2/6] WORKDIR /build                                                                                                                                                                                                                                                                                                                          0.1s
 => [builder 3/6] COPY . /app                                                                                                                                                                                                                                                                                                                             0.1s
 => [builder 4/6] WORKDIR /app                                                                                                                                                                                                                                                                                                                            0.0s
 => [builder 5/6] RUN npm install                                                                                                                                                                                                                                                                                                                        47.6s
 => [builder 6/6] RUN npm run build                                                                                                                                                                                                                                                                                                                      79.8s 
 => [stage-1 2/2] COPY --from=builder /app/build /usr/share/nginx/html                                                                                                                                                                                                                                                                                    0.1s 
 => exporting to image                                                                                                                                                                                                                                                                                                                                    0.1s 
 => => exporting layers                                                                                                                                                                                                                                                                                                                                   0.1s 
 => => writing image sha256:1ace7d82aae6cef4f8fccc4213b26c374d4bc1f27bc202f6417cd706c234d1f9                                                                                                                                                                                                                                                              0.0s 
 => => naming to docker.io/library/neko-api-key-tool                                                                                                                                                                                                                                                                                                      0.0s 
                                                                                                                                                                                                                                                                                                                                                               
 1 warning found (use docker --debug to expand):
 - FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 1)
是否存在问题

-------

Role: AI
```thinking
让我分析一下这个Docker构建输出，看看是否存在问题。

首先，我看到构建过程已经完成，最后显示"FINISHED"，并且成功创建了镜像 `docker.io/library/neko-api-key-tool`。这表明构建基本上是成功的。

不过，我注意到有一个警告：
```
1 warning found (use docker --debug to expand):
- FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 1)
```

这个警告是关于Dockerfile中的语法问题，具体是关于`FROM`和`as`关键字的大小写不匹配。在Dockerfile中，通常建议保持关键字大小写一致，例如都使用大写（`FROM`和`AS`）或都使用小写（`from`和`as`）。

这个警告不会影响镜像的功能，只是一个风格上的建议。如果想修复，可以编辑Dockerfile，确保这两个关键字的大小写一致。

从构建日志来看，这是一个多阶段构建的Dockerfile：
1. 第一阶段使用`node:16`作为基础镜像，用于构建应用
2. 第二阶段使用`nginx:1.19.0-alpine`作为基础镜像，用于部署应用

构建过程包括：
1. 复制应用代码
2. 安装npm依赖
3. 构建应用
4. 将构建结果复制到nginx容器中

整个构建过程花了约160秒，其中npm安装花了47.6秒，构建应用花了79.8秒，这些都是正常的时间范围。

除了那个小的警告外，我没有看到任何明显的错误或问题。镜像已经成功构建，并被命名为`neko-api-key-tool`。

所以，回答用户的问题：从构建日志来看，除了一个关于FROM和as关键字大小写不匹配的小警告外，没有其他问题。这个警告不会影响镜像的功能，只是一个代码风格的建议。
```

# Docker构建分析

从您提供的构建日志来看，Docker镜像已经成功构建完成，没有出现错误。构建过程总共用时约160秒，最终生成了名为`neko-api-key-tool`的镜像。

## 注意到的唯一警告

```
1 warning found (use docker --debug to expand):
- FromAsCasing: 'as' and 'FROM' keywords' casing do not match (line 1)
```

这是一个关于Dockerfile语法风格的警告，表示在Dockerfile的第一行中，`FROM`和`as`关键字的大小写不一致。例如，可能是这样的：

```dockerfile
FROM node:16 as builder
```

而Docker建议保持关键字大小写一致，例如：

```dockerfile
FROM node:16 AS builder
```

## 是否存在问题？

这个警告不会影响镜像的功能或运行，只是一个代码风格的建议。镜像已经成功构建，您可以正常使用它。

如果您想修复这个警告，只需编辑Dockerfile，确保`FROM`和`as`关键字的大小写一致即可。

您现在可以使用之前提供的随机端口工具来运行这个镜像：

```bash
./random-port.sh "docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool"
```

这将为您的容器分配一个随机可用端口，避免端口冲突问题。

-------

Role: Human
root@ser249578744169:/data/neko-api-key-tool/neko-api-key-tool# ./random-port.sh --dry-run "docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool"
-bash: ./random-port.sh: No such file or directory
root@ser249578744169:/data/neko-api-key-tool/neko-api-key-tool# 
该工具好像没有办法在任何时候生效

-------

Role: AI
```thinking
看起来用户尝试运行我们之前提供的`random-port.sh`脚本，但遇到了"No such file or directory"错误。这表明脚本文件不存在或者没有正确创建。

问题在于，我们只是提供了脚本的代码，但用户需要先创建这个文件并赋予执行权限才能使用它。我们需要指导用户如何正确地创建和使用这个脚本。

用户说"该工具好像没有办法在任何时候生效"，这表明他们可能误解了我们的意思，以为这个工具是系统自带的或者已经安装好的。实际上，这是一个需要手动创建的自定义脚本。

让我提供详细的步骤，指导用户如何创建和使用这个脚本：

1. 首先，需要创建脚本文件
2. 然后，给脚本添加执行权限
3. 最后，运行脚本

我还应该解释一下，这个脚本不是系统自带的工具，而是需要手动创建的自定义脚本。一旦创建好，它就可以在任何Linux系统上使用，但前提是需要先创建这个文件。
```

# 创建和使用随机端口工具

看起来您尝试运行`random-port.sh`脚本，但遇到了"No such file or directory"错误。这是因为我们之前只提供了脚本的代码，但您需要先创建这个文件才能使用它。

## 创建脚本文件

请按照以下步骤创建和使用这个工具：

1. **创建脚本文件**：

   ```bash
   # 创建脚本文件
   nano random-port.sh
   ```

   然后将我之前提供的完整脚本代码粘贴到编辑器中。
   按 Ctrl+O 保存，然后按 Ctrl+X 退出。

2. **添加执行权限**：

   ```bash
   # 添加执行权限
   chmod +x random-port.sh
   ```

3. **运行脚本**：

   ```bash
   # 使用dry-run模式测试
   ./random-port.sh --dry-run "docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool"
   ```

## 如果没有nano编辑器

如果您的系统没有安装nano，可以使用以下命令直接创建脚本文件：

```bash
cat > random-port.sh << 'EOF'
#!/bin/bash

# 设置语言环境为中文
export LANG=zh_CN.UTF-8

VERSION="1.0.0"

# 显示帮助信息
show_help() {
    echo "随机端口分配工具 v$VERSION"
    echo "用法: $0 [选项] <docker命令>"
    echo ""
    echo "选项:"
    echo "  -h, --help        显示此帮助信息"
    echo "  -v, --version     显示版本信息"
    echo "  -r, --range       指定端口范围 (默认: 1024-65535)"
    echo "  -e, --exclude     指定要排除的端口 (默认: 80,443)"
    echo "  -d, --dry-run     仅显示修改后的命令，不执行"
    echo ""
    echo "示例:"
    echo "  $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "  $0 --range 8000-9000 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "  $0 --exclude 80,443,3306,5432 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo ""
}

# 显示版本信息
show_version() {
    echo "随机端口分配工具 v$VERSION"
}

# 默认值
PORT_RANGE_MIN=1024
PORT_RANGE_MAX=65535
EXCLUDED_PORTS="80 443"
DRY_RUN=false

# 解析命令行参数
while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)
            show_help
            exit 0
            ;;
        -v|--version)
            show_version
            exit 0
            ;;
        -r|--range)
            if [[ $2 =~ ^([0-9]+)-([0-9]+)$ ]]; then
                PORT_RANGE_MIN="${BASH_REMATCH[1]}"
                PORT_RANGE_MAX="${BASH_REMATCH[2]}"
                shift 2
            else
                echo "错误: 端口范围格式不正确，应为 MIN-MAX"
                exit 1
            fi
            ;;
        -e|--exclude)
            EXCLUDED_PORTS="${2//,/ }"
            shift 2
            ;;
        -d|--dry-run)
            DRY_RUN=true
            shift
            ;;
        *)
            # 假设剩余的参数是docker命令
            break
            ;;
    esac
done

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "错误: 未提供docker命令"
    echo "请提供一个docker命令作为参数"
    echo "例如: $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "使用 -h 或 --help 查看帮助信息"
    exit 1
fi

# 获取已占用的端口列表
echo "正在检查系统已占用端口..."
OCCUPIED_PORTS=$(ss -tuln | grep LISTEN | awk '{print $5}' | cut -d: -f2 | sort -u)

# 生成可用端口列表
echo "正在生成可用端口列表 (范围: $PORT_RANGE_MIN-$PORT_RANGE_MAX, 排除: $EXCLUDED_PORTS)..."
AVAILABLE_PORTS=()
for port in $(seq $PORT_RANGE_MIN $PORT_RANGE_MAX); do
    # 检查端口是否在已占用列表或排除列表中
    if ! echo "$OCCUPIED_PORTS $EXCLUDED_PORTS" | grep -q -w "$port"; then
        AVAILABLE_PORTS+=($port)
    fi
done

# 检查是否有可用端口
if [ ${#AVAILABLE_PORTS[@]} -eq 0 ]; then
    echo "错误: 在指定范围内没有可用端口"
    exit 1
fi

# 获取原始docker命令
DOCKER_CMD="$@"

# 分割docker命令为数组
IFS=' ' read -r -a CMD_ARRAY <<< "$DOCKER_CMD"

# 查找所有端口映射
PORT_MAPPINGS=()
for i in "${!CMD_ARRAY[@]}"; do
    # 检查 -p 参数
    if [ "${CMD_ARRAY[$i]}" = "-p" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 -p=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^-p= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
    
    # 检查 --publish 参数
    if [ "${CMD_ARRAY[$i]}" = "--publish" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 --publish=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^--publish= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
done

if [ ${#PORT_MAPPINGS[@]} -eq 0 ]; then
    echo "错误: 未找到端口映射"
    echo "请确保docker命令包含类似 -p 80:80 或 --publish 80:80 的端口映射"
    exit 1
fi

NEW_DOCKER_CMD="$DOCKER_CMD"

# 处理每个端口映射
echo "端口映射替换计划:"
for mapping in "${PORT_MAPPINGS[@]}"; do
    # 提取端口映射部分
    if [[ "$mapping" =~ (-p|--publish)=([0-9]+):([0-9]+) ]]; then
        # 处理 -p=XXX:XXX 或 --publish=XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE}=${HOST_PORT}:${CONTAINER_PORT}"
    elif [[ "$mapping" =~ (-p|--publish)\ ([0-9]+):([0-9]+) ]]; then
        # 处理 -p XXX:XXX 或 --publish XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE} ${HOST_PORT}:${CONTAINER_PORT}"
    else
        echo "警告: 无法解析端口映射: $mapping"
        continue
    fi
    
    # 从可用端口列表中随机选择一个
    RANDOM_INDEX=$((RANDOM % ${#AVAILABLE_PORTS[@]}))
    RANDOM_PORT=${AVAILABLE_PORTS[$RANDOM_INDEX]}
    
    # 从可用端口列表中移除已选择的端口
    unset AVAILABLE_PORTS[$RANDOM_INDEX]
    AVAILABLE_PORTS=(${AVAILABLE_PORTS[@]})
    
    # 创建新的端口映射
    if [[ "$FULL_MAPPING" =~ = ]]; then
        NEW_MAPPING="${PARAM_TYPE}=${RANDOM_PORT}:${CONTAINER_PORT}"
    else
        NEW_MAPPING="${PARAM_TYPE} ${RANDOM_PORT}:${CONTAINER_PORT}"
    fi
    
    echo "  原端口映射: $FULL_MAPPING -> 新端口映射: $NEW_MAPPING"
    
    # 替换docker命令中的端口映射
    NEW_DOCKER_CMD="${NEW_DOCKER_CMD//$FULL_MAPPING/$NEW_MAPPING}"
done

# 输出修改后的命令并询问是否执行
echo ""
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"

if [ "$DRY_RUN" = true ]; then
    echo "仅显示模式，不执行命令"
    exit 0
fi

echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
    
    # 检查命令执行状态
    if [ $? -eq 0 ]; then
        echo "命令执行成功!"
    else
        echo "命令执行失败，请检查错误信息"
    fi
else
    echo "已取消执行"
fi
EOF

chmod +x random-port.sh
```

## 关于工具的说明

这个工具不是系统自带的，而是一个自定义脚本。一旦创建好，它就可以在任何Linux系统上使用，但前提是需要先创建这个文件。

您可以将这个脚本复制到任何Linux系统中，只要确保：
1. 脚本文件存在
2. 脚本有执行权限
3. 系统中安装了基本的命令行工具（如ss、grep、awk等）

创建好脚本后，您就可以在任何需要部署Docker容器的时候使用它来随机分配端口了。

-------

Role: Human
我的意思是，比如我把它放在了/root目录
那我在其他目录的时候就不能使用这个命令了
能不能让他在任何路径下可用

-------

Role: AI
```thinking
用户想要让这个脚本在任何路径下都可用，这是一个关于将脚本添加到系统PATH的问题。在Linux系统中，要让一个命令在任何路径下都可用，通常有以下几种方法：

1. 将脚本移动到已经在PATH中的目录，如`/usr/local/bin/`
2. 创建一个符号链接到PATH中的目录
3. 将脚本所在的目录添加到PATH环境变量中
4. 创建一个系统级别的别名

我将详细解释这几种方法，让用户可以选择最适合他们的方式。

方法1是最标准的做法，将脚本放在`/usr/local/bin/`目录下，这个目录通常已经在系统PATH中。

方法2是创建一个符号链接，指向脚本的实际位置。

方法3是修改PATH环境变量，将脚本所在的目录添加到PATH中。

方法4是创建一个别名，但这种方法只对当前用户有效，而且需要在每次登录时重新加载。

我会详细解释每种方法的具体步骤，让用户可以根据自己的需求选择最合适的方法。
```

# 让脚本在任何路径下可用

您想让`random-port.sh`脚本在任何路径下都可用，这是一个很好的需求。在Linux系统中，有几种方法可以实现这一点：

## 方法1：移动到系统PATH目录（推荐）

将脚本移动到已经在系统PATH中的目录，如`/usr/local/bin/`：

```bash
# 移动脚本到系统目录
sudo mv /root/random-port.sh /usr/local/bin/random-port

# 确保它有执行权限
sudo chmod +x /usr/local/bin/random-port
```

现在，您可以在任何目录下直接使用`random-port`命令了。

注意：我去掉了`.sh`后缀，这样命令看起来更像系统命令。如果您想保留后缀，也可以。

## 方法2：创建符号链接

如果您想保留脚本在原始位置，可以创建一个符号链接：

```bash
# 创建符号链接到系统PATH目录
sudo ln -s /root/random-port.sh /usr/local/bin/random-port

# 确保原始脚本有执行权限
chmod +x /root/random-port.sh
```

## 方法3：添加到PATH环境变量

如果您不想移动脚本或创建链接，可以将脚本所在的目录添加到PATH环境变量中：

```bash
# 编辑.bashrc文件
echo 'export PATH=$PATH:/root' >> ~/.bashrc

# 重新加载.bashrc
source ~/.bashrc
```

注意：这种方法只对当前用户有效，而且将`/root`目录添加到PATH可能不是最佳实践，因为它包含许多系统文件。

## 方法4：创建别名

创建一个命令别名：

```bash
# 编辑.bashrc文件
echo 'alias random-port="/root/random-port.sh"' >> ~/.bashrc

# 重新加载.bashrc
source ~/.bashrc
```

## 方法5：创建包装脚本

在`/usr/local/bin`中创建一个小的包装脚本：

```bash
# 创建包装脚本
cat > /usr/local/bin/random-port << 'EOF'
#!/bin/bash
/root/random-port.sh "$@"
EOF

# 添加执行权限
chmod +x /usr/local/bin/random-port
```

## 使用示例

无论您选择哪种方法，现在都可以在任何目录下使用该命令：

```bash
# 使用dry-run模式测试
random-port --dry-run "docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool"

# 或者如果您保留了.sh后缀
random-port.sh --dry-run "docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool"
```

我个人推荐方法1或方法2，因为它们是最标准的做法，并且对所有用户都有效。

-------

Role: Human
优化代码，就是这个代码的性能不够，运行太慢了，检测可用端口的时候有很长时间，我希望他秒出，可以参考我的思路，就是先随机一个端口，然后测一下他有没有被占用，如果被占用了就换一个，如果没被占用，那就直接采用那个端口，这样可以带来更高的运行效率和更少的性能开销

-------

Role: AI
```thinking
用户提出了一个很好的优化思路。当前脚本的性能问题主要在于它会先生成所有可用端口的列表，这在端口范围很大时（如1024-65535）会消耗大量时间和内存。

用户建议的方法是：
1. 先随机选择一个端口
2. 检查该端口是否被占用
3. 如果被占用，则重新随机选择
4. 如果未被占用，则直接使用

这种方法确实更高效，因为它避免了预先生成完整的可用端口列表，而是采用"尝试直到成功"的策略。这在大多数情况下会快得多，因为通常只需要尝试几次就能找到可用端口。

让我重写脚本，采用这种优化策略：

1. 保留脚本的基本结构和功能
2. 替换端口选择逻辑，使用随机尝试的方法
3. 添加最大尝试次数限制，以防止无限循环
4. 保持其他功能不变

我会确保新脚本保留原有的所有功能，包括：
- 支持多种端口映射格式
- 支持排除特定端口
- 支持指定端口范围
- 支持dry-run模式
- 中文输出
- 询问是否采纳端口映射
```

# 优化后的随机端口分配工具

根据您的建议，我重写了脚本以提高性能。新版本使用"先随机选择，再检查可用性"的策略，而不是预先生成所有可用端口的列表，这样可以大大提高执行速度。

```bash
#!/bin/bash

# 设置语言环境为中文
export LANG=zh_CN.UTF-8

VERSION="2.0.0"

# 显示帮助信息
show_help() {
    echo "随机端口分配工具 v$VERSION"
    echo "用法: $0 [选项] <docker命令>"
    echo ""
    echo "选项:"
    echo "  -h, --help        显示此帮助信息"
    echo "  -v, --version     显示版本信息"
    echo "  -r, --range       指定端口范围 (默认: 1024-65535)"
    echo "  -e, --exclude     指定要排除的端口 (默认: 80,443)"
    echo "  -d, --dry-run     仅显示修改后的命令，不执行"
    echo "  -m, --max-tries   最大尝试次数 (默认: 50)"
    echo ""
    echo "示例:"
    echo "  $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "  $0 --range 8000-9000 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "  $0 --exclude 80,443,3306,5432 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo ""
}

# 显示版本信息
show_version() {
    echo "随机端口分配工具 v$VERSION"
}

# 检查端口是否被占用
is_port_used() {
    local port=$1
    # 使用ss命令检查端口是否被占用
    ss -tuln | grep -q ":$port "
    return $?
}

# 检查端口是否在排除列表中
is_port_excluded() {
    local port=$1
    local excluded_ports=$2
    for excluded in $excluded_ports; do
        if [ "$port" = "$excluded" ]; then
            return 0
        fi
    done
    return 1
}

# 获取随机可用端口
get_random_available_port() {
    local min_port=$1
    local max_port=$2
    local excluded_ports=$3
    local max_tries=$4
    local tries=0
    
    while [ $tries -lt $max_tries ]; do
        # 生成随机端口
        local random_port=$(( $RANDOM % ($max_port - $min_port + 1) + $min_port ))
        
        # 检查端口是否在排除列表中
        if is_port_excluded $random_port "$excluded_ports"; then
            tries=$((tries + 1))
            continue
        fi
        
        # 检查端口是否被占用
        if ! is_port_used $random_port; then
            echo $random_port
            return 0
        fi
        
        tries=$((tries + 1))
    done
    
    echo "错误: 在$max_tries次尝试后未找到可用端口" >&2
    return 1
}

# 默认值
PORT_RANGE_MIN=1024
PORT_RANGE_MAX=65535
EXCLUDED_PORTS="80 443"
DRY_RUN=false
MAX_TRIES=50

# 解析命令行参数
while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)
            show_help
            exit 0
            ;;
        -v|--version)
            show_version
            exit 0
            ;;
        -r|--range)
            if [[ $2 =~ ^([0-9]+)-([0-9]+)$ ]]; then
                PORT_RANGE_MIN="${BASH_REMATCH[1]}"
                PORT_RANGE_MAX="${BASH_REMATCH[2]}"
                shift 2
            else
                echo "错误: 端口范围格式不正确，应为 MIN-MAX"
                exit 1
            fi
            ;;
        -e|--exclude)
            EXCLUDED_PORTS="${2//,/ }"
            shift 2
            ;;
        -d|--dry-run)
            DRY_RUN=true
            shift
            ;;
        -m|--max-tries)
            MAX_TRIES=$2
            shift 2
            ;;
        *)
            # 假设剩余的参数是docker命令
            break
            ;;
    esac
done

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "错误: 未提供docker命令"
    echo "请提供一个docker命令作为参数"
    echo "例如: $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "使用 -h 或 --help 查看帮助信息"
    exit 1
fi

# 获取原始docker命令
DOCKER_CMD="$@"

# 分割docker命令为数组
IFS=' ' read -r -a CMD_ARRAY <<< "$DOCKER_CMD"

# 查找所有端口映射
PORT_MAPPINGS=()
for i in "${!CMD_ARRAY[@]}"; do
    # 检查 -p 参数
    if [ "${CMD_ARRAY[$i]}" = "-p" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 -p=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^-p= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
    
    # 检查 --publish 参数
    if [ "${CMD_ARRAY[$i]}" = "--publish" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 --publish=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^--publish= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
done

if [ ${#PORT_MAPPINGS[@]} -eq 0 ]; then
    echo "错误: 未找到端口映射"
    echo "请确保docker命令包含类似 -p 80:80 或 --publish 80:80 的端口映射"
    exit 1
fi

NEW_DOCKER_CMD="$DOCKER_CMD"

# 处理每个端口映射
echo "端口映射替换计划:"
for mapping in "${PORT_MAPPINGS[@]}"; do
    # 提取端口映射部分
    if [[ "$mapping" =~ (-p|--publish)=([0-9]+):([0-9]+) ]]; then
        # 处理 -p=XXX:XXX 或 --publish=XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE}=${HOST_PORT}:${CONTAINER_PORT}"
    elif [[ "$mapping" =~ (-p|--publish)\ ([0-9]+):([0-9]+) ]]; then
        # 处理 -p XXX:XXX 或 --publish XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE} ${HOST_PORT}:${CONTAINER_PORT}"
    else
        echo "警告: 无法解析端口映射: $mapping"
        continue
    fi
    
    # 获取随机可用端口
    RANDOM_PORT=$(get_random_available_port $PORT_RANGE_MIN $PORT_RANGE_MAX "$EXCLUDED_PORTS" $MAX_TRIES)
    if [ $? -ne 0 ]; then
        exit 1
    fi
    
    # 将此端口添加到排除列表，避免后续重复使用
    EXCLUDED_PORTS="$EXCLUDED_PORTS $RANDOM_PORT"
    
    # 创建新的端口映射
    if [[ "$FULL_MAPPING" =~ = ]]; then
        NEW_MAPPING="${PARAM_TYPE}=${RANDOM_PORT}:${CONTAINER_PORT}"
    else
        NEW_MAPPING="${PARAM_TYPE} ${RANDOM_PORT}:${CONTAINER_PORT}"
    fi
    
    echo "  原端口映射: $FULL_MAPPING -> 新端口映射: $NEW_MAPPING"
    
    # 替换docker命令中的端口映射
    NEW_DOCKER_CMD="${NEW_DOCKER_CMD//$FULL_MAPPING/$NEW_MAPPING}"
done

# 输出修改后的命令并询问是否执行
echo ""
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"

if [ "$DRY_RUN" = true ]; then
    echo "仅显示模式，不执行命令"
    exit 0
fi

echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
    
    # 检查命令执行状态
    if [ $? -eq 0 ]; then
        echo "命令执行成功!"
    else
        echo "命令执行失败，请检查错误信息"
    fi
else
    echo "已取消执行"
fi
```

## 性能优化说明

1. **随机尝试策略**：不再预先生成所有可用端口列表，而是随机选择端口并检查可用性，直到找到可用端口。

2. **快速端口检查**：使用`ss`命令直接检查特定端口是否被占用，而不是获取所有已占用端口的列表。

3. **最大尝试次数**：添加了最大尝试次数限制（默认50次），以防止在极端情况下陷入无限循环。

4. **动态排除已分配端口**：当为一个映射找到可用端口后，会将该端口添加到排除列表，确保不会在同一次运行中重复分配。

## 使用方法

与之前相同，但增加了一个新选项：

```bash
# 设置最大尝试次数
./random-port.sh --max-tries 100 "docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool"
```

## 安装到系统路径

要让此脚本在任何路径下可用，请使用之前提到的方法之一，例如：

```bash
# 移动到系统PATH目录
sudo mv random-port.sh /usr/local/bin/random-port
sudo chmod +x /usr/local/bin/random-port
```

这个优化版本应该能够显著提高脚本的执行速度，特别是在处理大范围端口时。

-------

Role: Human
如果我现在需要把这个代码发送到github
请编写一个md文件
存储库名称
描述

-------

Role: AI
```thinking
用户希望我为这个随机端口分配工具创建一个GitHub存储库的相关内容，包括README.md文件、存储库名称和描述。我需要为这个项目创建一个专业的GitHub展示。

对于这个工具，我认为合适的存储库名称应该简洁明了，同时表达其功能。可以考虑：
- docker-random-port
- random-port-allocator
- docker-port-randomizer
- port-picker

描述应该简短地概括工具的主要功能和用途。

README.md文件应该包含以下内容：
1. 项目标题和简短描述
2. 安装说明
3. 使用方法和示例
4. 选项说明
5. 工作原理
6. 贡献指南
7. 许可证信息

我会为用户创建一个完整的README.md文件，以及推荐的存储库名称和描述。
```

# GitHub 存储库信息

## 推荐的存储库名称
`docker-port-randomizer`

## 推荐的存储库描述
一个高性能的Docker端口随机分配工具，解决端口冲突和选择困难问题。适用于所有Linux系统，特别优化用于Ubuntu。

## README.md 文件内容

```markdown
# Docker Port Randomizer

![版本](https://img.shields.io/badge/版本-2.0.0-blue)
![平台](https://img.shields.io/badge/平台-Linux-green)
![语言](https://img.shields.io/badge/语言-Bash-yellow)

一个高性能的Docker端口随机分配工具，用于解决Docker容器部署时的端口冲突和选择困难问题。当你需要部署多个服务但不想手动选择端口时，这个工具可以自动为你分配随机可用端口。

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
curl -o random-port.sh https://raw.githubusercontent.com/yourusername/docker-port-randomizer/main/random-port.sh

# 添加执行权限
chmod +x random-port.sh
```

### 方法2：克隆仓库

```bash
# 克隆仓库
git clone https://github.com/yourusername/docker-port-randomizer.git

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

## 许可证

本项目采用 MIT 许可证 - 详情请参阅 [LICENSE](LICENSE) 文件。

## 致谢

- 感谢所有为这个项目提供反馈和建议的用户
- 特别感谢那些在部署多个Docker容器时遇到端口选择困难症的开发者们，这个工具就是为你们创建的！
```

## 脚本文件

将优化后的脚本保存为 `random-port.sh`：

```bash
#!/bin/bash

# 设置语言环境为中文
export LANG=zh_CN.UTF-8

VERSION="2.0.0"

# 显示帮助信息
show_help() {
    echo "随机端口分配工具 v$VERSION"
    echo "用法: $0 [选项] <docker命令>"
    echo ""
    echo "选项:"
    echo "  -h, --help        显示此帮助信息"
    echo "  -v, --version     显示版本信息"
    echo "  -r, --range       指定端口范围 (默认: 1024-65535)"
    echo "  -e, --exclude     指定要排除的端口 (默认: 80,443)"
    echo "  -d, --dry-run     仅显示修改后的命令，不执行"
    echo "  -m, --max-tries   最大尝试次数 (默认: 50)"
    echo ""
    echo "示例:"
    echo "  $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "  $0 --range 8000-9000 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "  $0 --exclude 80,443,3306,5432 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo ""
}

# 显示版本信息
show_version() {
    echo "随机端口分配工具 v$VERSION"
}

# 检查端口是否被占用
is_port_used() {
    local port=$1
    # 使用ss命令检查端口是否被占用
    ss -tuln | grep -q ":$port "
    return $?
}

# 检查端口是否在排除列表中
is_port_excluded() {
    local port=$1
    local excluded_ports=$2
    for excluded in $excluded_ports; do
        if [ "$port" = "$excluded" ]; then
            return 0
        fi
    done
    return 1
}

# 获取随机可用端口
get_random_available_port() {
    local min_port=$1
    local max_port=$2
    local excluded_ports=$3
    local max_tries=$4
    local tries=0
    
    while [ $tries -lt $max_tries ]; do
        # 生成随机端口
        local random_port=$(( $RANDOM % ($max_port - $min_port + 1) + $min_port ))
        
        # 检查端口是否在排除列表中
        if is_port_excluded $random_port "$excluded_ports"; then
            tries=$((tries + 1))
            continue
        fi
        
        # 检查端口是否被占用
        if ! is_port_used $random_port; then
            echo $random_port
            return 0
        fi
        
        tries=$((tries + 1))
    done
    
    echo "错误: 在$max_tries次尝试后未找到可用端口" >&2
    return 1
}

# 默认值
PORT_RANGE_MIN=1024
PORT_RANGE_MAX=65535
EXCLUDED_PORTS="80 443"
DRY_RUN=false
MAX_TRIES=50

# 解析命令行参数
while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)
            show_help
            exit 0
            ;;
        -v|--version)
            show_version
            exit 0
            ;;
        -r|--range)
            if [[ $2 =~ ^([0-9]+)-([0-9]+)$ ]]; then
                PORT_RANGE_MIN="${BASH_REMATCH[1]}"
                PORT_RANGE_MAX="${BASH_REMATCH[2]}"
                shift 2
            else
                echo "错误: 端口范围格式不正确，应为 MIN-MAX"
                exit 1
            fi
            ;;
        -e|--exclude)
            EXCLUDED_PORTS="${2//,/ }"
            shift 2
            ;;
        -d|--dry-run)
            DRY_RUN=true
            shift
            ;;
        -m|--max-tries)
            MAX_TRIES=$2
            shift 2
            ;;
        *)
            # 假设剩余的参数是docker命令
            break
            ;;
    esac
done

# 检查是否提供了docker命令
if [ $# -eq 0 ]; then
    echo "错误: 未提供docker命令"
    echo "请提供一个docker命令作为参数"
    echo "例如: $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "使用 -h 或 --help 查看帮助信息"
    exit 1
fi

# 获取原始docker命令
DOCKER_CMD="$@"

# 分割docker命令为数组
IFS=' ' read -r -a CMD_ARRAY <<< "$DOCKER_CMD"

# 查找所有端口映射
PORT_MAPPINGS=()
for i in "${!CMD_ARRAY[@]}"; do
    # 检查 -p 参数
    if [ "${CMD_ARRAY[$i]}" = "-p" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 -p=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^-p= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
    
    # 检查 --publish 参数
    if [ "${CMD_ARRAY[$i]}" = "--publish" ] && [ $((i+1)) -lt ${#CMD_ARRAY[@]} ]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]} ${CMD_ARRAY[$i+1]}")
    fi
    
    # 检查 --publish=XXX:XXX 格式
    if [[ "${CMD_ARRAY[$i]}" =~ ^--publish= ]]; then
        PORT_MAPPINGS+=("${CMD_ARRAY[$i]}")
    fi
done

if [ ${#PORT_MAPPINGS[@]} -eq 0 ]; then
    echo "错误: 未找到端口映射"
    echo "请确保docker命令包含类似 -p 80:80 或 --publish 80:80 的端口映射"
    exit 1
fi

NEW_DOCKER_CMD="$DOCKER_CMD"

# 处理每个端口映射
echo "端口映射替换计划:"
for mapping in "${PORT_MAPPINGS[@]}"; do
    # 提取端口映射部分
    if [[ "$mapping" =~ (-p|--publish)=([0-9]+):([0-9]+) ]]; then
        # 处理 -p=XXX:XXX 或 --publish=XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE}=${HOST_PORT}:${CONTAINER_PORT}"
    elif [[ "$mapping" =~ (-p|--publish)\ ([0-9]+):([0-9]+) ]]; then
        # 处理 -p XXX:XXX 或 --publish XXX:XXX 格式
        PARAM_TYPE="${BASH_REMATCH[1]}"
        HOST_PORT="${BASH_REMATCH[2]}"
        CONTAINER_PORT="${BASH_REMATCH[3]}"
        FULL_MAPPING="${PARAM_TYPE} ${HOST_PORT}:${CONTAINER_PORT}"
    else
        echo "警告: 无法解析端口映射: $mapping"
        continue
    fi
    
    # 获取随机可用端口
    RANDOM_PORT=$(get_random_available_port $PORT_RANGE_MIN $PORT_RANGE_MAX "$EXCLUDED_PORTS" $MAX_TRIES)
    if [ $? -ne 0 ]; then
        exit 1
    fi
    
    # 将此端口添加到排除列表，避免后续重复使用
    EXCLUDED_PORTS="$EXCLUDED_PORTS $RANDOM_PORT"
    
    # 创建新的端口映射
    if [[ "$FULL_MAPPING" =~ = ]]; then
        NEW_MAPPING="${PARAM_TYPE}=${RANDOM_PORT}:${CONTAINER_PORT}"
    else
        NEW_MAPPING="${PARAM_TYPE} ${RANDOM_PORT}:${CONTAINER_PORT}"
    fi
    
    echo "  原端口映射: $FULL_MAPPING -> 新端口映射: $NEW_MAPPING"
    
    # 替换docker命令中的端口映射
    NEW_DOCKER_CMD="${NEW_DOCKER_CMD//$FULL_MAPPING/$NEW_MAPPING}"
done

# 输出修改后的命令并询问是否执行
echo ""
echo "原始命令: $DOCKER_CMD"
echo "修改后的命令: $NEW_DOCKER_CMD"

if [ "$DRY_RUN" = true ]; then
    echo "仅显示模式，不执行命令"
    exit 0
fi

echo -n "是否采纳此端口映射并执行命令? (y/n): "
read answer

if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    echo "正在执行命令..."
    eval $NEW_DOCKER_CMD
    
    # 检查命令执行状态
    if [ $? -eq 0 ]; then
        echo "命令执行成功!"
    else
        echo "命令执行失败，请检查错误信息"
    fi
else
    echo "已取消执行"
fi
```

## 许可证文件

创建一个 `LICENSE` 文件（MIT许可证）：

```
MIT License

Copyright (c) 2023 Your Name

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

这些文件应该足够让您创建一个专业的GitHub存储库，展示您的Docker端口随机分配工具。

-------

Role: Human
在项目md介绍中融入，告知这个项目完全由你开发，并且你的官网是
https://phapi.furina.junmatec.cn/
我现在已经创建了一个空的储存库 地址如下https://github.com/xwy-morax/docker-port-randomizer 可以替换到你之前的那些示例的地址

-------

Role: AI
```thinking
用户希望我在项目的README.md中融入以下信息：
1. 说明项目是由TIG开发的
2. 添加官网链接：https://phapi.furina.junmatec.cn/
3. 更新GitHub仓库地址为：https://github.com/xwy-morax/docker-port-randomizer

我需要修改之前提供的README.md文件，添加这些信息，并确保它们自然地融入到文档中。我会在适当的位置添加这些信息，比如在项目介绍部分提及开发者，在安装说明部分更新GitHub仓库地址，并在文档底部或顶部添加官网链接。

我还需要确保所有示例中的GitHub链接都更新为用户提供的实际仓库地址。
```

# 修改后的 README.md 文件

```markdown
# Docker Port Randomizer

![版本](https://img.shields.io/badge/版本-2.0.0-blue)
![平台](https://img.shields.io/badge/平台-Linux-green)
![语言](https://img.shields.io/badge/语言-Bash-yellow)

一个高性能的Docker端口随机分配工具，用于解决Docker容器部署时的端口冲突和选择困难问题。当你需要部署多个服务但不想手动选择端口时，这个工具可以自动为你分配随机可用端口。

> 本项目由 TIG 完全开发，官方网站：[https://phapi.furina.junmatec.cn/](https://phapi.furina.junmatec.cn/)

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

## 许可证

本项目采用 MIT 许可证 - 详情请参阅 [LICENSE](LICENSE) 文件。

## 致谢

- 感谢所有为这个项目提供反馈和建议的用户
- 特别感谢那些在部署多个Docker容器时遇到端口选择困难症的开发者们，这个工具就是为你们创建的！

## 关于开发者

本项目由 TIG 完全开发。如需了解更多信息或查看其他项目，请访问我们的官方网站：[https://phapi.furina.junmatec.cn/](https://phapi.furina.junmatec.cn/)
```

# 更新后的安装指令

所有安装和使用示例中的GitHub链接已更新为：`https://github.com/xwy-morax/docker-port-randomizer`

例如：
```bash
# 下载脚本
curl -o random-port.sh https://raw.githubusercontent.com/xwy-morax/docker-port-randomizer/main/random-port.sh

# 克隆仓库
git clone https://github.com/xwy-morax/docker-port-randomizer.git
```

# 添加的开发者信息

在README.md的顶部和底部都添加了开发者信息和官网链接：

1. 在顶部介绍下方：
   > 本项目由 TIG 完全开发，官方网站：[https://phapi.furina.junmatec.cn/](https://phapi.furina.junmatec.cn/)

2. 在底部添加了"关于开发者"部分：
   ```
   ## 关于开发者

   本项目由 TIG 完全开发。如需了解更多信息或查看其他项目，请访问我们的官方网站：[https://phapi.furina.junmatec.cn/](https://phapi.furina.junmatec.cn/)
   ```

这样，无论用户从README的哪个部分开始阅读，都能看到项目的开发者信息和官网链接。
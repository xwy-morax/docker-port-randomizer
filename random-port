#!/bin/bash

# 设置语言环境为中文
export LANG=zh_CN.UTF-8

VERSION="2.0.0"

# 显示帮助信息
show_help() {
    echo "随机端口分配工具 v$VERSION"
    echo "用法: $0 [选项] <docker 命令>"
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
    # 使用 ss 命令检查端口是否被占用
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
    
    echo "错误: 在$max_tries 次尝试后未找到可用端口" >&2
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
            # 假设剩余的参数是 docker 命令
            break
            ;;
    esac
done

# 检查是否提供了 docker 命令
if [ $# -eq 0 ]; then
    echo "错误: 未提供 docker 命令"
    echo "请提供一个 docker 命令作为参数"
    echo "例如: $0 \"docker run -d -p 80:80 --name neko-api-key-tool neko-api-key-tool\""
    echo "使用 -h 或 --help 查看帮助信息"
    exit 1
fi

# 获取原始 docker 命令
DOCKER_CMD="$@"

# 分割 docker 命令为数组
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
    echo "请确保 docker 命令包含类似 -p 80:80 或 --publish 80:80 的端口映射"
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
    
    # 替换 docker 命令中的端口映射
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
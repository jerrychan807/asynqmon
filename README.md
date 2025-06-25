# Asynqmon Nginx 反向代理配置测试

## 功能说明

现在 Asynqmon 支持通过 `--root-path` 参数设置路径前缀，以便与 nginx 反向代理配置兼容。

## 使用方法

### 1. 启动 Asynqmon 服务

```bash
# 使用 /asynqmon 作为路径前缀
./asynqmon --root-path=/asynqmon --port=8081

# 或者使用环境变量
ROOT_PATH=/asynqmon ./asynqmon --port=8081
```

### 2. Nginx 配置

```nginx
location /asynqmon/ {
    proxy_pass http://127.0.0.1:8081/asynqmon/;  # 重要：保持完整路径
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;

    auth_basic "Asynqmon Login";
    auth_basic_user_file /etc/nginx/.asynqmon_pass;
}
```

**注意**：`proxy_pass` 必须包含完整的路径 `/asynqmon/`，否则 nginx 会去掉路径前缀导致后端验证失败。

### 3. 测试验证

```bash
# 测试主页
curl -I http://localhost/asynqmon/

# 测试 API
curl -I http://localhost/asynqmon/api/queues

# 测试静态资源
curl -I http://localhost/asynqmon/static/js/main.*.js
```

## 测试结果

✅ **路径前缀功能正常工作**
- 主页路径：`/asynqmon/` → HTTP 200
- API 路径：`/asynqmon/api/queues` → 正确路由
- 静态资源：`/asynqmon/static/js/main.*.js` → HTTP 200
- 前端配置：`window.FLAG_ROOT_PATH="/asynqmon"` → 正确设置

✅ **向后兼容性**
- 不设置 `--root-path` 时，应用程序仍然正常工作
- 所有路径从根目录开始：`/`, `/api/queues`, `/static/js/main.*.js`

## 构建说明

如果遇到 Node.js 版本兼容性问题，使用以下命令构建前端资源：

```bash
cd ui
export NODE_OPTIONS="--openssl-legacy-provider"
yarn build
```

然后构建完整应用程序：

```bash
# 本地编译
go build -o asynqmon ./cmd/asynqmon
# 交叉编译
GOOS=linux GOARCH=amd64 go build -o asynqmon ./cmd/asynqmon
```

## 问题修复

### 修复了 "unexpected path prefix" 错误

**问题**：在 nginx 代理环境中访问 `/asynqmon/` 时出现 400 错误 "unexpected path prefix"

**原因**：`static.go` 中使用了 `filepath.Abs(r.URL.Path)` 将 URL 路径转换为文件系统绝对路径，导致路径前缀验证失败

**修复**：改为直接使用 `r.URL.Path` 并用 `filepath.Clean()` 清理路径，避免文件系统路径转换

**修改文件**：`static.go` 第30-48行

### nginx 代理路径问题

**问题**：通过 nginx 访问时出现 "unexpected path prefix" 错误，但直接访问后端正常

**原因**：nginx 配置 `proxy_pass http://127.0.0.1:8081/;` 以 `/` 结尾，会去掉 location 匹配的路径前缀

**解决方案**：修改 nginx 配置为 `proxy_pass http://127.0.0.1:8081/asynqmon/;` 保持完整路径

**nginx proxy_pass 规则**：
- `proxy_pass http://backend/;` → 去掉 location 前缀
- `proxy_pass http://backend/path/;` → 用 `/path/` 替换 location 前缀

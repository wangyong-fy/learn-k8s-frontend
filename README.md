# learn-k8s-frontend

K8s 全栈演示 — **前端**（Vue3 + Vite + Nginx）

## 技术栈

- Vue 3 + Vite 5
- 多阶段 Dockerfile：Node 构建 → Nginx 托管（npm 走 npmmirror 加速）

## 功能

- 消息列表展示（调 `/api/messages`）
- 新增消息表单（POST `/api/messages`）
- 顶部显示实际处理请求的 Pod 名与记录数（来自 `/api/health`）

前端**同源调用** `/api/...`，不做跨域处理——由 Ingress 按路径分流：

```
/      → frontend:80
/api   → backend:8080
```

## K8s 清单

`k8s/frontend.yaml` — Deployment(2 副本) + Service(ClusterIP:80)

## 本地开发

```bash
npm install --registry=https://registry.npmmirror.com
npm run dev
```

`vite.config.js` 已配 `/api` 代理到 `http://localhost:8080`。

## 构建

```bash
npm run build      # 产物在 dist/
```

Docker/kaniko 构建见 `Dockerfile`。

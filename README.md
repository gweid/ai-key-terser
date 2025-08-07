# 🔑 API Key 测活工具
> 一个在线工具，批量检测 OpenAI、Claude、Gemini API 密钥有效性

[![GitHub stars](https://img.shields.io/github/stars/weiruchenai1/api-key-tester?style=flat&color=yellow)](https://github.com/weiruchenai1/api-key-tester)
[![在线使用](https://img.shields.io/badge/在线使用-GitHub%20Pages-blue)](https://weiruchenai1.github.io/api-key-tester)

## ✨ 功能

- 🚀 批量测试多个 API 密钥
- 🎯 支持 OpenAI、Claude、Gemini 三大平台
- 🔄 智能重试机制，提高检测准确性
- 🌐 中英文界面切换
- 🌙 浅色/深色主题切换
- 📊 实时统计和错误详情显示
- 📋 一键复制有效/无效/速率限制密钥
- 🎛️ 支持输入自定义模型名称
- ⚡ 可调节并发数和重试次数
- 💻 纯前端，无需安装

## 🚀 使用

1. 选择 API 类型
2. 填写代理服务器 URL（必需）
3. 粘贴 API 密钥列表
4. 使用预设模型或输入自定义模型名
5. 根据需要选择合适的并发数和重试次数
6. 点击开始测试

## ⚠️ 重要提醒

由于浏览器 CORS 限制，必须使用代理服务器：
- **Gemini**: 
`https://gemini.weiruchenai.me/v1beta`
- **OpenAI**: 
`https://openai.weiruchenai.me/v1`
- **Claude**: 
`https://claude.weiruchenai.me/v1`

**公共代理风险提醒：**
- ⚠️ **安全风险**：API密钥可能被代理服务器记录
- 📉 **稳定性差**：可能随时失效或速度很慢
- 🚫 **使用限制**：可能有请求频率或数量限制

**强烈建议使用自己的反向代理以获得更好的安全性、稳定性和成功率。**

<details>
<summary>🛠️ 如何搭建自己的反向代理</summary>

<details>
<summary>📦 Cloudflare Workers 方案</summary>

1. **注册 Cloudflare 账号**：访问 [cloudflare.com](https://cloudflare.com) 注册

2. **创建 Worker**：
   - 进入 Cloudflare Dashboard
   - 点击 `Workers & Pages` > `Create application` > `Create Worker`
   - 给 Worker 起个名字（如 `api-proxy`）

3. **部署代码**：将以下代码粘贴到 Worker 编辑器中

```javascript
// OpenAI 代理
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // 设置目标 API
    const targets = {
      '/openai/': 'https://api.openai.com',
      '/claude/': 'https://api.anthropic.com', 
      '/gemini/': 'https://generativelanguage.googleapis.com'
    };
    
    let targetBase = null;
    let newPath = url.pathname;
    
    for (const [prefix, target] of Object.entries(targets)) {
      if (url.pathname.startsWith(prefix)) {
        targetBase = target;
        newPath = url.pathname.replace(prefix, '/');
        break;
      }
    }
    
    if (!targetBase) {
      return new Response('Not Found', { status: 404 });
    }
    
    const targetUrl = targetBase + newPath + url.search;
    
    const headers = new Headers(request.headers);
    headers.set('Host', new URL(targetBase).host);
    headers.delete('cf-connecting-ip');
    headers.delete('cf-ray');
    
    const response = await fetch(targetUrl, {
      method: request.method,
      headers: headers,
      body: request.body
    });
    
    const newResponse = new Response(response.body, response);
    newResponse.headers.set('Access-Control-Allow-Origin', '*');
    newResponse.headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    newResponse.headers.set('Access-Control-Allow-Headers', '*');
    
    return newResponse;
  },
};
```

4. **保存并部署**：点击 `Save and Deploy`

5. **获取代理地址**：
   - OpenAI: `https://your-worker.workers.dev/openai`
   - Claude: `https://your-worker.workers.dev/claude`
   - Gemini: `https://your-worker.workers.dev/gemini`
</details>

<details>
<summary>🖥️ Nginx 反向代理方案</summary>

如果你有自己的海外服务器，可以使用 Nginx 搭建反向代理：

**前置要求：**
- 一台海外服务器（VPS）
- 一个域名，并将以下子域名解析到服务器IP：
  - `openai.your-domain.com`
  - `claude.your-domain.com` 
  - `gemini.your-domain.com`

**1. 安装 Nginx**
```bash
sudo apt-get update
sudo apt-get install nginx
```

**2. 配置域名和SSL证书**
```bash
# 申请SSL证书（使用 Let's Encrypt）
sudo apt-get install certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

**3. 创建反向代理配置**
```bash
# 为每个API创建单独的配置文件
sudo nano /etc/nginx/sites-available/openai-proxy
```

**4. 添加OpenAI配置内容**
```nginx
# OpenAI 反向代理
server {
    listen 443 ssl;
    server_name openai.your-domain.com;
    
    ssl_certificate /etc/letsencrypt/live/claude.your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/claude.your-domain.com/privkey.pem;
    
    location / {
        # DNS解析器，禁用IPv6
        resolver 8.8.8.8 ipv6=off;
        
        # 反向代理配置
        proxy_pass https://api.openai.com/;
        proxy_ssl_server_name on;
        proxy_set_header Host api.openai.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 隐藏后端服务器的CORS头，避免重复
        proxy_hide_header Access-Control-Allow-Origin;
        proxy_hide_header Access-Control-Allow-Methods;
        proxy_hide_header Access-Control-Allow-Headers;
        proxy_hide_header Access-Control-Allow-Credentials;
        
        # 处理OPTIONS预检请求
        if ($request_method = 'OPTIONS') {
            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods 'GET, POST, PUT, DELETE, OPTIONS';
            add_header Access-Control-Allow-Headers '*';
            add_header Access-Control-Max-Age 86400;
            return 204;
        }
        
        # 为所有其他请求添加CORS头
        add_header Access-Control-Allow-Origin * always;
        add_header Access-Control-Allow-Methods 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header Access-Control-Allow-Headers '*' always;
    }
}
```

**5. 创建Claude配置**
```bash
sudo nano /etc/nginx/sites-available/claude-proxy
```

添加以下内容：
```nginx
# Claude 反向代理
server {
    listen 443 ssl;
    server_name claude.your-domain.com;
    
    ssl_certificate /etc/letsencrypt/live/claude.your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/claude.your-domain.com/privkey.pem;
    
    location / {
        # DNS解析器，禁用IPv6
        resolver 8.8.8.8 ipv6=off;
        
        # 反向代理配置
        proxy_pass https://api.anthropic.com/;
        proxy_ssl_server_name on;
        proxy_set_header Host api.anthropic.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 隐藏后端服务器的CORS头，避免重复
        proxy_hide_header Access-Control-Allow-Origin;
        proxy_hide_header Access-Control-Allow-Methods;
        proxy_hide_header Access-Control-Allow-Headers;
        proxy_hide_header Access-Control-Allow-Credentials;
        
        # 处理OPTIONS预检请求
        if ($request_method = 'OPTIONS') {
            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods 'GET, POST, PUT, DELETE, OPTIONS';
            add_header Access-Control-Allow-Headers '*';
            add_header Access-Control-Max-Age 86400;
            return 204;
        }
        
        # 为所有其他请求添加CORS头
        add_header Access-Control-Allow-Origin * always;
        add_header Access-Control-Allow-Methods 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header Access-Control-Allow-Headers '*' always;
    }
}
```

**6. 创建Gemini配置**
```bash
sudo nano /etc/nginx/sites-available/gemini-proxy
```

添加以下内容：
```nginx
# Gemini 反向代理
server {
    listen 443 ssl;
    server_name gemini.your-domain.com;
    
    ssl_certificate /etc/letsencrypt/live/claude.your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/claude.your-domain.com/privkey.pem;
    
    location / {
        # DNS解析器，禁用IPv6
        resolver 8.8.8.8 ipv6=off;
        
        # 反向代理配置
        proxy_pass https://generativelanguage.googleapis.com/;
        proxy_ssl_server_name on;
        proxy_set_header Host generativelanguage.googleapis.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 隐藏后端服务器的CORS头，避免重复
        proxy_hide_header Access-Control-Allow-Origin;
        proxy_hide_header Access-Control-Allow-Methods;
        proxy_hide_header Access-Control-Allow-Headers;
        proxy_hide_header Access-Control-Allow-Credentials;
        
        # 处理OPTIONS预检请求
        if ($request_method = 'OPTIONS') {
            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods 'GET, POST, PUT, DELETE, OPTIONS';
            add_header Access-Control-Allow-Headers '*';
            add_header Access-Control-Max-Age 86400;
            return 204;
        }
        
        # 为所有其他请求添加CORS头
        add_header Access-Control-Allow-Origin * always;
        add_header Access-Control-Allow-Methods 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header Access-Control-Allow-Headers '*' always;
    }
}
```

**7. 启用配置**
```bash
# 启用所有代理配置
sudo ln -s /etc/nginx/sites-available/openai-proxy /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/claude-proxy /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/gemini-proxy /etc/nginx/sites-enabled/
```

**配置说明：**
- 将 `your-domain.com` 替换为你的实际域名
- 每个API使用单独的配置文件，便于管理
- 证书路径会在申请SSL证书后自动生成，无需手动修改

**8. 重启 Nginx**
```bash
sudo nginx -t  # 检查配置
sudo nginx -s reload  # 重新加载配置
```

**6. 测试反向代理**
```bash
# 测试 OpenAI 代理
curl https://openai.your-domain.com/v1/models

# 测试 Claude 代理  
curl https://claude.your-domain.com/v1/messages

# 测试 Gemini 代理
curl https://gemini.your-domain.com/v1beta/models
```

**代理地址：**
测试成功后，在API Key测试工具中使用以下代理地址：
- OpenAI: `https://openai.your-domain.com`
- Claude: `https://claude.your-domain.com`  
- Gemini: `https://gemini.your-domain.com`
</details>

<details>
<summary>⚡ Vercel 方案</summary>

1. **Fork 项目**：
```bash
git clone https://github.com/你的用户名/api-proxy-vercel
cd api-proxy-vercel
```

2. **创建 api/[...path].js**：
```javascript
export default async function handler(req, res) {
  const { path } = req.query;
  const targetPath = Array.isArray(path) ? path.join('/') : path;
  
  const apiMappings = {
    'openai': 'https://api.openai.com',
    'claude': 'https://api.anthropic.com',
    'gemini': 'https://generativelanguage.googleapis.com'
  };
  
  const apiType = targetPath.split('/')[0];
  const targetBase = apiMappings[apiType];
  
  if (!targetBase) {
    return res.status(404).json({ error: 'API not supported' });
  }
  
  const targetUrl = `${targetBase}/${targetPath.split('/').slice(1).join('/')}`;
  
  try {
    const response = await fetch(targetUrl, {
      method: req.method,
      headers: {
        ...req.headers,
        host: new URL(targetBase).host,
      },
      body: req.method !== 'GET' ? JSON.stringify(req.body) : undefined,
    });
    
    const data = await response.text();
    
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    res.setHeader('Access-Control-Allow-Headers', '*');
    
    res.status(response.status).send(data);
  } catch (error) {
    res.status(500).json({ error: 'Proxy error' });
  }
}
```

3. **部署到 Vercel**：
```bash
npm i -g vercel
vercel --prod
```

### 使用自建代理

将工具中的代理 URL 替换为你的域名：
- **Cloudflare**: `https://your-worker.workers.dev/openai`
- **Nginx**: `https://openai.your-domain.com`  
- **Vercel**: `https://your-app.vercel.app/api/openai`
</details>

### 方案对比

| 方案 | 优势 | 劣势 | 国内访问情况 |
|------|------|------|-------------|
| **Cloudflare Workers** | 免费、简单、全球CDN | 有请求限制 | ⚠️ 访问不稳定 |
| **Nginx + 服务器** | 无限制、可定制、稳定 | 需要服务器、维护成本 | ✅ 完全可控 |
| **Vercel** | 简单部署、免费 | 有冷启动、请求限制 | ❌ 已被墙 |

> **重要提醒**：
> - **Cloudflare Workers** 在国内访问可能不稳定，速度较慢
> - **Vercel** 在国内已被墙，无法直接访问  
> - **Nginx + 海外服务器** 是最稳定可靠的方案

</details>

## 💡 适用场景

- API 密钥批量验证
- 清理过期的密钥
- 快速检测密钥状态
- 团队密钥管理

## 🛡️ 隐私安全

- ✅ 纯前端运行，密钥不会上传
- ✅ 开源透明，代码可审查
- ✅ 仅用于验证，不存储任何数据

## 🛠️ 技术栈

纯前端：HTML + CSS + JavaScript（零依赖）

## 📄 许可证

MIT License

---

# OpenAvatarChat LAM Docker 部署总结

本总结基于实际踩坑过程整理，覆盖 LAM 模式的 Docker 部署、必要模型准备、配置修改、启动顺序，以及常见错误与解决方法。

## 一、正确部署步骤（按顺序执行）

### 1) 拉代码与子模块
```bash
git clone https://github.com/HumanAIGC-Engineering/OpenAvatarChat.git
cd OpenAvatarChat
git submodule update --init --recursive --depth 1
```

### 2) Git LFS（子模块有大文件）
```bash
sudo git lfs install
sudo git submodule foreach --recursive git lfs pull
```

### 3) 下载 LAM 相关模型
```bash
# wav2vec2
git clone --depth 1 https://www.modelscope.cn/AI-ModelScope/wav2vec2-base-960h.git ./models/wav2vec2-base-960h

# LAM 权重（注意：下载的是 .tar.gz 外层包）
wget https://virutalbuy-public.oss-cn-hangzhou.aliyuncs.com/share/aigc3d/data/LAM/LAM_audio2exp_streaming.tar \
  -O ./models/LAM_audio2exp/LAM_audio2exp_streaming.tar.gz

# 解开外层 gzip，得到真正的权重文件
tar -xzf ./models/LAM_audio2exp/LAM_audio2exp_streaming.tar.gz -C ./models/LAM_audio2exp

# 确认权重文件存在（必须保留这个文件）
file ./models/LAM_audio2exp/pretrained_models/lam_audio2exp_streaming.tar
```

**期望输出**：`Zip archive data`（不是 gzip）。这是 torch 可直接加载的权重文件。

### 4) 修改 LAM 配置（LLM + TTS）
编辑 `config/chat_with_lam.yaml`：

```yaml
LLMOpenAICompatible:
  enabled: True
  module: llm/openai_compatible/llm_handler_openai_compatible
  model_name: "deepseek-v3.2"
  api_url: "https://api.chatanywhere.tech/v1"
  api_key: "你的key"  # 只写本地文件，不要提交到仓库

CosyVoice:
  enabled: False

Edge_TTS:
  enabled: True
  module: tts/edgetts/tts_handler_edgetts
  voice: "zh-CN-XiaoxiaoNeural"
```

`asset_path` 默认样例即可：
```
lam_samples/barbara.zip
```

### 5) 构建镜像
```bash
bash build_cuda128.sh
```

如果下载超时，建议：
1) 在 `Dockerfile.cuda12.8` 增加：
```
UV_HTTP_TIMEOUT=1800
UV_HTTP_RETRIES=5
```
2) 使用 host 网络构建：
```bash
docker build --network host -f Dockerfile.cuda12.8 -t open-avatar-chat:latest .
```

### 6) 运行容器
```bash
bash run_docker_cuda128.sh --config config/chat_with_lam.yaml
```

### 7) 访问服务
```text
http://<服务器IP>:8282/ui/index.html
```

### 8) 放行防火墙端口（CentOS/firewalld）
```bash
sudo firewall-cmd --add-port=8282/tcp --permanent
sudo firewall-cmd --reload
```

如果需要 HTTPS：
```bash
bash scripts/create_ssl_certs.sh
```
然后重启容器并访问 `https://<IP>:8282`。

## 二、常见错误与解决方法

### 1) `Unable to find image 'open-avatar-chat:latest' locally`
**原因**：镜像没构建。  
**解决**：
```bash
bash build_cuda128.sh
```

### 2) `Distribution not found ... fastrtc-0.0.28.dev0-py3-none-any.whl`
**原因**：子模块未拉取，缺少 `gradio_webrtc_videochat`。  
**解决**：
```bash
git submodule update --init --recursive --depth 1
sudo git submodule foreach --recursive git lfs pull
```
确认文件存在：
```
ls -lh src/third_party/gradio_webrtc_videochat/dist
```

### 3) `Failed to download ... due to network timeout`
**原因**：网络不稳定，uv 下载超时。  
**解决**：
- 加大 `UV_HTTP_TIMEOUT`
- 使用 `docker build --network host`
- 必要时更换 PyTorch 镜像源（如清华镜像）

### 4) `Because torch was not found in the package registry`
**原因**：无法访问 PyTorch 下载源或索引配置不生效。  
**解决**：
在 `pyproject.toml` 的 `[tool.uv.index]` 改成可用镜像源，例如：
```
url = "https://mirrors.tuna.tsinghua.edu.cn/pytorch/whl/cu128"
```

### 5) `No checkpoint found at ... lam_audio2exp_streaming.tar`
**原因**：权重文件被删了，或路径不对。  
**解决**：确保文件存在：
```
./models/LAM_audio2exp/pretrained_models/lam_audio2exp_streaming.tar
```
不要删除。

### 6) `_pickle.UnpicklingError: invalid load key, '\x1f'`
**原因**：权重仍是 gzip 外层包，torch.load 读到 gzip 头。  
**解决**：
```bash
mv ./models/LAM_audio2exp/pretrained_models/lam_audio2exp_streaming.tar \
   ./models/LAM_audio2exp/LAM_audio2exp_streaming.tar.gz
tar -xzf ./models/LAM_audio2exp/LAM_audio2exp_streaming.tar.gz -C ./models/LAM_audio2exp
file ./models/LAM_audio2exp/pretrained_models/lam_audio2exp_streaming.tar
```

### 7) 浏览器打不开 `https://IP:8282`
**原因**：没证书，服务实际启动为 HTTP。  
**解决**：
- 先用 `http://IP:8282/ui/index.html`
- 若需要麦克风/摄像头，生成证书后改用 https

### 8) 本机能访问但公网不通
**原因**：防火墙/安全组未放行 8282。  
**解决**：
```bash
sudo firewall-cmd --add-port=8282/tcp --permanent
sudo firewall-cmd --reload
```
并在云平台安全组放行 TCP 8282。


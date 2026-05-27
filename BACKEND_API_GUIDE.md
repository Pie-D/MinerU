# Hướng dẫn chạy MinerU Backend API

## Cách 1: Chạy riêng API backend (Khuyến nghị)

### Sử dụng file compose riêng:
```bash
cd /home/tqdiep/MinerU/docker
docker compose -f compose.api-only.yaml up -d
```

### Kiểm tra trạng thái:
```bash
# Xem logs
docker logs -f mineru-api

# Kiểm tra health
curl http://localhost:8000/health

# Xem API docs
# Mở browser: http://localhost:8000/docs
```

### Dừng service:
```bash
docker compose -f compose.api-only.yaml down
```

---

## Cách 2: Sử dụng file compose gốc với profile

### Chạy riêng API:
```bash
cd /home/tqdiep/MinerU/docker
docker compose --profile api up -d
```

### Chạy cả API và Gradio:
```bash
docker compose --profile api --profile gradio up -d
```

### Dừng service:
```bash
# Dừng API
docker compose --profile api down

# Dừng tất cả
docker compose --profile api --profile gradio down
```

---

## Test API nhanh

### 1. Health check:
```bash
curl http://localhost:8000/health
```

### 2. Parse file PDF đơn giản:
```bash
curl -X POST http://localhost:8000/file_parse \
  -F "files=@/path/to/your/document.pdf" \
  -F "backend=pipeline" \
  -F "lang_list=en" \
  -F "return_md=true"
```

### 3. Parse với backend hybrid (độ chính xác cao):
```bash
curl -X POST http://localhost:8000/file_parse \
  -F "files=@/path/to/your/document.pdf" \
  -F "backend=hybrid-auto-engine" \
  -F "lang_list=ch" \
  -F "return_md=true" \
  -F "return_images=true"
```

### 4. Parse bất đồng bộ (cho file lớn):
```bash
# Submit task
TASK_ID=$(curl -X POST http://localhost:8000/tasks \
  -F "files=@/path/to/your/document.pdf" \
  -F "backend=pipeline" \
  -F "lang_list=en" | jq -r '.task_id')

echo "Task ID: $TASK_ID"

# Check status
curl http://localhost:8000/tasks/$TASK_ID

# Get result (khi status = completed)
curl http://localhost:8000/tasks/$TASK_ID/result
```

---

## Tích hợp với service khác

### Python Service
```python
# requirements.txt
# requests==2.31.0

import requests

class MinerUClient:
    def __init__(self, base_url="http://localhost:8000"):
        self.base_url = base_url
    
    def parse_file(self, file_path, backend="pipeline", lang="en"):
        """Parse file đồng bộ"""
        with open(file_path, 'rb') as f:
            files = {'files': f}
            data = {
                'backend': backend,
                'lang_list': [lang],
                'return_md': True
            }
            response = requests.post(
                f'{self.base_url}/file_parse',
                files=files,
                data=data
            )
            response.raise_for_status()
            return response.json()
    
    def parse_file_async(self, file_path, backend="pipeline", lang="en"):
        """Parse file bất đồng bộ"""
        with open(file_path, 'rb') as f:
            files = {'files': f}
            data = {
                'backend': backend,
                'lang_list': [lang],
                'return_md': True
            }
            response = requests.post(
                f'{self.base_url}/tasks',
                files=files,
                data=data
            )
            response.raise_for_status()
            return response.json()['task_id']
    
    def get_task_status(self, task_id):
        """Lấy trạng thái task"""
        response = requests.get(f'{self.base_url}/tasks/{task_id}')
        response.raise_for_status()
        return response.json()
    
    def get_task_result(self, task_id):
        """Lấy kết quả task"""
        response = requests.get(f'{self.base_url}/tasks/{task_id}/result')
        response.raise_for_status()
        return response.json()

# Sử dụng
client = MinerUClient()

# Parse đồng bộ
result = client.parse_file('document.pdf', backend='pipeline', lang='en')
print(result['results'])

# Parse bất đồng bộ
task_id = client.parse_file_async('large_document.pdf')
print(f"Task submitted: {task_id}")

# Check status
import time
while True:
    status = client.get_task_status(task_id)
    if status['status'] in ['completed', 'failed']:
        break
    time.sleep(2)

# Get result
result = client.get_task_result(task_id)
print(result)
```

### Node.js/Express Service
```javascript
// npm install axios form-data

const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');

class MinerUClient {
  constructor(baseUrl = 'http://localhost:8000') {
    this.baseUrl = baseUrl;
  }

  async parseFile(filePath, backend = 'pipeline', lang = 'en') {
    const form = new FormData();
    form.append('files', fs.createReadStream(filePath));
    form.append('backend', backend);
    form.append('lang_list', lang);
    form.append('return_md', 'true');

    const response = await axios.post(`${this.baseUrl}/file_parse`, form, {
      headers: form.getHeaders()
    });

    return response.data;
  }

  async parseFileAsync(filePath, backend = 'pipeline', lang = 'en') {
    const form = new FormData();
    form.append('files', fs.createReadStream(filePath));
    form.append('backend', backend);
    form.append('lang_list', lang);
    form.append('return_md', 'true');

    const response = await axios.post(`${this.baseUrl}/tasks`, form, {
      headers: form.getHeaders()
    });

    return response.data.task_id;
  }

  async getTaskStatus(taskId) {
    const response = await axios.get(`${this.baseUrl}/tasks/${taskId}`);
    return response.data;
  }

  async getTaskResult(taskId) {
    const response = await axios.get(`${this.baseUrl}/tasks/${taskId}/result`);
    return response.data;
  }

  async waitForTask(taskId, pollInterval = 2000) {
    while (true) {
      const status = await this.getTaskStatus(taskId);
      if (status.status === 'completed') {
        return await this.getTaskResult(taskId);
      } else if (status.status === 'failed') {
        throw new Error('Task failed');
      }
      await new Promise(resolve => setTimeout(resolve, pollInterval));
    }
  }
}

// Sử dụng
const client = new MinerUClient();

// Parse đồng bộ
client.parseFile('document.pdf', 'pipeline', 'en')
  .then(result => console.log(result))
  .catch(err => console.error(err));

// Parse bất đồng bộ
client.parseFileAsync('large_document.pdf')
  .then(taskId => client.waitForTask(taskId))
  .then(result => console.log(result))
  .catch(err => console.error(err));
```

### Go Service
```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "mime/multipart"
    "net/http"
    "os"
    "time"
)

type MinerUClient struct {
    BaseURL string
}

func NewMinerUClient(baseURL string) *MinerUClient {
    return &MinerUClient{BaseURL: baseURL}
}

func (c *MinerUClient) ParseFile(filePath, backend, lang string) (map[string]interface{}, error) {
    file, err := os.Open(filePath)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    body := &bytes.Buffer{}
    writer := multipart.NewWriter(body)
    
    part, err := writer.CreateFormFile("files", filePath)
    if err != nil {
        return nil, err
    }
    io.Copy(part, file)
    
    writer.WriteField("backend", backend)
    writer.WriteField("lang_list", lang)
    writer.WriteField("return_md", "true")
    writer.Close()

    req, err := http.NewRequest("POST", c.BaseURL+"/file_parse", body)
    if err != nil {
        return nil, err
    }
    req.Header.Set("Content-Type", writer.FormDataContentType())

    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    return result, nil
}

func (c *MinerUClient) ParseFileAsync(filePath, backend, lang string) (string, error) {
    // Similar to ParseFile but POST to /tasks
    // Return task_id
    return "", nil
}

func (c *MinerUClient) GetTaskStatus(taskID string) (map[string]interface{}, error) {
    resp, err := http.Get(c.BaseURL + "/tasks/" + taskID)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    return result, nil
}

func main() {
    client := NewMinerUClient("http://localhost:8000")
    result, err := client.ParseFile("document.pdf", "pipeline", "en")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println(result)
}
```

---

## Cấu hình nâng cao

### 1. Thay đổi port:
```yaml
# Trong compose.api-only.yaml
ports:
  - 9000:8000  # Expose port 9000 thay vì 8000
```

### 2. Sử dụng nhiều GPU:
```yaml
# Trong compose.api-only.yaml
device_ids: ["0", "1"]  # Sử dụng GPU 0 và 1
```

### 3. Giảm VRAM usage:
```yaml
# Trong compose.api-only.yaml
command:
  --host 0.0.0.0
  --port 8000
  --gpu-memory-utilization 0.4  # Giảm xuống 40%
```

### 4. Enable public HTTP client (cho vlm/hybrid-http-client):
```yaml
# Trong compose.api-only.yaml
command:
  --host 0.0.0.0
  --port 8000
  --allow-public-http-client  # CẢNH BÁO: Có rủi ro SSRF
```

### 5. Disable API docs (production):
```yaml
# Trong compose.api-only.yaml
environment:
  MINERU_MODEL_SOURCE: local
  MINERU_API_ENABLE_FASTAPI_DOCS: 0  # Tắt /docs và /openapi.json
```

---

## Troubleshooting

### 1. Container không start:
```bash
# Xem logs
docker logs mineru-api

# Kiểm tra GPU
nvidia-smi
```

### 2. Out of memory:
```bash
# Giảm gpu-memory-utilization trong command
# Hoặc sử dụng backend pipeline thay vì hybrid/vlm
```

### 3. API trả về 503:
```bash
# Kiểm tra health
curl http://localhost:8000/health

# Restart container
docker restart mineru-api
```

### 4. Task bị stuck:
```bash
# Kiểm tra logs
docker logs -f mineru-api

# Restart nếu cần
docker restart mineru-api
```

---

## Performance Tips

1. **Chọn backend phù hợp:**
   - File đơn giản, nhiều ngôn ngữ: `pipeline`
   - File phức tạp, tiếng Trung/Anh: `vlm-auto-engine`
   - File phức tạp, nhiều ngôn ngữ: `hybrid-auto-engine`

2. **Sử dụng async API cho file lớn:**
   - File > 10 trang: dùng `/tasks` thay vì `/file_parse`

3. **Tối ưu tham số:**
   - Tắt `formula_enable` nếu không cần công thức
   - Tắt `table_enable` nếu không cần bảng
   - Tắt `return_images` nếu không cần ảnh

4. **Scale horizontal:**
   - Chạy nhiều container API với port khác nhau
   - Dùng load balancer (nginx, traefik) phía trước

---

## Monitoring

### Prometheus metrics (nếu có):
```yaml
# Thêm vào compose
ports:
  - 8000:8000
  - 9090:9090  # Metrics port
```

### Health check script:
```bash
#!/bin/bash
# health_check.sh

while true; do
  STATUS=$(curl -s http://localhost:8000/health | jq -r '.status')
  if [ "$STATUS" != "healthy" ]; then
    echo "$(date): API is unhealthy!"
    # Send alert
  fi
  sleep 60
done
```

---

## Xem thêm

- [API Documentation](./API_DOCUMENTATION.md) - Chi tiết về các endpoints
- [MinerU README](./README.md) - Tài liệu chính thức
- [Docker Compose](./docker/compose.yaml) - File compose gốc

# MinerU Backend API Documentation

## Base URL
```
http://localhost:8000
```

## API Endpoints

### 1. Health Check
**Endpoint:** `GET /health`

**Mô tả:** Kiểm tra trạng thái sức khỏe của service

**Response:**
```json
{
  "status": "healthy",
  "version": "x.x.x"
}
```

---

### 2. Parse File (Đồng bộ)
**Endpoint:** `POST /file_parse`

**Mô tả:** Upload và parse file ngay lập tức, đợi kết quả trả về trong cùng 1 request

**Content-Type:** `multipart/form-data`

**Parameters:**

| Tham số | Type | Mặc định | Mô tả |
|---------|------|----------|-------|
| `files` | File[] | **Required** | File PDF, ảnh, DOCX, PPTX, hoặc XLSX cần parse |
| `lang_list` | string[] | `["ch"]` | Danh sách ngôn ngữ (ch, en, korean, japan, arabic, v.v.) |
| `backend` | string | `"hybrid-auto-engine"` | Backend engine: `pipeline`, `vlm-auto-engine`, `vlm-http-client`, `hybrid-auto-engine`, `hybrid-http-client` |
| `parse_method` | string | `"auto"` | Phương pháp parse: `auto`, `txt`, `ocr` |
| `formula_enable` | boolean | `true` | Bật parse công thức toán học |
| `table_enable` | boolean | `true` | Bật parse bảng |
| `image_analysis` | boolean | `true` | Bật phân tích ảnh/biểu đồ (cho VLM/hybrid backend) |
| `server_url` | string | `null` | URL server OpenAI-compatible (cho http-client backend) |
| `return_md` | boolean | `true` | Trả về nội dung markdown |
| `return_middle_json` | boolean | `false` | Trả về middle JSON |
| `return_model_output` | boolean | `false` | Trả về model output JSON |
| `return_content_list` | boolean | `false` | Trả về content list JSON |
| `return_images` | boolean | `false` | Trả về ảnh đã extract |
| `response_format_zip` | boolean | `false` | Trả về kết quả dạng ZIP thay vì JSON |
| `return_original_file` | boolean | `false` | Bao gồm file gốc trong ZIP (chỉ khi `response_format_zip=true`) |
| `start_page_id` | integer | `0` | Trang bắt đầu parse (từ 0) |
| `end_page_id` | integer | `99999` | Trang kết thúc parse (từ 0) |

**Response (JSON):**
```json
{
  "task_id": "uuid-string",
  "status": "completed",
  "backend": "hybrid-auto-engine",
  "version": "x.x.x",
  "results": {
    "document_name.pdf": {
      "md_content": "# Markdown content...",
      "content_list": [...],
      "images": [...]
    }
  }
}
```

**Response (ZIP):** File ZIP chứa markdown, JSON, và ảnh

---

### 3. Submit Async Task (Bất đồng bộ)
**Endpoint:** `POST /tasks`

**Mô tả:** Submit task parse và nhận task_id ngay lập tức, không đợi kết quả

**Content-Type:** `multipart/form-data`

**Parameters:** Giống như `/file_parse`

**Response:**
```json
{
  "task_id": "uuid-string",
  "status": "pending",
  "message": "Task submitted successfully",
  "status_url": "http://localhost:8000/tasks/{task_id}",
  "result_url": "http://localhost:8000/tasks/{task_id}/result",
  "created_at": "2026-05-25T10:00:00Z"
}
```

---

### 4. Get Task Status
**Endpoint:** `GET /tasks/{task_id}`

**Mô tả:** Kiểm tra trạng thái của task

**Response:**
```json
{
  "task_id": "uuid-string",
  "status": "processing",
  "created_at": "2026-05-25T10:00:00Z",
  "updated_at": "2026-05-25T10:00:05Z",
  "status_url": "http://localhost:8000/tasks/{task_id}",
  "result_url": "http://localhost:8000/tasks/{task_id}/result"
}
```

**Status values:**
- `pending`: Task đang chờ xử lý
- `processing`: Task đang được xử lý
- `completed`: Task hoàn thành
- `failed`: Task thất bại

---

### 5. Get Task Result
**Endpoint:** `GET /tasks/{task_id}/result`

**Mô tả:** Lấy kết quả của task đã hoàn thành

**Response (nếu task chưa xong):**
```json
{
  "task_id": "uuid-string",
  "status": "processing",
  "message": "Task result is not ready yet"
}
```

**Response (nếu task hoàn thành):** Giống như response của `/file_parse`

---

## Backend Options

### 1. `pipeline` (Đa ngôn ngữ, không hallucination)
- Hỗ trợ nhiều ngôn ngữ
- Không cần GPU mạnh
- Độ chính xác trung bình

### 2. `vlm-auto-engine` (Độ chính xác cao, local)
- Chỉ hỗ trợ tiếng Trung và tiếng Anh
- Cần GPU mạnh
- Độ chính xác cao

### 3. `vlm-http-client` (Độ chính xác cao, remote)
- Chỉ hỗ trợ tiếng Trung và tiếng Anh
- Gọi đến OpenAI-compatible server
- Cần set `server_url`

### 4. `hybrid-auto-engine` (Thế hệ mới, local)
- Hỗ trợ nhiều ngôn ngữ
- Cần GPU mạnh
- Độ chính xác cao nhất

### 5. `hybrid-http-client` (Thế hệ mới, remote)
- Hỗ trợ nhiều ngôn ngữ
- Gọi đến OpenAI-compatible server
- Cần set `server_url`

---

## Ví dụ sử dụng

### Python
```python
import requests

# 1. Parse đồng bộ
with open('document.pdf', 'rb') as f:
    files = {'files': f}
    data = {
        'backend': 'hybrid-auto-engine',
        'lang_list': ['ch'],
        'return_md': True,
        'return_images': False
    }
    response = requests.post('http://localhost:8000/file_parse', files=files, data=data)
    result = response.json()
    print(result['results'])

# 2. Parse bất đồng bộ
with open('document.pdf', 'rb') as f:
    files = {'files': f}
    data = {'backend': 'pipeline', 'lang_list': ['en']}
    
    # Submit task
    response = requests.post('http://localhost:8000/tasks', files=files, data=data)
    task_id = response.json()['task_id']
    
    # Check status
    import time
    while True:
        status_response = requests.get(f'http://localhost:8000/tasks/{task_id}')
        status = status_response.json()['status']
        if status in ['completed', 'failed']:
            break
        time.sleep(2)
    
    # Get result
    result_response = requests.get(f'http://localhost:8000/tasks/{task_id}/result')
    result = result_response.json()
    print(result)
```

### cURL
```bash
# Health check
curl http://localhost:8000/health

# Parse file đồng bộ
curl -X POST http://localhost:8000/file_parse \
  -F "files=@document.pdf" \
  -F "backend=pipeline" \
  -F "lang_list=en" \
  -F "return_md=true"

# Submit async task
curl -X POST http://localhost:8000/tasks \
  -F "files=@document.pdf" \
  -F "backend=hybrid-auto-engine" \
  -F "lang_list=ch"

# Check task status
curl http://localhost:8000/tasks/{task_id}

# Get task result
curl http://localhost:8000/tasks/{task_id}/result
```

### Node.js
```javascript
const FormData = require('form-data');
const fs = require('fs');
const axios = require('axios');

// Parse đồng bộ
async function parseFile() {
  const form = new FormData();
  form.append('files', fs.createReadStream('document.pdf'));
  form.append('backend', 'pipeline');
  form.append('lang_list', 'en');
  form.append('return_md', 'true');
  
  const response = await axios.post('http://localhost:8000/file_parse', form, {
    headers: form.getHeaders()
  });
  
  console.log(response.data);
}

// Parse bất đồng bộ
async function parseFileAsync() {
  const form = new FormData();
  form.append('files', fs.createReadStream('document.pdf'));
  form.append('backend', 'hybrid-auto-engine');
  
  // Submit task
  const submitResponse = await axios.post('http://localhost:8000/tasks', form, {
    headers: form.getHeaders()
  });
  
  const taskId = submitResponse.data.task_id;
  
  // Poll for result
  while (true) {
    const statusResponse = await axios.get(`http://localhost:8000/tasks/${taskId}`);
    const status = statusResponse.data.status;
    
    if (status === 'completed') {
      const resultResponse = await axios.get(`http://localhost:8000/tasks/${taskId}/result`);
      console.log(resultResponse.data);
      break;
    } else if (status === 'failed') {
      console.error('Task failed');
      break;
    }
    
    await new Promise(resolve => setTimeout(resolve, 2000));
  }
}
```

---

## Swagger UI Documentation

Khi service đang chạy, bạn có thể truy cập Swagger UI để test API trực tiếp:

```
http://localhost:8000/docs
```

## OpenAPI Schema

```
http://localhost:8000/openapi.json
```

---

## Notes

1. **File hỗ trợ:** PDF, ảnh (PNG, JPG), DOCX, PPTX, XLSX
2. **Giới hạn file:** Tùy thuộc vào cấu hình server
3. **Timeout:** Task lớn có thể mất vài phút, nên dùng async API
4. **GPU:** Backend `auto-engine` cần GPU, `http-client` không cần
5. **Ngôn ngữ:** `lang_list` chỉ áp dụng cho `pipeline` và `hybrid` backend

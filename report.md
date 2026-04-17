
# Part 1
## Ex1.1
### 1. API key hardcode (Dòng 17, 18)
```python
OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"
DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"
```
- **Vấn đề:** Lưu thẳng thông tin nhạy cảm (API Key, Mật khẩu Database) vào mã nguồn. Nếu vô tình push đoạn code này lên GitHub, tài khoản của bạn sẽ bị rò rỉ và kẻ gian có thể đánh cắp tiền/dữ liệu trong tích tắc.
- **Cách khắc phục chuẩn:** Phải đọc thông tin này từ Biến môi trường (Environment Variables) thông qua `os.getenv("OPENAI_API_KEY")` hoặc dùng `.env`.

### 2. Port cố định và host chỉ định (Dòng 51, 52)
```python
host="localhost",   # ❌ chỉ chạy được trên local
port=8000,          # ❌ cứng port
```
- **Vấn đề:** 
  - `host="localhost"` (hay `127.0.0.1`) khiến server chỉ chấp nhận các request gửi từ chính trong máy tính đó. Khi deploy lên Cloud hoặc Docker, máy chủ từ bên ngoài truy cập vào sẽ bị chặn lại.
  - `port=8000` được code cứng. Trên các môi trường Cloud (như Railway, Render, Heroku), nền tảng sẽ tự động cấp một Port thông qua biến môi trường. Nếu bạn ép chạy port 8000, nền tảng sẽ đánh giá app không khởi động thành công.
- **Cách khắc phục chuẩn:** Đổi thành `host="0.0.0.0"` và `port=int(os.environ.get("PORT", 8000))`.

### 3. Debug mode và Reload (Dòng 21, 53)
```python
DEBUG = True
# ...
reload=True         # ❌ debug reload trong production
```
- **Vấn đề:** 
  - `reload=True` dùng để server tự khởi động lại mỗi khi bạn Ctrl+S lưu code. Trên production, tính năng này ngốn rất nhiều CPU/RAM để liên tục theo dõi hệ thống file.
  - `DEBUG = True` có thể khiến server trả về toàn bộ chi tiết lỗi (Stack trace) ra trình duyệt của người dùng khi app bị crash, làm lộ cấu trúc mã nguồn bên trong.
- **Cách khắc phục chuẩn:** Ở production, tắt chế độ reload. Tốt nhất là chạy ứng dụng thông qua CLI bằng lệnh thay vì viết script trong python: `uvicorn app:app --host 0.0.0.0 --port $PORT`.

### 4. Không có health check (Dòng 42)
- **Vấn đề:** Hiện tại ứng dụng chỉ có `/` và `/ask`. Nền tảng Cloud hoặc Load Balancer cần biết "App của bạn đã khởi động xong và sẵn sàng nhận traffic chưa?" hoặc "App có đang bị treo không?". Do không có endpoint nào phục vụ việc này, Cloud provider có thể tưởng app đã chết và liên tục khởi động lại nó, hoặc tệ hơn là vẫn gửi request tới dù app đã treo.
- **Cách khắc phục chuẩn:** Viết thêm một route đơn giản:
  ```python
  @app.get("/health")
  def health():
      return {"status": "ok"}
  ```

### 5. Không xử lý shutdown (Graceful Shutdown)
- **Vấn đề:** Không có logic để xử lý sự kiện khi server bị tắt. Khi bạn deploy bản cập nhật mới, Cloud Provider sẽ tắt app cũ đi bằng cách gửi tín hiệu `SIGTERM`. Nếu app không biết xử lý:
  - Các request của user đang được AI trả lời dở dang sẽ bị ngắt đột ngột (trả về lỗi 502/503).
  - Kết nối tới Database (nếu có) bị cắt đứt mà không được đóng đàng hoàng, có nguy cơ gây kẹt connection.
- **Cách khắc phục chuẩn:** Cần bắt tín hiệu tắt máy hoặc sử dụng `lifespan` event của FastAPI để đợi các request đang chạy xong (drain), đóng kết nối an toàn rồi mới cho ứng dụng thoát.

## Ex1.2

(lab12) D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment\01-localhost-vs-production\develop>python app.py         
Starting agent on localhost:8000...
INFO:     Will watch for changes in these directories: ['D:\\vinlab\\lab12\\day12_ha-tang-cloud_va_deployment\\01-localhost-vs-production\\develop']
INFO:     Uvicorn running on http://localhost:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [11932] using WatchFiles
INFO:     Started server process [11528]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
[DEBUG] Got question: Hello
[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this
[DEBUG] Response: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.
INFO:     127.0.0.1:52820 - "POST /ask?question=Hello HTTP/1.1" 200 OK

D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment\01-localhost-vs-production\develop>curl -X POST "http://localhost:8000/ask?question=Hello"
{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}

## Ex1.3

(lab12) D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment\01-localhost-vs-production\production>python app.py
WARNING:root:OPENAI_API_KEY not set — using mock LLM
INFO:     Started server process [19564]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     127.0.0.1:49395 - "POST /ask HTTP/1.1" 200 OK

D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment\01-localhost-vs-production\develop>curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d "{\"question\": \"Hello\"}"
{"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","model":"gpt-4o-mini"}

| Feature | Basic (`develop/app.py`) | Advanced (`production/app.py`) | Tại sao quan trọng? (Lợi ích khi lên Cloud) |
| :--- | :--- | :--- | :--- |
| **Config** | Hardcode thẳng vào code | Dùng Environment variables (`.env`) | **Bảo mật và linh hoạt:** Tránh lộ mật khẩu/API key lên GitHub. Dễ dàng đổi cấu hình giữa các môi trường (Dev, Staging, Prod) mà không phải sửa lại code. |
| **Health check** | Không có | Có endpoint `/health` | **Đảm bảo tính sẵn sàng (Availability):** Giúp Cloud/Load Balancer biết ứng dụng đã chạy xong để phân bổ traffic tới, hoặc tự động restart container nếu ứng dụng bị treo (deadlock). |
| **Logging** | Dùng `print()` | Dùng thư viện Structured Logging (JSON) | **Dễ dàng theo dõi và gỡ lỗi (Observability):** Chuỗi JSON giúp các hệ thống gom log (ElasticSearch, CloudWatch, Datadog) dễ dàng phân tích, lọc và tìm kiếm theo cấu trúc. |
| **Shutdown** | Đột ngột | Graceful Shutdown (Bắt `SIGTERM`) | **Bảo vệ dữ liệu và trải nghiệm người dùng:** Cho phép server xử lý nốt các request đang dang dở và đóng kết nối DB an toàn. Ngăn lỗi 502/503 khi update server. |


# Part 2

## Ex2.1


### 1. Base image là gì?
- **Base image đang dùng:** `python:3.11` (như ở dòng 8: `FROM python:3.11`).
- **Ý nghĩa:** Đây là hình ảnh nền tảng ban đầu để xây dựng Container. `python:3.11` là một image được dựng sẵn, bên trong đã cài đặt sẵn hệ điều hành Linux (thường là Debian) và ngôn ngữ Python bản 3.11. Từ nền tảng này, ta mới thêm code và thư viện của riêng mình vào.

### 2. Working directory là gì?
- **Working directory đang dùng:** `/app` (như ở dòng 11: `WORKDIR /app`).
- **Ý nghĩa:** Đây là thư mục làm việc mặc định **bên trong Container**. Mọi lệnh chạy phía sau (như `RUN`, `COPY`, `CMD`) nếu dùng đường dẫn tương đối (như dấu `.`) thì Docker sẽ ngầm hiểu là đang thao tác bên trong thư mục `/app` của Container.

### 3. Tại sao lại COPY requirements.txt trước khi COPY code?
Đây là một Best Practice cực kỳ quan trọng gọi là **Docker Layer Caching**.
- Mỗi lệnh trong Dockerfile tạo ra một "Layer" (lớp). Docker sẽ lưu lại (cache) các lớp này để tăng tốc cho lần build sau. 
- Khi bạn sửa lại code (`app.py`), layer chứa code bị thay đổi, dẫn đến tất cả các layer nằm sau nó đều phải chạy lại.
- **Nếu copy toàn bộ code cùng lúc với file thư viện:** Mỗi lần bạn sửa dù chỉ một chữ trong `app.py`, Docker cũng sẽ phải chạy lại lệnh `pip install` để cài lại toàn bộ hàng chục thư viện từ đầu (rất tốn thời gian).
- **Khi copy `requirements.txt` lên trước và chạy `pip install`:** Nếu bạn chỉ sửa code mà không cài thêm thư viện mới, file `requirements.txt` không đổi. Docker sẽ dùng lại bộ nhớ đệm (cache) của bước cài thư viện trước đó, và chỉ copy lại file code mới. Điều này giúp thời gian build lại hình ảnh giảm từ vài phút xuống chỉ còn vài giây!

### 4. CMD và ENTRYPOINT khác nhau thế nào?
Mặc dù trong Dockerfile này chỉ dùng `CMD ["python", "app.py"]`, nhưng sự khác biệt giữa hai lệnh này là câu hỏi kinh điển:
- **`CMD` (Command):** Cung cấp lệnh mặc định để chạy khi khởi động container. 
  - *Điểm đặc biệt:* Rất dễ bị ghi đè (override) từ bên ngoài. 
  - *Ví dụ:* Nếu bạn chạy `docker run my-agent ls -la`, thì lệnh `ls -la` sẽ lập tức thay thế hoàn toàn `["python", "app.py"]`.
- **`ENTRYPOINT`:** Định nghĩa lệnh "cốt lõi" luôn luôn phải chạy, không cho phép thay thế một cách dễ dàng.
  - *Điểm đặc biệt:* Nếu bạn truyền thêm chữ phía sau lệnh `docker run`, chữ đó sẽ được **nối thêm (append)** vào sau lệnh ENTRYPOINT như một tham số, chứ không ghi đè nó. 
  - *Khi nào dùng:* Dùng khi container của bạn được thiết kế giống như một phần mềm thực thi (executable) duy nhất và không muốn người dùng vô tình chạy lệnh khác.


## Ex2.2

(lab12) D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment>docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
[+] Building 2.1s (12/12) FINISHED docker:desktop-
 => [internal] load build definition from D  0.0s
 => => transferring dockerfile: 1.36kB       0.0s 
 => [internal] load metadata for docker.io/  1.7s 
 => [internal] load .dockerignore            0.0s
 => => transferring context: 2B              0.0s 
 => [1/7] FROM docker.io/library/python:3.1  0.0s 
 => => resolve docker.io/library/python:3.1  0.0s 
 => [internal] load build context            0.0s 
 => => transferring context: 227B            0.0s 
 => CACHED [2/7] WORKDIR /app                0.0s 
 => CACHED [3/7] COPY 02-docker/develop/req  0.0s
 => CACHED [4/7] RUN pip install --no-cache  0.0s 
 => CACHED [5/7] COPY 02-docker/develop/app  0.0s 
 => CACHED [6/7] RUN mkdir -p utils          0.0s 
 => CACHED [7/7] COPY utils/mock_llm.py uti  0.0s 
 => exporting to image                       0.1s 
 => => exporting layers                      0.0s 
 => => exporting manifest sha256:a90b47b8af  0.0s 
 => => exporting config sha256:dcce18594d16  0.0s 
 => => exporting attestation manifest sha25  0.0s 
 => => exporting manifest list sha256:d2682  0.0s 
 => => naming to docker.io/library/my-agent  0.0s 
 => => unpacking to docker.io/library/my-ag  0.0s 

(lab12) D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment>docker run -p 8000:8000 my-agent:develop
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     172.17.0.1:56218 - "POST /ask HTTP/1.1" 422 Unprocessable Entity
INFO:     172.17.0.1:55206 - "POST /ask HTTP/1.1" 422 Unprocessable Entity
INFO:     172.17.0.1:53718 - "POST /ask HTTP/1.1" 422 Unprocessable Entity
INFO:     172.17.0.1:39686 - "POST /ask?question=What-is-Docker? HTTP/1.1" 200 OK

D:\vinlab\lab12\day12_ha-tang-cloud_va_deploymentcurl -X POST "http://localhost:8000/ask?question=What-is-Docker?"
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}

D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment>docker images my-agent:develop
IMAGE              ID             DISK USAGE
my-agent:develop   d26825b81b37       1.66GB


## Ex2.3
(lab12) D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment>docker build -f 02-docker/production/Dockerfile -t my-agent:advanced .
[+] Building 225.8s (17/17) FINISHED docker:desktop-linu
 => [internal] load build definition from Dockerf  0.0s 
 => => transferring dockerfile: 3.32kB             0.0s 
 => [internal] load metadata for docker.io/librar  0.9s 
 => [internal] load .dockerignore                  0.0s 
 => => transferring context: 2B                    0.0s 
 => [internal] load build context                  0.0s 
 => => transferring context: 292B                  0.0s 
 => [builder 1/5] FROM docker.io/library/python:3  5.9s 
 => => resolve docker.io/library/python:3.11-slim  0.0s 
 => => sha256:d98f36ba21593a401 14.37MB / 14.37MB  5.1s 
 => => sha256:5435b2dcdf5cb7faa 29.78MB / 29.78MB  2.9s
 => => sha256:0816ee351b60c03c7a77a3f 251B / 251B  1.3s 
 => => sha256:37ffa6577b04e98598e 1.29MB / 1.29MB  3.1s 
 => => extracting sha256:5435b2dcdf5cb7faa0d5b1d4  1.2s 
 => => extracting sha256:37ffa6577b04e98598e2c104  0.1s 
 => => extracting sha256:d98f36ba21593a4014243f41  0.7s 
 => => extracting sha256:0816ee351b60c03c7a77a3f5  0.0s 
 => [builder 2/5] WORKDIR /app                     0.3s 
 => [runtime 2/8] RUN groupadd -r appuser && user  1.1s 
 => [builder 3/5] RUN apt-get update && apt-get  149.6s 
 => [runtime 3/8] WORKDIR /app                     0.1s 
 => [builder 4/5] COPY 02-docker/production/requi  0.1s 
 => [builder 5/5] RUN pip install --no-cache-dir  65.6s 
 => [runtime 4/8] COPY --from=builder /root/.loca  0.3s 
 => [runtime 5/8] COPY 02-docker/production/main.  0.1s 
 => [runtime 6/8] RUN mkdir -p /app/utils          0.3s 
 => [runtime 7/8] COPY utils/mock_llm.py /app/uti  0.0s 
 => [runtime 8/8] RUN chown -R appuser:appuser /a  0.3s 
 => exporting to image                             2.0s 
 => => exporting layers                            1.4s 
 => => exporting manifest sha256:73421a2ab30a98b7  0.0s 
 => => exporting config sha256:197328601682fc99cf  0.0s 
 => => exporting attestation manifest sha256:8859  0.0s 
 => => exporting manifest list sha256:478fee51a0c  0.0s 
 => => naming to docker.io/library/my-agent:advan => => exporting manifest list sha256:478fee51a0c  0.0s
 => => naming to docker.io/library/my-agent:advan  0.0s
 => => unpacking to docker.io/library/my-agent:ad  0.5s

 (lab12) D:\vinlab\lab12\day12_ha-tang-cloud_va_deal command,
ployment>docker images | findstr my-agent
WARNING: This output is designed for human readability. For machine-readable output, please use --format.
my-agent:advanced        478fee51a0c4        236MB         56.6MB
my-agent:develop         d26825b81b37       1.66GB          424MB   U
my-agent:latest          f75bfbf8453c       1.66GB          424MB   U



### 1. Stage 1 (Builder stage) làm gì?
*   **Mục tiêu:** Tạo ra một môi trường đầy đủ để "xây dựng" (build) các thư viện cần thiết.
*   **Các bước chính:**
    *   Sử dụng image `python:3.11-slim`.
    *   Cài đặt các công cụ biên dịch như `gcc`, `libpq-dev` (các công cụ này thường rất nặng).
    *   Cài đặt các thư viện từ `requirements.txt` vào một thư mục riêng (`/root/.local`).
*   **Kết quả:** Tạo ra các bộ thư viện đã sẵn sàng để chạy, nhưng cũng kéo theo rất nhiều "rác" từ quá trình build.

### 2. Stage 2 (Runtime stage) làm gì?
*   **Mục tiêu:** Tạo ra một môi trường cực kỳ tinh gọn và bảo mật để "chạy" ứng dụng.
*   **Các bước chính:**
    *   Khởi tạo từ một image `python:3.11-slim` mới hoàn toàn (sạch sẽ).
    *   Tạo user mới (`appuser`) để chạy app thay vì dùng quyền root (tăng tính bảo mật).
    *   **Chỉ COPY kết quả** (các thư viện đã build xong) từ Stage 1 sang Stage 2.
    *   Copy mã nguồn ứng dụng và thiết lập lệnh khởi chạy.
*   **Kết quả:** Một container chỉ chứa những gì tối thiểu nhất để app có thể chạy được.

### 3. Tại sao image nhỏ hơn? (Giảm từ 1.66GB xuống còn ~236MB)
Có 3 lý do chính khiến dung lượng giảm mạnh:
1.  **Loại bỏ "Build tools":** Toàn bộ các công cụ nặng như `gcc`, `apt cache`, và các file tạm phát sinh khi cài thư viện ở Stage 1 đều bị bỏ lại. Stage 2 không hề chứa chúng.
2.  **Sử dụng Base Image "Slim":** Thay vì dùng bản `python:3.11` đầy đủ (nặng gần 1GB), Dockerfile này dùng bản `python:3.11-slim` (chỉ hơn 100MB).
3.  **Kỹ thuật Multi-stage:** Đây là lý do cốt lõi. Nó cho phép ta "vắt chanh bỏ vỏ" — chỉ giữ lại những file thực thi cuối cùng và vứt bỏ toàn bộ "giàn giáo" đã dùng để xây dựng nên chúng.

Điều này không chỉ giúp tiết kiệm ổ cứng mà còn giúp việc đẩy image lên Cloud nhanh hơn và ứng dụng an toàn hơn (vì hacker có ít công cụ để phá hoại bên trong container hơn).

## Ex2.4
(lab12) D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment>docker compose -f 02-docker/production/docker-compose.yml up --build
time="2026-04-17T16:29:26+07:00" level=warning msg="D:\\vinlab\\lab12\\day12_ha-tang-cloud_va_deployment\\02-docker\\production\\docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion"
[+] Building 2.3s (19/19) FINISHED
 => [internal] load local bake definitions  0.0s 
 => => reading from stdin 610B              0.0s
 => [internal] load build definition from   0.1s
 => => transferring dockerfile: 3.32kB      0.0s 
 => [internal] load metadata for docker.io  1.0s 
 => [internal] load .dockerignore           0.0s
 => => transferring context: 2B             0.0s 
 => [internal] load build context           0.0s 
 => => transferring context: 237B           0.0s 
 => [builder 1/5] FROM docker.io/library/p  0.0s 
 => => resolve docker.io/library/python:3.  0.0s 
 => CACHED [runtime 2/8] RUN groupadd -r a  0.0s
 => CACHED [runtime 3/8] WORKDIR /app       0.0s 
 => CACHED [builder 2/5] WORKDIR /app       0.0s 
 => CACHED [builder 3/5] RUN apt-get updat  0.0s 
 => CACHED [builder 4/5] COPY 02-docker/pr  0.0s 
 => CACHED [builder 5/5] RUN pip install -  0.0s 
 => CACHED [runtime 4/8] COPY --from=build  0.0s 
 => CACHED [runtime 5/8] COPY 02-docker/pr  0.0s 
 => CACHED [runtime 6/8] RUN mkdir -p /app  0.0s
 => CACHED [runtime 7/8] COPY utils/mock_l  0.0s 
 => CACHED [runtime 8/8] RUN chown -R appu  0.0s 
 => exporting to image                      0.1s 
 => => exporting layers                     0.0s 
 => => exporting manifest sha256:14d6746fb  0.0s 
 => => exporting config sha256:55c03565c18  0.0s 
 => => exporting attestation manifest sha2  0.0s 
 => => exporting manifest list sha256:bf88  0.0s 
 => => naming to docker.io/library/product  0.0s 
 => => unpacking to docker.io/library/prod  0.0s 
 => resolving provenance for metadata file  0.0s 
[+] Running 8/8
 ✔ production-agent               Built     0.0s 
 ✔ Network production_internal    Created   0.1s 
 ✔ Volume production_redis_data   Created   0.0s 
 ✔ Volume production_qdrant_data  Created   0.0s 
 ✔ Container production-redis-1   Created   0.2s 
 ✔ Container production-qdrant-1  Created   0.2s 
 ✔ Container production-agent-1   Created   0.1s 
 ✔ Container production-nginx-1   Created   0.1s 
Attaching to agent-1, nginx-1, qdrant-1, redis-1
redis-1  | 1:C 17 Apr 2026 09:29:30.632 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis-1  | 1:C 17 Apr 2026 09:29:30.632 * Redis version=7.4.8, bits=64, commit=00000000, modified=0, pid=1, just started                           
redis-1  | 1:C 17 Apr 2026 09:29:30.632 * Configuration loaded                                    
redis-1  | 1:M 17 Apr 2026 09:29:30.633 * monotonic clock: POSIX clock_gettime
redis-1  | 1:M 17 Apr 2026 09:29:30.635 * Running mode=standalone, port=6379.                     
redis-1  | 1:M 17 Apr 2026 09:29:30.636 * Server initialized                                      
redis-1  | 1:M 17 Apr 2026 09:29:30.637 * Ready to accept connections tcp                         
qdrant-1  |            _                 _       
qdrant-1  |   __ _  __| |_ __ __ _ _ __ | |_  
qdrant-1  |  / _` |/ _` | '__/ _` | '_ \| __|    
qdrant-1  | | (_| | (_| | | | (_| | | | | |_     
qdrant-1  |  \__, |\__,_|_|  \__,_|_| |_|\__| 
qdrant-1  |     |_|                              
qdrant-1  |                                      
qdrant-1  | Version: 1.9.0, build: b99d5074      
qdrant-1  | Access web UI at http://localhost:6333/dashboard                                      
qdrant-1  |                                      
qdrant-1  | 2026-04-17T09:29:30.704283Z  INFO storage::content_manager::consensus::persistent: Initializing new raft state at ./storage/raft_state.json
qdrant-1  | 2026-04-17T09:29:30.738578Z  INFO qdrant: Distributed mode disabled                   
qdrant-1  | 2026-04-17T09:29:30.739468Z  INFO qdrant: Telemetry reporting enabled, id: d191d055-d949-4666-b674-1d81849bde2d                        
qdrant-1  | 2026-04-17T09:29:30.749054Z  INFO qdrant::tonic: Qdrant gRPC listening on 6334        
qdrant-1  | 2026-04-17T09:29:30.749120Z  INFO qdrant::tonic: TLS disabled for gRPC API            
qdrant-1  | 2026-04-17T09:29:30.749519Z  INFO qdrant::actix: TLS disabled for REST API            
qdrant-1  | 2026-04-17T09:29:30.750862Z  INFO qdrant::actix: Qdrant HTTP listening on 6333        
qdrant-1  | 2026-04-17T09:29:30.752067Z  INFO actix_server::builder: Starting 11 workers          
qdrant-1  | 2026-04-17T09:29:30.752101Z  INFO actix_server::server: Actix runtime found; starting in Actix runtime


## Ex2.4

Edited docker-compose.yml
Edited report.md
Viewed report.md:305-308

Lỗi `container production-qdrant-1 is unhealthy` xảy ra vì lệnh kiểm tra sức khỏe (**Healthcheck**) mà bạn thiết lập cho Qdrant trong file `docker-compose.yml` đang bị thất bại.

### Phân tích nguyên nhân:
Trong file `docker-compose.yml`, bạn có đoạn:
```yaml
  qdrant:
    image: qdrant/qdrant:v1.9.0
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/health"]
```
**Vấn đề là:** Image chính thức của Qdrant (`v1.9.0`) được xây dựng rất tinh gọn và **không có sẵn lệnh `curl`** bên trong. Khi Docker chạy lệnh `curl` để kiểm tra, nó sẽ báo lỗi "command not found", dẫn đến trạng thái `unhealthy`.

Vì dịch vụ `agent` của bạn có cấu hình `depends_on: qdrant: condition: service_healthy`, nên khi Qdrant bị coi là "không khỏe", `agent` sẽ bị chặn không cho khởi động.

### 🚀 CÁCH KHẮC PHỤC

Bạn có thể thay thế `curl` bằng `wget` (thường có sẵn trong các image Linux slim) hoặc đơn giản nhất là bỏ qua bước kiểm tra sức khỏe này của Qdrant để app có thể chạy tiếp.

**Cách 1: Sửa Healthcheck sang dùng `wget` (Khuyên dùng)**
Sửa lại phần `healthcheck` của dịch vụ `qdrant` trong file `docker-compose.yml`:

```yaml
    healthcheck:
      # Thay curl bằng wget
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:6333/health"]
      interval: 15s
      timeout: 5s
      retries: 3
```

**Cách 2: Đơn giản hóa (Nếu Cách 1 vẫn lỗi)**
Nếu cả `wget` cũng không có, bạn có thể tạm thời yêu cầu `agent` chỉ cần đợi Qdrant "khởi động xong" chứ không cần đợi "khỏe hẳn":

Trong phần `agent`, sửa lại `depends_on`:
```yaml
    depends_on:
      redis:
        condition: service_healthy
      qdrant:
        condition: service_started  # Đổi từ service_healthy thành service_started
```

**Sau khi sửa, bạn chạy lại lệnh:**
```bash
docker compose -f 02-docker/production/docker-compose.yml up --build
```

*(Mẹo: Bạn có thể kiểm tra xem container đang bị lỗi gì bằng cách gõ lệnh `docker inspect --format "{{json .State.Health}}" production-qdrant-1` để xem chi tiết log của healthcheck).*


(lab12) D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment>docker compose -f 02-docker/production/docker-compose.yml up --build
time="2026-04-17T16:36:32+07:00" level=warning msg="D:\\vinlab\\lab12\\day12_ha-tang-cloud_va_deployment\\02-docker\\production\\docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion"
[+] Building 2.0s (19/19) FINI
 => [internal] load loc  0.0s 
 => => reading fro 610B  0.0s
 => [internal] load bui  0.0s
 => => transferr 3.32kB  0.0s 
 => [internal] load met  1.4s 
 => [internal] load .do  0.0s
 => => transferring  2B  0.0s 
 => [builder 1/5] FROM   0.0s 
 => => resolve docker.i  0.0s 
 => [internal] load bui  0.0s 
 => => transferrin 237B  0.0s 
 => CACHED [runtime 2/8  0.0s 
 => CACHED [runtime 3/8  0.0s 
 => CACHED [builder 2/5  0.0s 
 => CACHED [builder 3/5  0.0s 
 => CACHED [builder 4/5  0.0s 
 => CACHED [builder 5/5  0.0s 
 => CACHED [runtime 4/8  0.0s 
 => CACHED [runtime 5/8  0.0s 
 => CACHED [runtime 6/8  0.0s 
 => CACHED [runtime 7/8  0.0s 
 => CACHED [runtime 8/8  0.0s 
 => exporting to image   0.1s 
 => => exporting layers  0.0s 
 => => exporting manife  0.0s 
 => => exporting config  0.0s 
 => => exporting attest  0.0s 
 => => exporting manife  0.0s 
 => => naming to docker  0.0s 
 => => unpacking to doc  0.0s 
 => resolving provenanc  0.0s 
[+] Running 2/3
 ✔ production-agent           
    Built0.0s
[+] Running 2/3oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
[+] Running 2/3oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
[+] Running 2/3oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
[+] Running 2/3oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
[+] Running 2/3oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
[+] Running 2/3oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
[+] Running 3/4oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
 ✔ Container production-redis-[+] Running 3/4 
 ✔ production-agent               Built0.0s .6s
 ✔ Container production-redis-[+] Running 4/4 
 ✔ production-agent               Built0.0s .6s
 ✔ Container production-redis-1   Running0.0s 
 ✔ Container production-qdrant-1  Recreated0.6s
 ✔ Container production-agent-1   Recreated0.2s
Attaching to agent-1, nginx-1, qdrant-1, redis-1
qdrant-1  |            _                 _
qdrant-1  |   __ _  __| |_ __ __ _ _ __ | |_                
qdrant-1  |  / _` |/ _` | '__/ _` | '_ \| __|               
qdrant-1  | | (_| | (_| | | | (_| | | | | |_                
qdrant-1  |  \__, |\__,_|_|  \__,_|_| |_|\__|               
qdrant-1  |     |_|                                         
qdrant-1  |                   
qdrant-1  | Version: 1.9.0, build: b99d5074                 
qdrant-1  | Access web UI at http://localhost:6333/dashboard
qdrant-1  |                   
qdrant-1  | 2026-04-17T09:36:35.849976Z  INFO storage::content_manager::consensus::persistent: Loading raft state from ./storage/raft_state.json      
qdrant-1  | 2026-04-17T09:36:35.869374Z  INFO qdrant: Distributed mode disabled           
qdrant-1  | 2026-04-17T09:36:35.869881Z  INFO qdrant: Telemetry reporting enabled, id: aaadc3ad-9044-48da-a574-c0064e4a298e
qdrant-1  | 2026-04-17T09:36:35.876668Z  INFO qdrant::actix: TLS disabled for REST API    
qdrant-1  | 2026-04-17T09:36:35.877504Z  INFO qdrant::actix: Qdrant HTTP listening on 6333

qdrant-1  | 2026-04-17T09:36:35.888775Z  INFO qdrant::tonic: TLS disabled for gRPC API
dependency failed to start: coqdrant-1  | 2026-04-17T09:36:35.888775Z  INFO qdrant::tonic: TLS disabled for gRPC API    
dependency failed to start: container production-qdrant-1 is unhealthy

(lab12) D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment>docker compose -f 02-docker/production/docker-compose.yml up --build
time="2026-04-17T16:39:13+07:00" level=warning msg="D:\\vinlab\\lab12\\day12_ha-tang-cloud_va_deployment\\02-docker\\production\\docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion"
[+] Building 1.7s (19/19) FINI
 => [internal] load loc  0.0s 
 => => reading fro 610B  0.0s
 => [internal] load bui  0.0s
 => => transferr 3.32kB  0.0s 
 => [internal] load met  0.9s 
 => [internal] load .do  0.0s
 => => transferring  2B  0.0s 
 => [builder 1/5] FROM   0.0s 
 => => resolve docker.i  0.0s 
 => [internal] load bui  0.0s 
 => => transferrin 237B  0.0s
 => CACHED [runtime 2/8  0.0s 
 => CACHED [runtime 3/8  0.0s 
 => CACHED [builder 2/5  0.0s 
 => CACHED [builder 3/5  0.0s 
 => CACHED [builder 4/5  0.0s 
 => CACHED [builder 5/5  0.0s 
 => CACHED [runtime 4/8  0.0s 
 => CACHED [runtime 5/8  0.0s 
 => CACHED [runtime 6/8  0.0s 
 => CACHED [runtime 7/8  0.0s 
 => CACHED [runtime 8/8  0.0s 
 => exporting to image   0.1s 
 => => exporting layers  0.0s 
 => => exporting manife  0.0s 
 => => exporting config  0.0s 
 => => exporting attest  0.0s 
 => => exporting manife  0.0s 
 => => naming to docker  0.0s 
 => => unpacking to doc  0.0s 
 => resolving provenanc  0.0s 
[+] Running 2/3
 ✔ production-agent           
    Built0.0s
[+] Running 2/3oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
[+] Running 2/3oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
[+] Running 2/3oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
[+] Running 2/3oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
[+] Running 3/4oduction-redis-
 ✔ production-agent               Built0.0s roduction-qdrant
 ✔ Container production-redis-[+] Running 4/4 
 ✔ production-agent               Built0.0s .4s
 ✔ Container production-redis-1   Running0.0s 
 ✔ Container production-qdrant-1  Recreated0.4s
 ✔ Container production-agent-1   Recreated0.1s
Attaching to agent-1, nginx-1, qdrant-1, redis-1
qdrant-1  |            _                 _
qdrant-1  |   __ _  __| |_ __ __ _ _ __ | |_                
qdrant-1  |  / _` |/ _` | '__/ _` | '_ \| __|               
qdrant-1  | | (_| | (_| | | | (_| | | | | |_                
qdrant-1  |  \__, |\__,_|_|  \__,_|_| |_|\__|
qdrant-1  |     |_|                                         
qdrant-1  |                   
qdrant-1  | Version: 1.9.0, build: b99d5074                 
qdrant-1  | Access web UI at http://localhost:6333/dashboard
qdrant-1  |                   
qdrant-1  | 2026-04-17T09:39:16.184856Z  INFO storage::content_manager::consensus::persistent: Loading raft state from ./storage/raft_state.json      
qdrant-1  | 2026-04-17T09:39:16.199146Z  INFO qdrant: Distributed mode disabled           
qdrant-1  | 2026-04-17T09:39:16.199319Z  INFO qdrant: Telemetry reporting enabled, id: f93ac682-5bdd-41b4-bc1b-8fe9d7e49e48
qdrant-1  | 2026-04-17T09:39:16.202518Z  INFO qdrant::actix: TLS disabled for REST API    
qdrant-1  | 2026-04-17T09:39:16.202828Z  INFO qdrant::actix: Qdrant HTTP listening on 6333

qdrant-1  | 2026-04-17T09:39:16.202852Z  INFO actix_server::builder: Starting 11 workers  
qdrant-1  | 2026-04-17T09:39:16.202914Z  INFO actix_server::server: Actix runtime found; starting in Actix runtime      
qdrant-1  | 2026-04-17T09:39:16.209141Z  INFO qdrant::tonic: Qdrant gRPC listening on 6334

qdrant-1  | 2026-04-17T09:39:16.209180Z  INFO qdrant::tonic: TLS disabled for gRPC API    
nginx-1   | /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
nginx-1   | /docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/     
nginx-1   | /docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
nginx-1   | 10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
nginx-1   | 10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
nginx-1   | /docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh 
nginx-1   | /docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
nginx-1   | /docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
nginx-1   | /docker-entrypoint.sh: Configuration complete; ready for start up             
agent-1   | INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)       
agent-1   | INFO:     Started parent process [1]            
agent-1   | INFO:     Started server process [9]
agent-1   | INFO:     Started server process [8]            
agent-1   | INFO:     Waiting for application startup.      
agent-1   | INFO:     Waiting for application startup.      
agent-1   | {"time":"2026-04-17 09:39:18,743","level":"INFO","msg":"Starting agent..."}   
agent-1   | {"time":"2026-04-17 09:39:18,743","level":"INFO","msg":"Starting agent..."}   
agent-1   | {"time":"2026-04-17 09:39:18,843","level":"INFO","msg":"Agent ready"}
agent-1   | {"time":"2026-04-17 09:39:18,843","level":"INFO","msg":"Agent ready"}         
agent-1   | INFO:     Application startup complete.     
agent-1   | INFO:     127.0.0.1:34420 - "GET /health HTTP/1.1" 200 OK
agent-1   | INFO:     172.18.0.5:37146 - "GET /health HTTP/1.1" 200 OK
agent-1   | INFO:     127.0.0.1:48932 - "GET /health HTTP/1.1" 200 OK


D:\vinlab\lab12\day12_ha-tang-cloud_va_deployment>curl http://localhost/health            
{"status":"ok","uptime_seconds":14.0,"version":"2.0.0","timestamp":"2026-04-17T09:39:32.781295"}


Viết 1 API nhỏ (Python Flask hoặc Node Express) với 2 endpoint: / và /health.
Viết Dockerfile, build và chạy với -p 8080:5000.
Xong khi: curl localhost:8080/health trả về ok, và image dưới 200MB (gợi ý: dùng base image slim hoặc alpine).

B1: chuẩn bị API
**Flask**:

```
cat << 'EOF' > app.py
from flask import Flask, jsonify
import time

app = Flask(__name__)
start_time = time.time()

@app.route('/', methods=['GET'])
def index():
    return jsonify({"message": "Welcome to the API!"}), 200

@app.route('/health', methods=['GET'])
def health():
    return jsonify({
        "status": "healthy",
        "uptime_seconds": round(time.time() - start_time, 2)
    }), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

**Node.js Express**:
```
cat << 'EOF' > index.js
const express = require('express');
const app = express();
const PORT = 5000;

app.get('/', (req, res) => {
  res.status(200).json({ message: 'Welcome to the API!' });
});

app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    uptime: process.uptime(),
    timestamp: new Date().toISOString()
  });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server is running on port ${PORT}`);
});
EOF
```






result: 
<img width="661" height="190" alt="image" src="https://github.com/user-attachments/assets/2e6e9c8f-27b7-4bde-b9c3-39598469f1d7" />

# Toolify Core — VPS Server Preparation Guide
# Coolify + Ubuntu 24 LTS

## What this guide covers
- Background remover (rembg + Python Flask)
- PDF to JPG (Ghostscript)
- PDF to Word (LibreOffice)
- Word to PDF (LibreOffice)
- All served as lightweight API endpoints

---

## Step 1 — SSH into your VPS

```bash
ssh root@YOUR_VPS_IP
```

---

## Step 2 — Update system & install core dependencies

```bash
apt update && apt upgrade -y
apt install -y python3 python3-pip python3-venv \
  ghostscript libreoffice \
  libgl1 libglib2.0-0 libsm6 libxrender1 libxext6 \
  nginx curl git
```

---

## Step 3 — Create API project folder

```bash
mkdir -p /opt/toolifycore-api
cd /opt/toolifycore-api
python3 -m venv venv
source venv/bin/activate
```

---

## Step 4 — Install Python packages

```bash
pip install flask flask-cors rembg[gpu] pillow gunicorn onnxruntime
```

> Note: If your VPS has no GPU, use `rembg` instead of `rembg[gpu]`

---

## Step 5 — Create the API server

Create file: `/opt/toolifycore-api/app.py`

```python
from flask import Flask, request, send_file, jsonify
from flask_cors import CORS
from rembg import remove
from PIL import Image
import io, os, subprocess, tempfile

app = Flask(__name__)
CORS(app, origins=["https://toolifycore.com"])

# ─── Background Removal ───────────────────────────────────
@app.route('/api/remove-bg', methods=['POST'])
def remove_bg():
    if 'image' not in request.files:
        return jsonify({'error': 'No image provided'}), 400
    
    file = request.files['image']
    input_bytes = file.read()
    
    try:
        output_bytes = remove(input_bytes)
        return send_file(
            io.BytesIO(output_bytes),
            mimetype='image/png',
            as_attachment=False,
            download_name='removed-bg.png'
        )
    except Exception as e:
        return jsonify({'error': str(e)}), 500


# ─── PDF to JPG ───────────────────────────────────────────
@app.route('/api/pdf-to-jpg', methods=['POST'])
def pdf_to_jpg():
    if 'file' not in request.files:
        return jsonify({'error': 'No file provided'}), 400
    
    file = request.files['file']
    dpi = request.form.get('dpi', '150')
    
    with tempfile.TemporaryDirectory() as tmpdir:
        input_path = os.path.join(tmpdir, 'input.pdf')
        output_path = os.path.join(tmpdir, 'page')
        file.save(input_path)
        
        subprocess.run([
            'gs', '-dBATCH', '-dNOPAUSE', '-sDEVICE=jpeg',
            f'-r{dpi}', f'-sOutputFile={output_path}_%03d.jpg',
            input_path
        ], check=True, capture_output=True)
        
        jpg_files = sorted([f for f in os.listdir(tmpdir) if f.endswith('.jpg')])
        
        if len(jpg_files) == 1:
            with open(os.path.join(tmpdir, jpg_files[0]), 'rb') as f:
                return send_file(io.BytesIO(f.read()), mimetype='image/jpeg')
        
        # Multiple pages: return first page (ZIP support can be added later)
        with open(os.path.join(tmpdir, jpg_files[0]), 'rb') as f:
            return send_file(io.BytesIO(f.read()), mimetype='image/jpeg')


# ─── PDF to Word ──────────────────────────────────────────
@app.route('/api/pdf-to-word', methods=['POST'])
def pdf_to_word():
    if 'file' not in request.files:
        return jsonify({'error': 'No file provided'}), 400
    
    file = request.files['file']
    
    with tempfile.TemporaryDirectory() as tmpdir:
        input_path = os.path.join(tmpdir, 'input.pdf')
        file.save(input_path)
        
        subprocess.run([
            'libreoffice', '--headless', '--convert-to', 'docx',
            '--outdir', tmpdir, input_path
        ], check=True, capture_output=True)
        
        docx_path = os.path.join(tmpdir, 'input.docx')
        with open(docx_path, 'rb') as f:
            return send_file(
                io.BytesIO(f.read()),
                mimetype='application/vnd.openxmlformats-officedocument.wordprocessingml.document',
                download_name='converted-toolifycore.docx'
            )


# ─── Word to PDF ──────────────────────────────────────────
@app.route('/api/word-to-pdf', methods=['POST'])
def word_to_pdf():
    if 'file' not in request.files:
        return jsonify({'error': 'No file provided'}), 400
    
    file = request.files['file']
    
    with tempfile.TemporaryDirectory() as tmpdir:
        input_path = os.path.join(tmpdir, 'input.docx')
        file.save(input_path)
        
        subprocess.run([
            'libreoffice', '--headless', '--convert-to', 'pdf',
            '--outdir', tmpdir, input_path
        ], check=True, capture_output=True)
        
        pdf_path = os.path.join(tmpdir, 'input.pdf')
        with open(pdf_path, 'rb') as f:
            return send_file(
                io.BytesIO(f.read()),
                mimetype='application/pdf',
                download_name='converted-toolifycore.pdf'
            )


# ─── Health check ─────────────────────────────────────────
@app.route('/api/health', methods=['GET'])
def health():
    return jsonify({'status': 'ok', 'service': 'Toolify Core API'})


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001, debug=False)
```

---

## Step 6 — Test the API

```bash
cd /opt/toolifycore-api
source venv/bin/activate
python app.py
# Visit http://YOUR_VPS_IP:5001/api/health
```

---

## Step 7 — Run with Gunicorn (production)

```bash
pip install gunicorn
gunicorn --workers 2 --bind 0.0.0.0:5001 app:app
```

---

## Step 8 — Create systemd service (auto-start)

Create file: `/etc/systemd/system/toolifycore-api.service`

```ini
[Unit]
Description=Toolify Core API Server
After=network.target

[Service]
User=root
WorkingDirectory=/opt/toolifycore-api
Environment="PATH=/opt/toolifycore-api/venv/bin"
ExecStart=/opt/toolifycore-api/venv/bin/gunicorn --workers 2 --bind 0.0.0.0:5001 app:app
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl daemon-reload
systemctl enable toolifycore-api
systemctl start toolifycore-api
systemctl status toolifycore-api
```

---

## Step 9 — Deploy on Coolify (recommended)

Instead of systemd, you can deploy the API as a Coolify app:

1. Push `/opt/toolifycore-api` to a GitHub repo (private)
2. Coolify → New Resource → Python App
3. Set start command: `gunicorn --workers 2 --bind 0.0.0.0:5001 app:app`
4. Set port: `5001`
5. Add domain: `api.toolifycore.com`
6. Enable SSL

---

## Step 10 — Nginx reverse proxy (optional)

If you want `api.toolifycore.com` to serve on port 443:

```nginx
server {
    listen 443 ssl;
    server_name api.toolifycore.com;

    # SSL handled by Coolify or certbot

    location /api/ {
        proxy_pass http://127.0.0.1:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        client_max_body_size 20M;
    }
}
```

```bash
nginx -t && systemctl reload nginx
```

---

## Step 11 — Update your tool HTML files

In `remove-bg.html`, update the server URL input default:
```
https://api.toolifycore.com/api/remove-bg
```

In future `pdf-to-jpg.html`:
```
https://api.toolifycore.com/api/pdf-to-jpg
```

---

## API Endpoints Summary

| Endpoint | Method | Input | Output |
|---|---|---|---|
| `/api/health` | GET | — | JSON status |
| `/api/remove-bg` | POST | image file | PNG (transparent) |
| `/api/pdf-to-jpg` | POST | PDF file | JPG image |
| `/api/pdf-to-word` | POST | PDF file | DOCX file |
| `/api/word-to-pdf` | POST | DOCX file | PDF file |

---

## Security notes

- API is CORS-restricted to `toolifycore.com` only
- Add rate limiting with `flask-limiter` if traffic grows:

```bash
pip install flask-limiter
```

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(app, key_func=get_remote_address,
                  default_limits=["100 per hour"])
```

- Max file size: set in nginx `client_max_body_size 20M`
- Temp files auto-deleted after each request

---

## VPS Resource Requirements

| Service | RAM | CPU | Storage |
|---|---|---|---|
| rembg (background removal) | 1.5GB | 2 vCPU | 500MB models |
| Ghostscript (PDF to JPG) | 256MB | 1 vCPU | — |
| LibreOffice (PDF/Word) | 512MB | 1 vCPU | 300MB |
| **Total recommended VPS** | **4GB RAM** | **2 vCPU** | **20GB SSD** |

Your current Contabo VPS should handle this comfortably.

---

## Estimated server cost

- Contabo 4GB VPS: ~€5/month
- rembg model downloads: free (one-time ~150MB)
- Total additional cost: €0 (using existing VPS)

---
*Toolify Core Server Guide v1.0*

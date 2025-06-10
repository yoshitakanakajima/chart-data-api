# chart-data-api
Docker上で動作するFastAPI Webサービス

## ディレクトリ構成
以下の構成になります。
アプリケーションは、APIのバージョンごとに作成。
Dockerfileは、コンテナごとに作成（現状はアプリのみ、nginxやDBを追加する場合、それぞれディレクトを作成して、そこに配置）。
ライブラリインストールファイルや、コンテナ定義ファイルなどは、ルートに配置。

```
.
├── src
│   └── v1
│       ├── __init__.py
│       └── main.py
├── docker
│       └── api
│           └── Dockerfile
├── docker-compose.yml
└── requirements.txt
```

## ライブラリのインストール
アプリケーションに必要なライブラリと、そのバージョンを定義する。
* **FastAPI** は`2025/06/07`現在の最新バージョンを使用。
* **Pydantic（パイダンティック）** は、Pythonのデータバリデーションライブラリで、`2025/06/07`現在の最新バージョンを使用。
* **Uvicorn（ユビコーン）** は、非同期サーバーゲートウェイインターフェース（Asynchronous Server Gateway Interface）で、非同期対応の「Python Webサーバー」「フレームワーク」「アプリケーション間標準インターフェース」を提供 <sup>※1</sup>

```requirements.txt
fastapi>=0.115.12,<0.116.0
pydantic>=2.11.5,<2.12.0
uvicorn>=0.34.3,<0.35.0
```

※１ WSGI（Web Server Gateway Interface）の後継。WSGIは*同期Pythonアプリ*の標準を提供。長時間ポーリングなどの存続期間の長い接続は許可されない。

## Dockerファイルの作成

### docker-compose.yml
Webサービスを定義し、ポートマッピングやボリュームの設定する。

```docker-compose.yml
version: '3'

services:
  web:
    container_name: fast_api
    build:
      context: .
      dockerfile: docker/api/Dockerfile
    ports:
      - "8000:8000"
    volumes:
      - ./src:/var/www/api/src
```

### Dockerfile
ログ出力をリアルタイムで行う（バッファリングを有効にすると、異常終了した際、バッファに残ったログは出力されない）
pipでライブラリインストールを行う際、キャッシュを使うと早くなるが、ダウンロードしたファイルが残り続けて容量が重たくなる。
しかし、キャッシュを消すコマンドがないため、キャッシュを使わずにインストールを実行する。
Uvicornにプロキシ経由（Nginxなどのロードバランサー）でHTTPSで動作しているアプリケーションに対して、送信されるヘッダを信頼するよう`--proxy-headers`オプションを追加する。
`src.main:app` は、main.pyの `app = FastAPI()` を指す。

```Dockerfile
# Python3を親イメージ
FROM python:3.9

# Pythonの出力バッファリングを無効にする
ENV PYTHONUNBUFFERED 1

# pipを最新バージョンにアップグレード
RUN pip install --upgrade pip

# 作業ディレクトリを作成
WORKDIR /var/www/api

# ライブラリ定義を作業ディレクトリにコピー
COPY requirements.txt /var/www/api/requirements.txt

# ライブラリをインストール
RUN pip install --no-cache-dir --upgrade -r /var/www/api/requirements.txt

# コードを作業ディレクトリにコピー
COPY ./src /var/www/api/src

# ASGIの起動
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "80"]
```

## FastAPIコードを作成
`main.py`ファイルを作成する

```python
from typing import Union

from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello World"}

@app.get("/patients/{patient_id}")
def read_patient(patient_id: int, q: Union[str, None] = None):
    return {"patient_id": patient_id, "q": q}
```

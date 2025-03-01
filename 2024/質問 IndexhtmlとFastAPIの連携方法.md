#### Q.
- FastAPIとIndex.htmlをどのように連携させるのですか？例えば、Index.htmlにある取得ボタンを押すと、テキストボックスにFastAPIで取得した結果を表示するなど。どのようにコーディングしたらいよいですか？具体的なイメージがわかりません。

#### A.
- 
FastAPIと`index.html`を連携させるには、以下の手順を踏むことができます。具体的には、FastAPIをバックエンドとして使用し、JavaScriptでAPIリクエストを行って結果を表示します。以下に具体例を示します。

### 1. FastAPIのセットアップ

まず、FastAPIのエンドポイントを作成します。以下のような簡単な例を示します。

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# CORS設定を追加
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 必要に応じて許可するオリジンを指定
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/fetch_data")
async def fetch_data():
    return {"message": "Hello from FastAPI!"}
```

### 2. `index.html`の作成

次に、フロントエンドのHTMLファイルを作成します。ここでは、ボタンを押すとテキストボックスにAPIから取得したデータが表示されるようにします。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FastAPI and HTML Integration</title>
</head>
<body>
    <h1>FastAPI and HTML Integration</h1>
    <input type="text" id="result" placeholder="API response will appear here">
    <button onclick="fetchData()">取得</button>

    <script>
        async function fetchData() {
            try {
                const response = await fetch('http://127.0.0.1:8000/fetch_data');
                const data = await response.json();
                document.getElementById('result').value = data.message;
            } catch (error) {
                console.error('Error fetching data:', error);
            }
        }
    </script>
</body>
</html>
```

### 3. サーバーの起動

最後に、FastAPIサーバーを起動します。ターミナルで以下のコマンドを実行します：

```bash
uvicorn main:app --reload
```

このセットアップにより、ブラウザで`index.html`を開いてボタンを押すと、FastAPIからのレスポンスがテキストボックスに表示されます。

必要に応じてこれをカスタマイズしてみてください。さらに質問があれば、どうぞお知らせください！ 😊

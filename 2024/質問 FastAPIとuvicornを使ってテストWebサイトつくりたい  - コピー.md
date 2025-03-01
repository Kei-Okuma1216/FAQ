#### Q.
- 
この両者ＦａｓｔＡＰＩとuvicornを使ってテストWebサイトを作ろうと思います。データベースが裏に欲しいです。どうつくればよいですか？

#### A.
- 
素晴らしいですね！`FastAPI`と`uvicorn`を使ってテストWebサイトを作成し、データベースを接続する手順を説明します。ここでは、SQLiteデータベースを使った例を紹介します。SQLiteは軽量で扱いやすく、テストや小規模プロジェクトに最適です。

### 1. 必要なパッケージのインストール
まずは、必要なパッケージをインストールします。以下のコマンドを実行してください：

```bash
pip install fastapi uvicorn sqlalchemy databases sqlite
```

`SQLAlchemy`はPythonのORM（オブジェクト関係マッピング）ライブラリで、データベース操作を容易にします。`databases`は非同期データベース接続を提供します。

### 2. FastAPIアプリケーションの作成
次に、FastAPIアプリケーションを作成し、データベースに接続するコードを記述します。

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sqlalchemy import create_engine, Column, Integer, String, MetaData
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# データベースの設定
DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# モデル定義
class Item(Base):
    __tablename__ = "items"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    description = Column(String, index=True)

# テーブル作成
Base.metadata.create_all(bind=engine)

# FastAPIインスタンス作成
app = FastAPI()

# Pydanticモデル定義
class ItemCreate(BaseModel):
    name: str
    description: str

@app.post("/items/", response_model=ItemCreate)
def create_item(item: ItemCreate):
    db = SessionLocal()
    db_item = Item(name=item.name, description=item.description)
    db.add(db_item)
    db.commit()
    db.refresh(db_item)
    db.close()
    return db_item

@app.get("/items/{item_id}", response_model=ItemCreate)
def read_item(item_id: int):
    db = SessionLocal()
    db_item = db.query(Item).filter(Item.id == item_id).first()
    db.close()
    if db_item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return db_item
```

### 3. サーバーの起動
FastAPIアプリケーションを起動します。

```bash
uvicorn main:app --reload
```

これで、FastAPIアプリケーションが起動し、SQLiteデータベースと接続された状態になります。

### 4. HTMLとの連携
先ほどの`index.html`ファイルにAPIからデータを取得するコードを追加します。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FastAPI and SQLite Integration</title>
</head>
<body>
    <h1>FastAPI and SQLite Integration</h1>
    <input type="text" id="name" placeholder="Name">
    <input type="text" id="description" placeholder="Description">
    <button onclick="createItem()">Create Item</button>
    <button onclick="fetchItem()">Fetch Item</button>
    <input type="number" id="item_id" placeholder="Item ID">
    <div id="result"></div>

    <script>
        async function createItem() {
            const name = document.getElementById('name').value;
            const description = document.getElementById('description').value;
            const response = await fetch('http://127.0.0.1:8000/items/', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ name, description })
            });
            const data = await response.json();
            document.getElementById('result').innerText = `Created: ${data.name} - ${data.description}`;
        }

        async function fetchItem() {
            const itemId = document.getElementById('item_id').value;
            const response = await fetch(`http://127.0.0.1:8000/items/${itemId}`);
            const data = await response.json();
            if (response.ok) {
                document.getElementById('result').innerText = `Fetched: ${data.name} - ${data.description}`;
            } else {
                document.getElementById('result').innerText = `Error: ${data.detail}`;
            }
        }
    </script>
</body>
</html>
```

これで、`index.html`で入力されたデータをFastAPIを通してSQLiteデータベースに保存し、取得することができます。

必要に応じてコードをカスタマイズし、プロジェクトの特定の要件に合わせてください。何か質問や他にお手伝いできることがあれば、教えてくださいね！ 😊

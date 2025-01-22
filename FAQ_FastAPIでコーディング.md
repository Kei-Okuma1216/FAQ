# 質問・回答

<details><summary>APIをテストしたい</summary>

## Q.
FastAPIでmain.pyをこのようにつくりました。しかしuvicornでテスト時にhttp://127.0.0.1:8000/に続けて、どのように入力すればよいですか？APIのテストをしたい。
>from datetime import datetime, time, timedelta 
>from typing import Union 
>from uuid import UUID 
>from fastapi import Body, FastAPI 
>app = FastAPI() 
>@app.put("/items/{item_id}") 
>async def read_items( item_id: UUID, start_datetime:
> datetime = Body(), end_datetime: datetime = Body(), process_after: 
>timedelta = Body(), 
>repeat_at: Union[time, None] = Body(default=None), ): start_process = start_datetime + process_after duration = end_datetime start_process return { "item_id": item_id, "start_datetime": start_datetime, "end_datetime": end_datetime, "process_after": process_after, "repeat_at": repeat_at, "start_process": start_process, "duration": duration, }

## A.
FastAPIで作成したAPIをテストするために、以下の手順を参考にしてください。

まず、`uvicorn`を使ってFastAPIアプリケーションを起動します。ターミナルで以下のコマンドを実行します：

```bash
uvicorn main:app --reload
```

このコマンドは、`main.py`ファイルの中にある`app`オブジェクト（FastAPIアプリケーション）を起動します。`--reload`オプションは、コードを変更すると自動的にサーバーが再起動するようにします。

サーバーが起動すると、`http://127.0.0.1:8000` でアクセスできるようになります。

次に、APIをテストするために、`PUT`リクエストを送信する必要があります。例えば、以下のようなJSONデータを使用します：

```json
{
  "start_datetime": "2025-01-16T10:00:00Z",
  "end_datetime": "2025-01-16T12:00:00Z",
  "process_after": 3600,  // 1 hour in seconds
  "repeat_at": "10:30:00" // Optional
}
```

このデータを `http://127.0.0.1:8000/items/{item_id}` に送信します。ここで、`{item_id}` はUUID形式のIDです。

`curl`コマンドを使用してリクエストを送信する場合の例を示します：

```bash
curl -X PUT "http://127.0.0.1:8000/items/123e4567-e89b-12d3-a456-426614174000" -H "Content-Type: application/json" -d @data.json
```

ここで、`data.json`は先ほどのJSONデータが入っているファイルです。

あるいは、PostmanやInsomniaなどのAPIテストツールを使用して、リクエストを作成することもできます。これらのツールでは、リクエストのURL、メソッド（PUT）、ヘッダー（Content-Type: application/json）、およびボディ（JSONデータ）を設定することで、簡単にテストが行えます。

もし他に詳しく知りたいことや、疑問があれば教えてくださいね。😊

</details>

<details><summary>FastAPIのVersion情報</summary>

## Q.
自分の使っているFastAPIのヴァージョンはどこでわかりますか？

## A.
FastAPIのバージョンを確認するには、いくつかの方法があります。

1. **コマンドラインから確認する方法**:
   ターミナルまたはコマンドプロンプトで次のコマンドを実行します：

   ```bash
   pip show fastapi
   ```

   これにより、FastAPIのバージョン情報やインストール場所などの詳細が表示されます。出力の中の`Version`フィールドにバージョン番号が記載されています。

2. **Pythonスクリプト内で確認する方法**:
   Pythonスクリプト内でFastAPIのバージョンを確認することもできます。以下のコードを実行して、バージョンを表示します：

   ```python
   import fastapi
   print(fastapi.__version__)
   ```

   このコードを実行すると、現在インストールされているFastAPIのバージョンが表示されます。

これらの方法で、簡単にFastAPIのバージョンを確認することができます。お試しください！他にも質問があれば、どうぞ教えてくださいね。😊

</details>

<details><summary>model_config = {"extra": "forbid"} はハードコートするべき文なのですね？</summary>

## Q.
model_config = {"extra": "forbid"} はハードコートするべき文なのですね？

## A.
`model_config = {"extra": "forbid"}` は、Pydanticモデルに対して余分なフィールドを許可しない設定を行うための重要な設定です。しかし、すべての状況で必ずしもハードコードする必要はありません。状況に応じて柔軟に使用することが望ましいです。

例えば、次の場合にはハードコードすることをおすすめします：
1. **セキュリティの確保**：予期しないフィールドがデータに含まれている場合、エラーを発生させることでセキュリティを強化することができます。
2. **データの一貫性**：特定のフィールドのみを受け入れることで、データの一貫性を維持しやすくなります。

ただし、柔軟性が必要な場合には、設定を変更することも検討してください。例えば、開発やデバッグ段階でより多くの情報を受け入れる必要がある場合は、`extra: "allow"` に設定することもあります。

Pydanticの設定オプションについてもっと詳しく知りたい場合は、[Pydanticの公式ドキュメント](https://pydantic-docs.helpmanual.io/)を参照してください。

どうでしょうか、他にも質問があれば教えてください！


</details>

<details><summary>EdgeでURL欄に移動するショートカットは何ですか？</summary>

## Q.
EdgeでURL欄に移動するショートカットは何ですか？

## A.
EdgeでURL欄に移動するショートカットは、通常「Ctrl + L」です。これでブラウザのアドレスバーに焦点が当たり、キーボードでURLを簡単に入力できます。

他に何かお手伝いできることがありますか？


</details>

<details><summary>dbの依存性注入</summary>

## Q.
このようなコードは依存性注入で使いますか？ 
```python
 async def get_db(): db = DBSession() try: yield db finally: db.close()
```
## A.
はい、このコードは依存性注入に使うことができます。`get_db`関数はデータベースセッションを生成し、そのセッションを外部のコードに渡すためのコンテキストマネージャーとして機能します。これにより、データベースセッションの管理を外部のコードから行うことができ、テストや再利用が容易になります。

例えば、以下のように依存性注入を使用して、データベースセッションを関数に渡すことができます：

```python
async def some_function(db):
    # データベースセッションを使用して何かを行う
    result = await db.query("SELECT * FROM some_table")
    return result

async def main():
    db = await get_db()
    result = await some_function(db)
    print(result)
```

このようにすることで、`some_function`はデータベースセッションを引数として受け取り、そのセッションを使用してデータベース操作を行います。これにより、テスト時には別のデータベースセッションを注入することができ、実際のデータベースとは独立してテストを行うことができます。

何か他に質問がありますか？

もちろんです！もう一度説明しますね。

依存性注入とは、コードの依存関係を外部から注入する手法です。これにより、コードのテストや再利用が容易になります。例えば、データベースセッションを外部から注入することで、テスト時には実際のデータベースとは独立してテストを行うことができます。

具体的には、以下のような手順で依存性注入を行います：

1. **コンテキストマネージャーを作成**: `get_db`関数はデータベースセッションを生成し、そのセッションを外部のコードに渡すためのコンテキストマネージャーとして機能します。

2. **依存性を注入**: `some_function`関数はデータベースセッションを引数として受け取り、そのセッションを使用してデータベース操作を行います。

3. **テスト時の別のセッションの使用**: テスト時には、`get_db`関数を使って別のデータベースセッションを注入し、実際のデータベースとは独立してテストを行います。

以下は具体的なコード例です：

```python
async def get_db():
    db = DBSession()
    try:
        yield db
    finally:
        await db.close()

async def some_function(db):
    # データベースセッションを使用して何かを行う
    result = await db.query("SELECT * FROM some_table")
    return result

async def main():
    db = await get_db()
    result = await some_function(db)
    print(result)
```

このようにすることで、`some_function`はデータベースセッションを引数として受け取り、そのセッションを使用してデータベース操作を行います。これにより、テスト時には別のデータベースセッションを注入することができ、実際のデータベースとは独立してテストを行うことができます。

何か他に質問がありますか？


</details>

<details><summary>Cookieのmax-ageを設定したい</summary>

## Q.
以下のコードでCookieのMax-Ageを取得したい。どう変えればよいですか？ 
> @app.get("/", response_class=HTMLResponse)
> async def read_root(request: Request):
> # もしtokenがついていたら、独自のページに遷移する。
>    token = request.cookies.get("token")
>   if token: 
>         # Tokenがある場合は /cde にリダイレクト 
>         if request.cookies("max_age"): 
>           print(f"max_age: {request.cookies('max_age')}") 
>           return RedirectResponse(url="/cde") 
>   return templates.TemplateResponse("login.html", {"request": request})

## A.
クッキーの`max-age`属性は通常クライアント（ブラウザ）側で管理されるものであり、サーバー側のリクエストオブジェクトから直接アクセスすることはできません。しかし、クッキーの有効期限情報を保持し、サーバー側で利用する方法はいくつかあります。

以下は、クッキーを設定する際にその有効期限情報をサーバー側の状態として保持し、それを利用して`max-age`を確認する方法の一例です：

1. **クッキーの有効期限情報を保存するための辞書を作成**：

```python
import json
from datetime import datetime, timedelta
from fastapi import FastAPI, Response, Request, HTTPException
from fastapi.responses import HTMLResponse, RedirectResponse
from fastapi.templating import Jinja2Templates

app = FastAPI()
templates = Jinja2Templates(directory="templates")

# 有効期限情報を保存するための辞書
token_expiry = {}

def save_token_expiry(token: str, expires: timedelta):
    expire_time = datetime.utcnow() + expires
    token_expiry[token] = expire_time

def load_token_expiry(token: str):
    return token_expiry.get(token)
```

2. **クッキーを設定するエンドポイントを作成**：

```python
@app.post("/set-cookie")
async def set_cookie(response: Response):
    token = "your-jwt-token"
    expires = timedelta(seconds=30)
    
    # 有効期限情報を保存
    save_token_expiry(token, expires)

    response.set_cookie(
        key="token",
        value=token,
        max_age=expires.total_seconds(),
    )
    return {"message": "Cookie set"}
```

3. **クッキーの有効期限を確認するエンドポイントを作成**：

```python
@app.get("/", response_class=HTMLResponse)
async def read_root(request: Request):
    token = request.cookies.get("token")
    if token:
        expire_time = load_token_expiry(token)
        if expire_time:
            max_age = (expire_time - datetime.utcnow()).total_seconds()
            print(f"max_age: {max_age}")
        return RedirectResponse(url="/cde")
    
    return templates.TemplateResponse("login.html", {"request": request})
```

この方法で、サーバー側でクッキーの`max-age`を確認することができます。

- `save_token_expiry`関数でクッキーの有効期限情報を辞書に保存します。
- `load_token_expiry`関数で保存された有効期限情報を取得します。
- `read_root`エンドポイントでクッキーの有効期限を確認し、その残り時間（`max_age`）を計算して出力します。

他にも質問があれば、どうぞお知らせください！😊

</details>

<details><summary>お弁当屋は閲覧できるが、他はNotAuthorizedエラーにしたい</summary>

## Q.
このコードに権限番号2を渡すと閲覧できるが、1や3ではNotAuthorizedエラーに遷移するコードはどう書けばよいですか？
## A.
`permission` の渡し方については、いくつかの方法があります。以下にいくつかの例を示します。

### 1. クエリパラメータとして渡す
URLのクエリパラメータとして権限を渡す方法です。例えば、次のようにリクエストURLを構成します：

```http
GET /today?permission=2
```

これにより、サーバー側で `permission` パラメータを受け取ります。

### 2. ヘッダーとして渡す
リクエストヘッダーとして権限を渡す方法です。例えば、次のようにします：

```python
@app.get("/today", response_class=HTMLResponse)
def shop_today_order(request: Request, hx_request: Optional[str] = Header(None), permission: Optional[int] = Header(None)):
    if permission != 2:
        raise NotAuthorizedException()
    
    # ... (以下、省略)
```

この場合、リクエストを送る際に以下のようにヘッダーを設定します：

```http
GET /today
Headers:
    permission: 2
```

### 3. クッキーとして渡す
クッキーに権限情報を保存し、それをサーバー側で読み取る方法です。例えば、ログイン時にクッキーを設定し、そのクッキーを利用して権限を確認します：

```python
@app.get("/today", response_class=HTMLResponse)
def shop_today_order(request: Request, hx_request: Optional[str] = Header(None)):
    permission = request.cookies.get("permission")
    if permission != "2":
        raise NotAuthorizedException()
    
    # ... (以下、省略)
```

この場合、ログイン時にクッキーを設定します：

```python
response.set_cookie(key="permission", value="2")
```

### 4. フォームデータとして渡す
フォームデータとして権限を渡す方法です。例えば、POSTリクエストで権限情報を送ります：

```python
@app.post("/today", response_class=HTMLResponse)
def shop_today_order(request: Request, hx_request: Optional[str] = Header(None), permission: int = Form(...)):
    if permission != 2:
        raise NotAuthorizedException()
    
    # ... (以下、省略)
```

これにより、フォームデータから `permission` パラメータを受け取ります。

### まとめ
どの方法を使用するかは、アプリケーションの要件やセキュリティポリシーに依存します。それぞれの方法にはメリットとデメリットがありますので、最適な方法を選択してください。

もしさらに詳しい説明や具体的なコード例が必要であれば、教えてくださいね！ 😊

</details>

<details><summary>aaa</summary>

## Q.


## A.


</details>

<details><summary>aaa</summary>

## Q.


## A.


</details>

<details><summary>aaa</summary>

## Q.


## A.


</details>

<details><summary>aaa</summary>

## Q.


## A.


</details>

<details><summary>aaa</summary>

## Q.


## A.


</details>
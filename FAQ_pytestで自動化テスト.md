# 質問・回答

<details><summary>responseの中身を知りたい</summary>

#### Q. uvicornサーバーを（手動で）起動してpytestを実行するとエラーになりました。
ログが以下のようになる。

collected 2 items

app\tests\test_login4.py FF                                                                                      [100%]

====================================================== FAILURES =======================================================
_________________________________________________ test_login_success __________________________________________________

    @pytest.mark.asyncio
    async def test_login_success():
        async with httpx.AsyncClient(base_url="https://127.0.0.1:8000", verify=ssl_context, follow_redirects=True) as client:

            # 1️⃣ Cookie なしで / にアクセス → /login にリダイレクトされるべき
            response = await client.get("/")
            assert response.status_code == 200  # login.html を受け取る

            # 2️⃣ /login に正しい認証情報を送信
            response = await client.post("/login", data={"username": "user1", "password": "user1"})
>           assert response.status_code == 303  # リダイレクトが発生
E           assert 200 == 303
E            +  where 200 = <Response [200 OK]>.status_code

app\tests\test_login4.py:36: AssertionError
このコードの一番最終行が36行目になります。
@pytest.mark.asyncio
async def test_login_success():
    async with httpx.AsyncClient(base_url="https://127.0.0.1:8000", verify=ssl_context, follow_redirects=True) as client:

        # 1️⃣ Cookie なしで / にアクセス → /login にリダイレクトされるべき
        response = await client.get("/")
        assert response.status_code == 200  # login.html を受け取る

        # 2️⃣ /login に正しい認証情報を送信
        response = await client.post("/login", data={"username": "user1", "password": "user1"})
        assert response.status_code == 303  # リダイレクトが発生
        assert "set-cookie" in response.headers  # Cookie が設定されているか確認

#### A.
おっけー！✨ `test_login_success` で `assert response.status_code == 303` のテストが失敗してるんだね！💡  

```
E           assert 200 == 303
E            +  where 200 = <Response [200 OK]>.status_code
```
**つまり、`POST /login` のリクエストが `200 OK` を返していて、`303 See Other` になっていない** ってこと！  

---

## **🔥 エラーの原因を探る**
### **✅ ① `/login` のレスポンスを確認**
テストコードをちょっと修正して、レスポンスの詳細を `print()` してみよう！  

👇 **`test_login4.py` の `test_login_success` を修正**
```python
@pytest.mark.asyncio
async def test_login_success():
    async with httpx.AsyncClient(base_url="https://127.0.0.1:8000", verify=ssl_context, follow_redirects=True) as client:
        
        # 1️⃣ Cookie なしで `/` にアクセス → `/login` にリダイレクトされるべき
        response = await client.get("/")
        assert response.status_code == 200  # `login.html` を受け取る

        # 2️⃣ `/login` に正しい認証情報を送信
        response = await client.post("/login", data={"username": "user1", "password": "user1"})
        
        # 🔥 デバッグ用にレスポンス詳細を表示
        print("\n🔍 DEBUG: /login response")
        print(f"Status Code: {response.status_code}")
        print(f"Headers: {response.headers}")
        print(f"Body: {response.text}")

        assert response.status_code == 303  # リダイレクトが発生
        assert "set-cookie" in response.headers  # Cookie が設定されているか確認
```
✨ **これで `/login` のレスポンスが `200 OK` になっている理由が分かるはず！**  

---

### **✅ ② FastAPI の `/login` のコードをチェック**
FastAPI 側の `/login` エンドポイント (`main.py`) で `303` を返す処理があるか確認しよう！  

👇 **FastAPI の `/login` のコード**
```python
@app.post("/login", response_class=HTMLResponse)
async def login_post(request: Request, response: Response,
    form_data: OAuth2PasswordRequestForm = Depends()):
    try:
        username = form_data.username
        password = form_data.password

        user = await authenticate_user(request, username, password) 
        if user is None:
            raise HTTPException(status_code=400, detail="ログインに失敗しました。")

        # 正常ログイン時のリダイレクトURL
        redirect_url = {1: "/order_complete", 2: "/manager/today", 10: "/shops/today", 99: "/admin/today"}.get(user.get_permission(), "/error")

        response = RedirectResponse(url=redirect_url, status_code=303)  # 🔥 ここで 303 を返す！
        set_all_cookies(response, {
            'sub': user.get_username(),
            'token': user.get_token(),
            'exp': user.get_exp(),
            'permission': user.get_permission()
        })

        return response  # 🔥 ここがちゃんと動いてるか確認！

    except HTTPException as e:
        print(f"/login_post HTTPエラー: {e.detail}")
        encoded_message = urllib.parse.quote(e.detail)
        return RedirectResponse(url=f"/login?message={encoded_message}", status_code=303)

    except Exception as e:
        print(f"/login 予期せぬエラー: {e}")
        encoded_message = urllib.parse.quote("予期せぬエラーが発生しました。")
        return RedirectResponse(url=f"/login?message={encoded_message}", status_code=303)
```
### **👀 確認ポイント**
1. `await authenticate_user(request, username, password)` が **`None` を返していないか**
   - **ログイン失敗すると `400` になるから、`303` じゃなく `200 OK` になってる可能性がある！**
2. `RedirectResponse(url=redirect_url, status_code=303)` が**実行されているか**
3. `set_all_cookies(response, {...})` で **Cookie がセットされているか**

---

### **✅ ③ `authenticate_user` の動作確認**
`await authenticate_user(request, username, password)` が `None` を返してると、`303` にならないよ！  

👇 **テスト用に `print()` を追加して、どんな値が返ってくるか確認！**
```python
async def authenticate_user(request: Request, username, password) -> Optional[User]:
    try:
        user = await select_user(username)
        print(f"\n🔍 DEBUG: authenticate_user")
        print(f"user: {user}")

        if user is None:
            print("❌ ユーザーが見つからない！")
            return None

        if user.get_password() != password:
            print("❌ パスワードが違う！")
            return None

        return user  # ✅ ここで `user` が返れば成功！

    except Exception as e:
        print(f"認証エラー: {e}")
        return None
```
✨ **これで `user` が正しく取得できているか確認できる！**

---

## **🚀 まとめ**
✅ **`test_login_success` に `print()` を追加して、レスポンスの `status_code` を確認！**  
✅ **FastAPI 側の `/login` の `RedirectResponse(status_code=303)` がちゃんと動いているか確認！**  
✅ **`authenticate_user` の `user` が `None` になってないかチェック！**  

**まずは `/login` のレスポンス詳細を `print()` してみて、どこで `303` が `200` に変わってるか見てみよう！**  
それでまた報告してくれたら、もっと詳しく調査できるよ！🔥💖

</details>

<details><summary>staticフォルダを認識しない</summary>

#### Q.
この赤文字エラーは何でしょうか？
ImportError while loading conftest 'C:\Obento-Test\v_0.1.3\app\tests\conftest.py'.
app\tests\conftest.py:10: in <module>
    from main import app  # これで main.py を正しくインポートできる
app\main.py:355: in <module>
    app.mount("/static", StaticFiles(directory="static"), name="static")
C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\site-packages\starlette\staticfiles.py:56: in __init__
    raise RuntimeError(f"Directory '{directory}' does not exist")
E   RuntimeError: Directory 'static' does not exist

#### A.
おっけー！✨ **`RuntimeError: Directory 'static' does not exist`** のエラーが出てるね！💡  

---

## **🔥 エラーの原因**
```
RuntimeError: Directory 'static' does not exist
```
これは **FastAPI の `StaticFiles` のマウント時に `static` フォルダが見つからない** ってエラーだよ！💦  

**エラーが出た行（`main.py:355`）を確認すると、`app.mount("/static", StaticFiles(directory="static"), name="static")` の部分で発生してる！**  

つまり、`static` フォルダが存在しない or pytest 実行時のカレントディレクトリが違うせいで認識できていない可能性があるよ！💡

---

## **✅ 解決策① `static` フォルダを作る**
まず、プロジェクトのルート (`C:\Obento-Test\v_0.1.3\app\`) に **`static` フォルダがあるか確認** してみよう！  

### **💡 Windows のコマンドプロンプトで確認**
```bash
dir C:\Obento-Test\v_0.1.3\app\
```
もし `static` フォルダが無かったら、作成しよう！  

```bash
mkdir C:\Obento-Test\v_0.1.3\app\static
```
💡 **この方法で `static` フォルダがあれば、エラーは解決するはず！** 🚀✨  

---

## **✅ 解決策② `StaticFiles` のパスを修正**
もし `static` フォルダが **`app/static/` の中にあるなら、FastAPI の `StaticFiles` のパスを修正** しよう！  

👇 **`main.py` の `app.mount` を修正**
```python
import os
from fastapi.staticfiles import StaticFiles

static_path = os.path.join(os.path.dirname(__file__), "static")  # 絶対パスに変換
app.mount("/static", StaticFiles(directory=static_path), name="static")
```
💡 **この方法なら、pytest 実行時も `static` フォルダを正しく認識できる！**  

---

## **✅ 解決策③ `pytest` のカレントディレクトリを調整**
もし `pytest` の実行ディレクトリが `app/tests/` になっていると、FastAPI が `static` を見つけられないことがあるよ💡  

👇 **正しい実行方法**
```bash
cd C:\Obento-Test\v_0.1.3\
pytest app/tests
```
💡 **これで FastAPI が `static` フォルダを正しく認識できるはず！** 🎯  

---

## **🚀 まとめ**
✅ **`static` フォルダが `C:\Obento-Test\v_0.1.3\app\static\` にあるか確認！**
✅ **`app.mount("/static", StaticFiles(directory="static"), name="static")` を絶対パスに変更！**
✅ **`pytest` は `app/` の外で実行！（`cd C:\Obento-Test\v_0.1.3\ && pytest app/tests`）**

✨ **これで `RuntimeError: Directory 'static' does not exist` は解決するはず！**  
試してみて、まだ問題があったらまた教えてね〜！🔥💖

</details>

<details><summary>importエラーになる</summary>

#### Q.
from main import app　conftest.pyの5行目です。
このエラーになります。
ImportError while loading conftest 'C:\Obento-Test\v_0.1.3\app\tests\conftest.py'.
app\tests\conftest.py:5: in <module>
    from ..main import app
E   ImportError: attempted relative import with no known parent package

#### A.
おっけー！✨ **`conftest.py` の `from ..main import app` で `ImportError` が発生** してるね！💡  

これは **pytest の実行方法** や **import の書き方** が原因だから、解決策を紹介するよ！🎯  

---

## **🔥 `ImportError` の原因**
```
ImportError: attempted relative import with no known parent package
```
このエラーは **`conftest.py` から `main.py` をインポートしようとしたけど、`app` がパッケージとして認識されていない** から発生するよ！💦  

---

## **✅ 解決策① `sys.path` を使う（おすすめ！）**
**`conftest.py` の最初に `sys.path` を追加** すると、どこで `pytest` を実行しても `main.py` を正しくインポートできる！✨  

👇 **`app/tests/conftest.py` を修正**
```python
import sys
import os

# `app/` を `sys.path` に追加（Python に `app/` を認識させる）
sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "..")))

from main import app  # これで `main.py` を正しくインポートできる
```
💡 **これを入れると、`pytest` の実行場所を気にせず `main.py` をインポートできる！**

---

## **✅ 解決策② `pytest` を `app/` の外で実行する**
`pytest` は **カレントディレクトリによってパッケージの認識が変わる** から、  
**必ず `app/` の外で実行する！**  

👇 **正しい実行方法**
```bash
cd C:\Obento-Test\v_0.1.3\
pytest app/tests
```
💡 **これで `app` が Python のパッケージとして認識される！** 🎯  

---

## **✅ 解決策③ `pytest` に `PYTHONPATH` を設定する**
もし上の `sys.path` の方法を使いたくない場合、  
`pytest` 実行時に `PYTHONPATH` を指定すれば、`app/` を認識させることができるよ！  

👇 **Windows のコマンドプロンプトで実行**
```bash
set PYTHONPATH=C:\Obento-Test\v_0.1.3\app
pytest app/tests
```
💡 **環境変数 `PYTHONPATH` に `app/` をセットすることで、`import main` が通る！**

---

## **🚀 まとめ**
✅ **`sys.path` を `conftest.py` に追加**（おすすめ！）  
✅ **`pytest` を `app/` の外で実行**  
✅ **`PYTHONPATH` を設定して `pytest` を実行**  

✨ **このどれかで `ImportError` は解決するはず！** 試してみて、まだ問題あったら教えてね〜！🔥💖

</details>

<details><summary>上記のコマンド入力を、Windowsのショートカットのリンク先（T）に登録したいです</summary>

#### Q.
cd C:\Obento-Test\v_0.1.3\
pytest app/tests
上記のコマンド入力を、Windowsのショートカットのリンク先（T）に登録したいです。

#### A.
おっけー！✨ **Windows のショートカットを作って、クリックするだけで `pytest` を実行できるようにする方法** を教えるね！💡🎯  

---

## **✅ ① Windows のショートカットを作成**
1. **デスクトップ or 任意のフォルダで「新しいショートカット」を作成**
   - 何もないところで **右クリック →「新規作成」→「ショートカット」** を選択  

2. **リンク先（T）に以下を入力**
   ```bash
   cmd /k "cd /d C:\Obento-Test\v_0.1.3\ && pytest app/tests"
   ```
   💡 **ポイント！**
   - `cmd /k` → **コマンドプロンプトを開いたままにする**
   - `cd /d` → **ドライブが違っても `C:\Obento-Test\v_0.1.3\` に移動できる**
   - `pytest app/tests` → **`app/tests` 内のテストを実行！**

3. **ショートカットの名前を「pytest 実行」にする（自由に変更OK！）**  
4. **「完了」をクリック！** 🎉  

---

## **✅ ② ショートカットをカスタマイズ（オプション）**
### **💡 アイコンを変更**
1. ショートカットを **右クリック →「プロパティ」**  
2. 「ショートカット」タブの「アイコンの変更」をクリック  
3. 好きなアイコンを選ぶ！（例えば `C:\Windows\System32\shell32.dll` の中にアイコンがあるよ！）

---

## **✅ ③ ショートカットを実行！**
作成したショートカットを **ダブルクリックすると、`pytest` が自動で実行される！** 🚀✨  

---

## **🎯 まとめ**
✅ **ショートカットを作成して、`cmd /k "cd /d C:\Obento-Test\v_0.1.3\ && pytest app/tests"` を登録！**  
✅ **ダブルクリックするだけで `pytest` が実行できる！** 🎉  
✅ **アイコンも変更できるので、カスタマイズしよう！**  

これで `pytest` を **ワンクリックで実行できるようになったね！**🔥😆  
試してみて、問題があったらまた聞いてね〜！💖✨

</details>

<details><summary>pytest は app/ の外で実行！とは？</summary>

#### Q.
pytest は app/ の外で実行！
これはtestsフォルダをappフォルダと同じ階層に配置するという意味ですか？

#### A.
ナイス質問！✨ **`pytest` は `app/` の外で実行** っていうのは、  
**「`tests/` フォルダを `app/` の外に移動する」って意味じゃないよ！** 😆  

**`tests/` フォルダは `app/` の中のままでOK！**  
ただし、**pytest を実行するときのカレントディレクトリを `app/` の外にする** ってことだよ！💡  

---

## **✅ `tests/` の正しい配置**
**今の `app/` の構成はこんな感じだよね？**
```
C:\Obento-Test\v_0.1.3\
 ├── app/
 │    ├── main.py  ← FastAPI のエントリーポイント
 │    ├── utils/
 │    │    ├── exception.py
 │    ├── tests/  ← `tests/` はこのままでOK！
 │    │    ├── conftest.py
 │    │    ├── test_login4.py
```
**✨ `tests/` は `app/` の中に置いたままでOK！移動しなくていいよ！** 👍  

---

## **✅ `pytest` の実行方法**
### **❌ 間違った実行方法（`tests/` の中で `pytest` 実行）**
```bash
cd C:\Obento-Test\v_0.1.3\app\tests
pytest
```
🚨 **これをやると、`main.py` が正しくインポートできず `ImportError` になる！**  

---

### **✅ 正しい実行方法（`app/` の外で `pytest` を実行）**
```bash
cd C:\Obento-Test\v_0.1.3\
pytest app/tests
```
💡 **この方法なら `app/` を Python のパッケージとして認識できるから、エラーにならない！** 🎉  

---

## **✅ `sys.path` を使う方法（どこで実行してもOKにする）**
もし `pytest` をどこで実行しても大丈夫なようにしたいなら、  
`conftest.py` の最初に **`sys.path` を追加** する方法もあるよ！👇  

```python
import sys
import os

# `app/` を `sys.path` に追加（Python に `app/` を認識させる）
sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "..")))

from main import app  # これで `main.py` を正しくインポートできる
```
💡 **このコードを入れると、`pytest` を `tests/` の中で実行しても `main.py` をインポートできる！**  

---

## **🚀 まとめ**
✅ **`tests/` フォルダは `app/` の中のままでOK！移動しなくていい！**  
✅ **`pytest` は `app/` の外 (`C:\Obento-Test\v_0.1.3\`) で実行する！**  
✅ **どこで実行しても大丈夫にしたいなら `sys.path` を `conftest.py` に追加する！**  

これで `pytest` がバッチリ動くはず！✨  
試してみて、まだ問題があったらまた教えてね〜！🔥💖

</details>

<details><summary>main.py から FastAPI アプリをインポート</summary>

#### Q.
from main import app
# ← main.py から FastAPI アプリをインポート 　　conftest.pyに降格と赤文字エラーになりました。

ImportError while loading conftest 'C:\Obento-Test\v_0.1.3\app\tests\conftest.py'.
conftest.py:5: in <module>
    from ..main import app
E   ImportError: attempted relative import with no known parent package

どうすればよいですか？

#### A.
おっけー！✨ `conftest.py` で `from main import app` をインポートしようとしたら、  
`ImportError: attempted relative import with no known parent package` ってエラーが出たんだね！💡  

これは **pytest の実行方法の問題** だから、解決策を紹介するよ！✨  

---

## **✅ 解決策 ① `pytest` を `app/` の外で実行する**
今、`pytest` を `app/tests/` の中で実行してないかな？  
Python の **パッケージのルート（`app/`）の外で実行** しないと `main.py` を正しくインポートできないんだ💡  

### **💡 こうするとエラーになる**
```bash
cd C:\Obento-Test\v_0.1.3\app\tests
pytest
```
⬆️ **`tests/` の中で実行すると、`main.py` を `import` できないからエラーになる！** 🚨  

---

### **🔥 `app/` の外で実行しよう！**
```bash
cd C:\Obento-Test\v_0.1.3\
pytest app/tests
```
✨ **`app/tests` を指定して実行すると、エラーが出ずにテストが動くはず！**  

---

## **✅ 解決策 ② `sys.path` を使って `app/` をパスに追加**
もし **どうしても `pytest` を `tests/` の中で実行したいなら、`sys.path` を使う方法もある！**  

### **`conftest.py` の最初にこのコードを追加**
```python
import sys
import os

# `app/` を `sys.path` に追加（Python に `app/` を認識させる）
sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), "..")))

from main import app  # これで `main.py` を正しくインポートできる
```
✨ **このコードを入れると、`pytest` をどこで実行しても `main.py` を正しくインポートできる！**

---

## **🚀 まとめ**
✅ **`pytest` は `app/` の外で実行！**
```bash
cd C:\Obento-Test\v_0.1.3\
pytest app/tests
```
✅ **どうしても `tests/` の中で実行するなら、`sys.path` を追加する！**

これで `pytest` もバッチリ動くはず！✨  
試してみて、まだエラー出たら教えてね〜！🔥💖

</details>

<details><summary>./my-local.key`（秘密鍵ファイル）が見つからない</summary>

#### Q.
ImportError while loading conftest 'C:\Obento-Test\v_0.1.3\app\tests\conftest.py'.
conftest.py:4: in <module>
    from main import app  # ← main.py から FastAPI アプリをインポート
..\main.py:13: in <module>
    from local_jwt_module import SECRET_KEY, get_new_token, check_cookie_token
..\local_jwt_module.py:28: in <module>
    private_key = load_private_key("./my-local.key")
..\local_jwt_module.py:24: in load_private_key
    with open(key_file, "rb") as key_file:
E   FileNotFoundError: [Errno 2] No such file or directory: './my-local.key'
こんなエラーが赤文字で出ました。どうすればいいですか？

#### A.
おっけー！✨ エラーメッセージを解析すると、  
**`conftest.py` を読み込もうとしたときに `local_jwt_module.py` 内の `load_private_key` でエラーが発生** してるね！💡  

---

## **🔥 エラーの原因**
```
FileNotFoundError: [Errno 2] No such file or directory: './my-local.key'
```
このエラーは **`./my-local.key`（秘密鍵ファイル）が見つからない** ってことを意味してるよ！  

- `local_jwt_module.py` の `load_private_key("./my-local.key")` が、  
  **テスト実行時に `my-local.key` を開こうとして、見つからずにエラーになってる** んだね💦  
- `main.py` を普通に実行すると問題ないのに、`pytest` 実行時にエラーになる理由は、  
  **pytest の実行ディレクトリが `app/tests/` になっているから！**  

---

## **✅ 解決策**
### **① `my-local.key` の正しいパスを指定する**
**テスト時にも確実に `my-local.key` を読み込めるように、絶対パスを使うように修正！**

👇 **`local_jwt_module.py` の `load_private_key` を以下のように変更**
```python
import os

def load_private_key(key_file: str):
    key_path = os.path.abspath(os.path.join(os.path.dirname(__file__), key_file))  # 絶対パスに変換
    with open(key_path, "rb") as key_file:
        return key_file.read()
```
💡 **この修正で、`my-local.key` のパスが確実に解決される！**

---

### **② `my-local.key` が本当に存在するか確認**
テスト時に `my-local.key` が存在しない場合は、エラーになるよ！  
**まず `my-local.key` が `app/` にあるか確認してね！**

```bash
ls -l C:\Obento-Test\v_0.1.3\app\
```
（Windowsなら `dir C:\Obento-Test\v_0.1.3\app\`）

もし **ファイルが無いなら、作成するか、テスト用のダミーファイルを使う方法もあるよ！**  

---

### **③ `conftest.py` で `my-local.key` があるか確認**
テストの前に `my-local.key` があるか確認するコードを `conftest.py` に追加！👇

```python
import os
import pytest

@pytest.fixture(scope="session", autouse=True)
def check_private_key():
    key_path = os.path.abspath(os.path.join(os.path.dirname(__file__), "../my-local.key"))
    
    if not os.path.exists(key_path):
        pytest.exit(f"❌ my-local.key が見つかりません！パスを確認してください: {key_path}")
    
    print(f"✅ my-local.key が見つかりました: {key_path}")
```
💡 **これで `my-local.key` が無かったら pytest 実行前にエラーメッセージを出せる！**

---

## **🚀 まとめ**
1. **`local_jwt_module.py` の `load_private_key` を絶対パスに変更**
2. **`my-local.key` が `app/` にあるかチェック**
3. **`conftest.py` に `check_private_key()` を追加して、テスト前に `my-local.key` の存在を確認！**

✨ これで `pytest` でも `my-local.key` を正しく読み込めるはず！  
試してみて、まだエラー出るなら教えてね〜！🔥💖

</details>

<details><summary>conf_test.pyはどこで読みますか？</summary>

#### Q.
conf_test.pyはどこで読みますか？

#### A.
ナイス質問！✨ `conftest.py` は **pytest が自動的に認識して読み込む設定ファイル** だから、  
特別に `import` する必要はなく、**適切な場所に置いておけば pytest が勝手に読み込んでくれる** よ！💡  

---

## **✅ `conftest.py` の置き場所**
`conftest.py` は **テストファイル (`test_*.py`) があるディレクトリに配置** すれば OK！  

👇 例えば、`test_login4.py` があるディレクトリがこんな感じなら…  
```
app/
 ├── main.py  ← FastAPI のエントリーポイント
 ├── utils/
 │    ├── exception.py
 │    ├── __init__.py
 ├── tests/
 │    ├── conftest.py  👈 ここに置く！
 │    ├── test_login4.py
```
✨ **この `tests/conftest.py` は `pytest` が自動的に読み込むから、特に `import` しなくてもOK！**  

---

## **✅ ちゃんと `conftest.py` が読み込まれているか確認**
テストを実行するときに `pytest -v` で詳細ログを出して、**`conftest.py` が読み込まれているか確認** できるよ👇  

```bash
pytest -v
```

もし `conftest.py` が読み込まれていない場合は、  
```bash
pytest --trace-config
```
を実行すると、**どの設定ファイル (`conftest.py`) が読み込まれているか** が分かるよ！💡  

---

## **🚀 まとめ**
1. **`conftest.py` は `test_*.py` があるディレクトリに置くだけで自動で読み込まれる！**
2. **`import` は不要！pytest が勝手に認識してくれる**
3. **ちゃんと読み込まれているか確認するなら `pytest -v` や `pytest --trace-config` を実行**

✨ **これで `pytest` で FastAPI をテストする準備はバッチリ！**  
試してみて、まだ問題があったら詳細教えてね〜！🔥💖

</details>

<details><summary>pytest でテスト実行時に FastAPI のサーバーが動いていないのが原因</summary>

#### Q.
pytestでテストを実施しました。しかしこのようなログのエラーになります。何かわかりますか？
================================================= test session starts =================================================
platform win32 -- Python 3.12.8, pytest-8.3.4, pluggy-1.5.0
rootdir: C:\Obento-Test\v_0.1.3\app
configfile: pytest.ini
plugins: anyio-4.4.0, asyncio-0.25.3
asyncio: mode=Mode.AUTO, asyncio_default_fixture_loop_scope=function
collected 2 items

test_login4.py::test_login_success FAILED                                                                        [ 50%]
test_login4.py::test_login_failure FAILED                                                                        [100%]

====================================================== FAILURES =======================================================
_________________________________________________ test_login_success __________________________________________________

    @contextlib.contextmanager
    def map_httpcore_exceptions() -> typing.Iterator[None]:
        global HTTPCORE_EXC_MAP
        if len(HTTPCORE_EXC_MAP) == 0:
            HTTPCORE_EXC_MAP = _load_httpcore_exceptions()
        try:
>           yield

C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\site-packages\httpx\_transports\default.py:101:
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\site-packages\httpx\_transports\default.py:394: in handle_async_request
    resp = await self._pool.handle_async_request(req)
C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\site-packages\httpcore\_async\connection_pool.py:256: in handle_async_request
    raise exc from None
C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\site-packages\httpcore\_async\connection_pool.py:236: in handle_async_request
    response = await connection.handle_async_request(
C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\site-packages\httpcore\_async\connection.py:101: in handle_async_request
    raise exc
C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\site-packages\httpcore\_async\connection.py:78: in handle_async_request
    stream = await self._connect(request)
C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\site-packages\httpcore\_async\connection.py:124: in _connect
    stream = await self._network_backend.connect_tcp(**kwargs)
C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\site-packages\httpcore\_backends\auto.py:31: in connect_tcp
    return await self._backend.connect_tcp(
C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\site-packages\httpcore\_backends\anyio.py:113: in connect_tcp
    with map_exceptions(exc_map):
C:\Users\tensy\AppData\Local\Programs\Python\Python312\Lib\contextlib.py:158: in __exit__
    self.gen.throw(value)
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

#### A.
おっけー！✨ `test_login4.py` のコードを確認したよ！  
エラーメッセージとコードの内容からすると、**pytest でテスト実行時に FastAPI のサーバーが動いていないのが原因** っぽい！💡  

---

## **🔥 エラーの原因**
```
httpcore._backends.anyio.py:113: in connect_tcp
ConnectionRefusedError: [WinError 10061] 
ターゲット コンピューターによって拒否されたため、接続できませんでした。
```
このエラーは **テスト時に `httpx.AsyncClient` が `127.0.0.1:8000` に接続しようとしたけど、FastAPI のサーバーが動いていなかった** から起こるよ！💦  

---

## **✅ 解決策**
### **① テスト実行前に FastAPI のサーバーを起動**
まず、FastAPI のサーバーが **動いているかチェック！**  

```bash
netstat -ano | findstr :8000
```
💡 **ポート `8000` が使われていなければ、FastAPI を起動する！**
```bash
uvicorn main:app --reload
```
✨ **サーバーを動かした状態で `pytest` を実行すれば、テストが通るはず！**  

---

### **② pytest で FastAPI を自動起動する**
テストごとに手動で `uvicorn` を起動するのが面倒なら、pytest のフィクスチャで **FastAPI をテスト用に起動するように設定** しよう！  

👇 **`conftest.py` を作成（または修正）**
```python
import pytest
import asyncio
from fastapi.testclient import TestClient
from main import app  # ← `main.py` から FastAPI アプリをインポート
import uvicorn

@pytest.fixture(scope="module")
def test_app():
    # テスト用クライアントを作成
    client = TestClient(app)
    return client

@pytest.fixture(scope="module", autouse=True)
def run_server():
    """ テスト実行前にサーバーをバックグラウンドで起動 """
    config = uvicorn.Config(app, host="127.0.0.1", port=8000, log_level="info")
    server = uvicorn.Server(config)
    
    loop = asyncio.get_event_loop()
    task = loop.create_task(server.serve())
    
    yield  # テスト実行
    
    task.cancel()  # テスト後にサーバーを停止
```
✨ **この設定をすれば、pytest 実行時に FastAPI のサーバーが自動で起動＆停止される！**

---

### **③ `test_login4.py` の修正**
上の `conftest.py` を作ったら、**`test_login4.py` の `base_url` を `http://127.0.0.1:8000` にする！**  

```python
async with httpx.AsyncClient(base_url="http://127.0.0.1:8000", verify=False, follow_redirects=True) as client:
```
💡 `verify=False` にすることで、ローカル環境での証明書エラーを回避できるよ！

---

## **🚀 まとめ**
1. **手動で `uvicorn` を起動してテストする**
   ```bash
   uvicorn main:app --reload
   pytest
   ```
2. **`conftest.py` を作って、pytest 実行時に FastAPI を自動起動**
3. **`test_login4.py` の `base_url` を `http://127.0.0.1:8000` に変更して、`verify=False` にする**

---

✨ **これで `pytest` で FastAPI を正しくテストできるはず！**  
試してみて、まだエラーが出たら詳細教えてね〜！🔥💖

</details>

<details><summary>この例外raise およびcatchを正しくテストするには、assertはどのように記述すればよいですか？</summary>

#### Q.
現在はこのカスタム例外をraiseすると、これ専用のハンドラーでキャッチしています。
class CustomException(HTTPException):
    def __init__(self, status_code: int, detail: str):
        print(f"🚨 CustomException 発生！ status_code={status_code}, message={detail}")  # 追加
        super().__init__(status_code=status_code, detail=detail)

@app.exception_handler(CustomException)
async def custom_exception_handler(
    request: Request, exc: CustomException):
    print(f"例外ハンドラーが呼ばれました: {exc.detail}")  # デバッグ用
    return templates.TemplateResponse(
        "error.html",
        {"request": request, "message": exc.detail},
        status_code=exc.status_code
    )

上記の例外ハンドラーで、error.htmlで遷移しています。
この例外raise およびcatchを正しくテストするには、assertはどのように記述すればよいですか？

#### A.
カスタム例外 (`CustomException`) の `raise` および `catch` をテストするには、以下の2つの点をチェックすればOKです。

1. **レスポンスのステータスコードが `exc.status_code` になっているか**
2. **レスポンスの内容が `error.html` のテンプレートで正しくレンダリングされているか**

---

### **1. `pytest` でカスタム例外をテストする方法**
#### **テストコード (`test_custom_exception.py`)**
```python
import pytest
from httpx import AsyncClient
from main import app  # FastAPIアプリ
import ssl

# SSL設定（テスト環境用）
ssl_context = ssl.create_default_context()
ssl_context.check_hostname = False
ssl_context.verify_mode = ssl.CERT_NONE


@pytest.mark.asyncio
async def test_custom_exception():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/test_exception")  # 例外が発生するエンドポイント

        # 1️⃣ ステータスコードのチェック
        assert response.status_code == 400  # `CustomException(400, "これはテストエラーです")` の通り

        # 2️⃣ レスポンスのHTMLテンプレートをチェック
        assert "これはテストエラーです" in response.text  # error.html でエラーメッセージが表示されているか
```

---

### **2. 例外を発生させるエンドポイント**
現在の `main.py` には、例外を発生させるエンドポイント `/test_exception` があります:
```python
@app.get("/test_exception")
async def test_exception():
    raise CustomException(400, "これはテストエラーです")
```
このエンドポイントを使って、カスタム例外の動作をテストできます。

---

### **3. `pytest` の実行**
ターミナルで以下のコマンドを実行:
```sh
pytest test_custom_exception.py
```

✅ **テストが成功すると、 `400` のステータスコードと `error.html` に "これはテストエラーです" が含まれることを確認できます。**

---

### **補足**
- `error.html` にテンプレート変数 `{message}` があるなら、レスポンスの `response.text` に `message` の内容が含まれていることを確認すればOKです。
- **リダイレクト (303) を期待する場合** は、`response.status_code == 303` に変更すれば、 `/login` などへのリダイレクトもテストできます。

---

この方法で、カスタム例外の `raise` と `catch` が正しく動作しているかを確認できます！ 🚀

</details>

<details><summary>テスト実施計画書を作っています。テストを他人にやってもらうための書類はなんといいますか？</summary>

#### Q.
テスト実施計画書を作っています。テストを他人にやってもらうための書類はなんといいますか？

#### A.
テストを他人に実施してもらうための書類は、一般的に以下のように呼ばれます。

1. **テスト実施手順書（Test Execution Procedure）**  
   - テストをどのように実施するかを詳細に記載した手順書です。  
   - テストの手順、前提条件、期待結果などを明確に記述します。

2. **テスト仕様書（Test Specification）**  
   - どのようなテストケースを実施するのかを定義した文書です。  
   - テストケースごとの入力値、期待結果、実施方法などが記載されます。

3. **テスト指示書（Test Instructions）**  
   - テスト担当者に対して具体的なテスト実施の指示を与える書類です。  
   - 期間、対象範囲、使用環境、注意点などが記載されることが多いです。

4. **テスト手順書（Test Procedure）**  
   - テストを実施する際の詳細な手順を説明する文書です。  
   - 初心者でも迷わず実施できるよう、具体的な操作手順を記述します。

どの書類を用意するかは、テストの規模や目的によりますが、実施者がスムーズにテストできるように **「テスト実施手順書」** や **「テスト指示書」** を作成するのが一般的です。

</details>

<details><summary>テスターがテスト指示書やテスト手順書に従ってテストを実施する場合、その結果を記録する書類を何と言いますか？</summary>

#### Q.
テスターがテスト指示書やテスト手順書に従ってテストを実施する場合、その結果を記録する書類を何と言いますか？

#### A.
テスターがテストを実施し、その結果を記録する書類は、一般的に以下のように呼ばれます。

1. **テスト結果報告書（Test Report）**  
   - テスト全体の結果をまとめた報告書。  
   - 実施したテストケースの成功・失敗状況、発生した不具合、考察、改善点などを記載します。

2. **テスト結果記録書（Test Result Record）**  
   - 実施したテストごとの結果を記録する書類。  
   - 各テストケースの実施状況（合格/不合格）、実行日時、担当者、不具合の有無などを記録します。

3. **テスト実施記録（Test Execution Log）**  
   - テストの実施履歴を詳細に記録する文書。  
   - 実施した手順や環境、異常が発生した際の状況などを記録します。

4. **テスト結果シート（Test Result Sheet）**  
   - テストケースごとの実施結果を表形式でまとめたもの。  
   - Excelなどで管理されることが多く、テストの進捗管理にも使われます。

5. **バグ報告書（Bug Report）**  
   - テスト中に発見された不具合を記録する文書。  
   - 不具合の発生条件、再現手順、影響範囲、スクリーンショットなどを記載します。

状況に応じて、**「テスト結果記録書」や「テスト結果シート」** に詳細なデータを記録し、最終的に**「テスト結果報告書」** にまとめて報告する形が一般的です。

</details>

<details><summary>テスト実施記録は、むかしはExcelシートで記録していました。現代のWebアプリの製造者の多くは何を使って、テスト実施記録をしていますか？マークダウンファイルですか？</summary>

#### Q.
テスト実施記録は、むかしはExcelシートで記録していました。現代のWebアプリの製造者の多くは何を使って、テスト実施記録をしていますか？マークダウンファイルですか？

#### A.
現代のWebアプリの製造者が**テスト実施記録**を管理する際に使うツールは、プロジェクトの規模や開発手法によって異なります。Excelの代わりに、以下のようなツールやフォーマットがよく利用されています。

### **1. テスト管理ツール（専用ツール）**
✅ **TestRail**  
✅ **Xray（Jiraプラグイン）**  
✅ **Zephyr（Jiraプラグイン）**  
✅ **qTest**  
✅ **PractiTest**  

- これらは専用のテスト管理ツールで、**テストケースの作成、実施状況の記録、バグ管理、進捗管理** を一元化できます。
- **Jira** や **GitHub** などと連携し、開発とQAがスムーズに連携可能。

---

### **2. チケット管理ツール（JiraやGitHub Issuesを活用）**
✅ **Jira**（テスト管理プラグイン + カスタムフィールド）  
✅ **GitHub Projects + Issues**  
✅ **Azure DevOps（Test Plans機能）**  

- テストケースごとに「チケット（Issue）」を作成し、実施結果をコメントで記録。
- **手動テスト** も **自動テストの結果** も同じプラットフォームで管理可能。

---

### **3. マークダウン（Markdownファイル）**
✅ **GitHub / GitLab のリポジトリで `test_results.md` などを管理**  
✅ **Notion や Obsidian で Markdown ベースの記録**  
✅ **内部Wiki（Confluence, Notion）で Markdown を活用**  

- 軽量でシンプルな記録方法。
- **コードレビューのようにPRでテスト結果を管理** できるため、開発とQAの連携がスムーズ。
- ただし、**大規模なプロジェクトでは管理が煩雑になりがち**。

---

### **4. スプレッドシート（Google Sheets / Excel Online）**
✅ **Google Sheets + スクリプト連携**  
✅ **Excel Online + Power Automate で自動化**  

- 依然として **小規模・中規模プロジェクトでは人気**。
- **マクロやスクリプトで自動化** すれば、従来のExcelより効率的。

---

### **5. CI/CD パイプラインと連携したテスト結果管理**
✅ **Jenkins + Test Reports**  
✅ **GitHub Actions + Allure Reports**  
✅ **CircleCI / GitLab CI/CD + レポート機能**  

- **自動テストの結果** をCI/CDパイプラインで管理し、手動テストの結果と統合。
- **Allure, Extent Reports** などのレポートツールと連携するケースが多い。

---

### **結論：現代の主流は？**
✅ **小規模 → Markdown, Notion, Google Sheets**  
✅ **中規模 → Jira + Xray / Zephyr, GitHub Issues**  
✅ **大規模 → TestRail, qTest, Azure DevOps**  
✅ **自動テストが中心 → CI/CD + Allure Reports**  

マークダウンファイルを使うケースもありますが、**JiraやTestRailのような専用ツールを使う企業が多い** です。  
特にアジャイルやDevOps環境では、**チケット管理ツールやCI/CDと連携** できるツールのほうが好まれています。

</details>

<details><summary>markdownでテスト実施記録書を作る</summary>

#### Q.
markdownでテスト実施記録書を作る方法はありますか？

#### A.
### **Markdownでテスト実施記録書を作る方法**
Markdownを使ってテスト実施記録書を作成することは可能です。シンプルな構造で管理しやすく、GitHub/GitLabと連携してバージョン管理もできるため、小規模・中規模の開発チームに適しています。

---

## **1. 基本フォーマット（Markdown）**
以下は、シンプルな **テスト実施記録書** のMarkdownフォーマット例です。

```md
# テスト実施記録書

## 基本情報
- **プロジェクト名**: Sample Web App
- **テスト実施日**: 2025-03-04
- **テスト担当者**: 田中 太郎
- **テスト環境**: Chrome 120.0 / Windows 11
- **テスト対象バージョン**: v1.2.3

---

## **テストケース一覧**

| No. | テストケース | 入力値 | 期待結果 | 実施結果 | 判定 | 備考 |
|----|------------|------|--------|--------|----|------|
| 1  | ログイン機能が正常に動作する | test@example.com / password123 | ダッシュボード画面へ遷移 | OK | ✅ |  |
| 2  | 無効なパスワードでログイン | test@example.com / wrongpass | エラーメッセージ表示 | OK | ✅ |  |
| 3  | パスワードリセット | test@example.com | メール送信成功 | NG | ❌ | メール未送信 |

---

## **不具合一覧（バグレポート）**

### **#001 メール未送信バグ**
- **発生個所**: パスワードリセット機能
- **発生条件**: `test@example.com` でリセット申請
- **期待結果**: リセットメールが送信される
- **実際の結果**: メールが送信されない
- **再現手順**:
  1. ログインページへアクセス
  2. 「パスワードを忘れた場合」をクリック
  3. `test@example.com` を入力し送信
- **優先度**: 高
- **対応状況**: 修正中（2025-03-04）

---

## **テスト結果まとめ**
- **実施ケース数**: 3
- **成功ケース数**: 2
- **失敗ケース数**: 1
- **成功率**: `66.7%`
- **テスト評価**: ⚠️ 部分的にNGあり、修正対応必要
```

---

## **2. Markdownを活用したテスト管理のポイント**
- **GitHub/GitLabリポジトリに `test_results.md` として保存**  
  → Gitの履歴管理で「どのバージョンでどのテスト結果だったか」を追跡可能  
- **Pull Request (PR)でテスト結果をレビュー**  
  → テスト実施後、PRのコメントに記録して開発者と共有  
- **CI/CDパイプラインと連携（Allure ReportsやGitHub Actions）**  
  → 自動テストの結果をMarkdownで出力して記録可能  

---

## **3. GitHubやNotionと組み合わせた運用例**
✅ **GitHub Issueで管理** → Markdown形式でテスト実施記録をIssueに記述  
✅ **Notionで記録** → Markdownがそのまま使えるので、Wiki風に管理可能  
✅ **CI/CDレポートと連携** → 自動テストの結果をMarkdownで出力し、PRに添付  

---

### **結論**
Markdownでテスト実施記録書を作成するのは十分に可能です。  
ただし、大規模プロジェクトでは管理が煩雑になるため、**Jira + Xray や TestRail などのツールと組み合わせる** のが現実的です。  
小規模チームなら **GitHub + Markdown** で手軽に管理できるのが大きなメリットです！

</details>

<details><summary>test_results.md　でGitHubに保存すると何か起こりますか？

</summary>

#### Q.
test_results.md　でGitHubに保存すると何か起こりますか？

#### A.
### **`test_results.md` を GitHub に保存すると何が起こるか？**
GitHubに `test_results.md` を保存すると、以下のようなメリットや活用方法があります。

---

## **1. 履歴管理ができる（変更履歴の追跡）**
GitHubは **バージョン管理システム** なので、テスト結果の変更履歴を追跡できます。

✅ **どのバージョンの時点でどんなテスト結果だったのか** を確認できる  
✅ **過去のテスト実施記録と比較** し、進捗や改善状況を把握できる  
✅ **コミットログを活用** して、誰がいつテストを実施したか明確になる  

📌 **例**:  
Gitの履歴を見ると「2025-03-04 のテストではエラーが1件あったが、2025-03-06 には解消されている」といった情報がわかる。

```sh
git log test_results.md
```

---

## **2. Pull Request（PR）でレビュー可能**
GitHubでは、テスト実施後に **Pull Request (PR)** で `test_results.md` をアップデートし、  
開発チームとテスト結果を共有＆レビューできます。

✅ PRの **コメント機能** を使って、開発者とテスターがディスカッション  
✅ 修正が必要な場合、 **修正ブランチと紐づけて管理** できる  
✅ `test_results.md` をレビューしてからマージすることで、品質向上  

📌 **例:**  
1. `test_results.md` を更新して PR を作成
2. 開発者がPRの **コメント機能** で指摘・修正  
3. テスト結果に問題なければ **マージ & 承認**

```sh
git add test_results.md
git commit -m "Update test results for version 1.2.3"
git push origin feature/test-results
```

---

## **3. GitHub Actions で自動化（CI/CDとの連携）**
GitHub Actions を使うと、**自動テストの結果を `test_results.md` に書き出す** ことも可能です。

✅ CI/CDパイプライン（Jenkins, GitHub Actions, CircleCI）と連携し、**テスト結果を自動記録**  
✅ **Allure Reports や JUnit XMLをMarkdownに変換** して `test_results.md` に追加  
✅ 毎回手動でテスト結果を入力しなくても、自動化できる  

📌 **例:**  
GitHub Actions で自動テストの結果を `test_results.md` に記録するワークフロー

```yaml
name: Test Report Update

on:
  push:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Run Tests
        run: |
          pytest --junitxml=report.xml

      - name: Convert JUnit XML to Markdown
        run: |
          python scripts/convert_junit_to_md.py report.xml > test_results.md

      - name: Commit and Push Results
        run: |
          git config --global user.name "github-actions"
          git config --global user.email "actions@github.com"
          git add test_results.md
          git commit -m "Update test results"
          git push origin main
```

---

## **4. GitHub Wiki や Pages で公開**
✅ `test_results.md` を **GitHub Wiki** や **GitHub Pages** で公開可能  
✅ 関係者が **ブラウザからテスト結果を閲覧** できる  
✅ Markdownの見やすさを活かし、**フォーマットを統一** できる  

📌 **例:**  
- `docs/` フォルダに `test_results.md` を入れて GitHub Pages で表示  
- `wiki/` に `Test_Results` ページを作成して管理

---

## **5. GitHub Issues と連携してバグ管理**
GitHubのIssueを使って、不具合を記録＆管理できます。

✅ `test_results.md` で「失敗」したケースは、**Issueに変換**  
✅ `test_results.md` 内に **Issueのリンクを貼る** ことで管理しやすくする  
✅ `test_results.md` の更新と **Issueのクローズ** を自動化可能（GitHub Actions）  

📌 **例:**  
テストケースが失敗した場合、対応する Issue を作成
```md
### **#001 メール未送信バグ**
- **発生個所**: パスワードリセット機能
- **Issue**: [#42 メールが送信されない](https://github.com/example/project/issues/42)
```

---

## **結論：GitHubに `test_results.md` を保存すると…**
1. **テスト履歴がGitで管理できる**（変更履歴が追える）  
2. **PRでレビュー＆修正** ができる（開発者とテスターの連携がスムーズ）  
3. **GitHub Actionsで自動化** できる（手動で書く手間を減らせる）  
4. **GitHub WikiやPagesで公開** 可能（チームで簡単に参照できる）  
5. **GitHub Issuesと連携** してバグ管理が楽になる  

👉 **小規模プロジェクトなら `test_results.md` で十分！**  
👉 **大規模なら GitHub Actions や Jira、TestRail などと併用！**  

### **次のステップ**
- `test_results.md` を **実際にGitHubリポジトリで運用** してみる  
- **GitHub Actions を活用して、テスト結果の自動記録** に挑戦  
- **バグ管理をIssueと連携** させて、効率的なテストプロセスを作る  

📌 **もし実際に Markdown のフォーマットを改良したい場合、具体的な要件を教えてください！**

</details>

<details><summary>test_results.md　はGitHubで使うと聞きましたが、test_results.mdde

</summary>

#### Q.
test_results.md　はGitHubで使うと聞きましたが、test_results.mdde

#### A.
途中でメッセージが切れているようですが、「test_results.md」はGitHubでどのように使うのか、またはその内容について知りたいということでしょうか？  

一般的に `test_results.md` は、テストの結果を記録するMarkdownファイルとして使われます。GitHubのリポジトリでこのファイルを使うと、次のような目的で活用できます：  

1. **テスト結果のドキュメント化**  
   - CI/CD（継続的インテグレーション/デリバリー）ツールでテストを実行し、その結果を `test_results.md` に記録することがある。  
   - 人間が読める形式でテストの成功・失敗を記録し、プロジェクトの品質管理に役立てる。  

2. **自動生成レポートの出力**  
   - GitHub Actions などのCIツールを使用し、テストの結果をMarkdown形式で生成して `test_results.md` に出力することができる。  
   - 例えば、pytestやJUnitの結果を変換してMarkdownファイルとして保存することがある。  

3. **プロジェクトメンバーとの共有**  
   - リポジトリのドキュメントとして、最新のテスト結果をチームと共有するために使う。  
   - プルリクエストのレビュー時に参考情報として活用可能。  

このように、`test_results.md` はテスト結果を整理し、GitHub上で簡単に閲覧できるようにするために使われることが多いです。  

具体的に何を知りたいのか、もう少し詳しく教えてもらえますか？ 😊

</details>

<details><summary>test_results.md はどうやって作りますか？その書式は一体何ですか？

</summary>

#### Q.
test_results.md はどうやって作りますか？その書式は一体何ですか？

#### A.
`test_results.md` は単なる **Markdown形式のファイル** なので、自由に作成できます。以下の手順で作成できます。  

### 📌 **1. `test_results.md` の作成方法**
#### ✅ **手動で作成する場合**
1. 任意のエディタ（VS Code、Notepad++ など）を開く  
2. `test_results.md` という名前で新しいファイルを作成  
3. 以下のようなMarkdownのフォーマットでテスト結果を記述  
4. GitHubリポジトリに追加してコミット＆プッシュ  

#### ✅ **自動生成する場合**
- **Pythonスクリプト（pytest + pytest-md）でMarkdownレポートを作成**
- **GitHub ActionsでCIの結果をMarkdownに出力**
- **JenkinsやCircleCIでMarkdown形式のテストレポートを生成**

---

### 📝 **2. `test_results.md` の基本フォーマット**
Markdownはシンプルな書式なので、普通のテキストに記号を付けるだけで書けます。

#### **💡 シンプルな例**
```md
# 📝 テスト結果レポート

## ✅ 概要
- 実行日時: 2025-03-04
- テスト環境: Python 3.10, Ubuntu 22.04
- テストフレームワーク: pytest

## ✅ 成功したテスト
- `test_login_success` ✔️
- `test_signup_valid_data` ✔️
- `test_logout` ✔️

## ❌ 失敗したテスト
| テスト名             | エラー内容                          |
|----------------------|----------------------------------|
| `test_invalid_login` | AssertionError: expected 200, got 401 |
| `test_missing_email` | ValueError: Email is required   |

## 🔍 詳細ログ
```
```
============================= test session starts =============================
collected 5 items

tests/test_auth.py ..F.                                              [ 80%]
tests/test_profile.py F                                              [100%]

================================ FAILURES =================================
____________________________ test_invalid_login ___________________________
AssertionError: expected 200, got 401
=========================== short test summary ===========================
FAILED tests/test_auth.py::test_invalid_login - AssertionError: expected...
FAILED tests/test_profile.py::test_missing_email - ValueError: Email is ...
```
```

---

### 🎯 **3. 自動で `test_results.md` を作る方法**
#### ✅ **Python (pytest + pytest-md) を使う**
Pythonの `pytest-md` プラグインを使えば、pytestのテスト結果をMarkdown形式で出力できます。

1. `pytest-md` をインストール  
   ```sh
   pip install pytest-md
   ```
2. `pytest` を実行し、Markdownレポートを作成  
   ```sh
   pytest --md-report=./test_results.md
   ```
3. 生成された `test_results.md` を確認  

#### ✅ **GitHub ActionsでMarkdownを作る**
GitHub ActionsのCI/CDで、テスト結果を自動で `test_results.md` に記録できます。

```yaml
name: Run Tests and Save Report

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: チェックアウト
        uses: actions/checkout@v3

      - name: Python セットアップ
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: 依存関係をインストール
        run: pip install pytest pytest-md

      - name: テストを実行してMarkdownレポートを作成
        run: pytest --md-report=test_results.md

      - name: レポートをアップロード
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: test_results.md
```

GitHub Actions でこの設定を使うと、テスト実行後に `test_results.md` が生成され、GitHubの「Artifacts（アーティファクト）」からダウンロードできます。

---

### ✅ **まとめ**
- `test_results.md` は **Markdown形式** のファイルで、自由に手動作成できる
- 基本的には **# 見出し、表、リスト、コードブロック** などを使って書く
- `pytest-md` などを使えば **自動でMarkdownレポート** を生成可能
- **GitHub Actions でCI結果をMarkdownに出力** することもできる

具体的にどの方法を試したいですか？ 😊

</details>

<details><summary>ソフトウェアテストで、テスト実施記録をマークダウンに書くのは妥当ですか？

</summary>

#### Q.
ソフトウェアテストで、テスト実施記録をマークダウンに書くのは妥当ですか？

#### A.
はい、ソフトウェアテストの **テスト実施記録** をマークダウン（Markdown）で書くのは妥当な選択です。  
特に以下のような理由から、Markdown はテスト記録のフォーマットとして適しています。

---

## ✅ **Markdown を使うメリット**
### 1️⃣ **軽量で可読性が高い**
- Markdown はシンプルな記法なので、**余計な装飾なしにテスト結果を簡潔に記述** できます。
- プレーンテキストに近い形式のため、バージョン管理しやすく、レビューしやすい。

### 2️⃣ **Git との相性が良い**
- Markdown は GitHub や GitLab などのリポジトリ管理ツールでそのまま表示可能。
- **テスト結果の履歴管理がしやすく、差分（diff）も明確に確認可能**。

### 3️⃣ **フォーマットの統一が簡単**
- ヘッダー (`#`), 箇条書き (`-`), テーブル (`|`), コードブロック (```) などで、  
  **統一されたフォーマットのテスト実施記録を作成** できる。
- チーム全体で **一定の書式を維持** しやすい。

### 4️⃣ **他のツールと連携しやすい**
- Markdown は **JIRA, Confluence, GitHub Issues, Notion** などのツールでもサポートされているため、  
  **他のドキュメントと統合しやすい**。
- HTML や PDF に簡単に変換できる。

---

## 📌 **Markdown を使ったテスト実施記録の例**
```md
# テスト実施記録

## 🛠 テスト概要
- **テスト実施日:** 2025/03/05
- **テスト担当者:** Alice
- **対象バージョン:** v1.2.3
- **環境:** Windows 11, Python 3.10, SQLite

## 🔍 テスト項目
| No | テストケース | 結果 | メモ |
|----|------------|------|------|
| 1  | ユーザーが正常にログインできるか | ✅ 成功 | - |
| 2  | 無効なパスワードでエラーが出るか | ✅ 成功 | - |
| 3  | データベースのUserテーブルが存在するか | ❌ 失敗 | `no such table: User` エラー発生 |

## 📝 メモ・考察
- `sqlite_database.py` を `database/` に移動した影響で、データベースのパスが変更されている可能性がある。
- `sample.db` の場所を確認し、正しくパスを指定するよう修正が必要。

## 🚀 次のアクション
- [ ] `sample.db` のパスを調整
- [ ] `User` テーブルの存在を確認し、必要ならマイグレーションを実施
```

---

## ❌ **Markdown では不向きなケース**
### 1. **テストケースの自動管理が必要**
- テスト管理ツール（TestRail, Zephyr, Xray など）を使用して、  
  **テストケースや実施結果を自動で管理・集計する場合は Markdown では限界がある**。

### 2. **複雑なレポートやグラフが必要**
- Markdown では **リッチなレポートや統計データの可視化（グラフ・チャート）** が難しい。
- その場合は **Excel, Google Sheets, Jupyter Notebook** などの選択肢を検討。

### 3. **大規模プロジェクトでの管理**
- 大規模なプロジェクトでは、TestRail などの専用テスト管理ツールのほうが、  
  **進捗管理やテストケースのトラッキングがしやすい**。

---

## 🎯 **結論**
✅ **Markdown は、軽量で Git との相性が良いため、小～中規模のテスト実施記録には最適。**  
✅ **特に、手動テストの記録やバグ報告を残す用途なら Markdown は十分に使える。**  
❌ **ただし、大規模プロジェクトや自動テストの管理には専用ツールの方が向いている。**

👉 もし **チームでの運用ルールや既存のワークフロー** に合うなら、Markdown を活用するのは良い選択です！

</details>

<details><summary>markdownでmdファイルをビューアモードで開いたとき、テーブルとチェックボックスはどのように書けばよいですか？</summary>

#### Q.
markdownでmdファイルをビューアモードで開いたとき、テーブルとチェックボックスはどのように書けばよいですか？

#### A.
 Markdown で **.md ファイルをビューアモード（プレビュー表示）** したときに、テーブルやチェックボックスを正しく表示させるには、以下のように記述します。

---

## ✅ **テーブルの書き方**
Markdown のテーブルは `|`（パイプ記号）と `-`（ハイフン）を使います。

```md
| No | テストケース                        | 結果  | メモ |
|----|--------------------------------|------|------|
| 1  | ユーザーが正常にログインできるか | ✅ 成功 | - |
| 2  | 無効なパスワードでエラーが出るか | ✅ 成功 | - |
| 3  | データベースのUserテーブルが存在するか | ❌ 失敗 | `no such table: User` |
```

🔽 **ビューアモードでの表示（GitHub, VSCode など）**
| No | テストケース                        | 結果  | メモ |
|----|--------------------------------|------|------|
| 1  | ユーザーが正常にログインできるか | ✅ 成功 | - |
| 2  | 無効なパスワードでエラーが出るか | ✅ 成功 | - |
| 3  | データベースのUserテーブルが存在するか | ❌ 失敗 | `no such table: User` |

### 🔹 **ポイント**
- `|`（パイプ記号）で列を区切る
- `-`（ハイフン）を 1 行目に入れてヘッダーを作成
- `:---:` で列の **左寄せ（`:---`）、中央寄せ（`:---:`）、右寄せ（`---:`）** を指定できる

```md
| 左寄せ  | 中央寄せ | 右寄せ  |
|:------|:------:|------:|
| 例1    | 例2    | 例3    |
```

🔽 **表示**
| 左寄せ  | 中央寄せ | 右寄せ  |
|:------|:------:|------:|
| 例1    | 例2    | 例3    |

---

## ✅ **チェックボックスの書き方**
Markdown では `- [ ]` を使ってチェックリスト（ToDoリスト）を作成できます。

```md
## 🚀 次のアクション
- [ ] `sample.db` のパスを調整
- [ ] `User` テーブルの存在を確認し、必要ならマイグレーションを実施
- [x] エラーメッセージのログを確認
```

🔽 **ビューアモードでの表示（GitHub, VSCode など）**
## 🚀 次のアクション
- [ ] `sample.db` のパスを調整
- [ ] `User` テーブルの存在を確認し、必要ならマイグレーションを実施
- [x] エラーメッセージのログを確認 ✅（チェック済み）

### 🔹 **ポイント**
- `- [ ]` → チェックなし
- `- [x]` → ✅ チェック済み
- GitHub などでは **チェックボックスをクリックすると状態が変更される**（ただし、VSCode などでは編集モードで手動で `[x]` にする必要あり）

---

## 🎯 **まとめ**
✅ **テーブル** → `|` と `-` を使う  
✅ **チェックボックス** → `- [ ]` or `- [x]` を使う  
✅ **GitHub, VSCode, GitLab, JIRA, Notion などのプレビューで正しく表示される**  

👉 **実際に `.md` ファイルで試して、プレビュー表示を確認するのがおすすめ！**

</details>

<details><summary>pyファイルが全部消えた</summary>

#### Q.
コマンドプロンプトで以下のように入力した。3/5水　13：20
>
Microsoft Windows [Version 10.0.26100.3194]
(c) Microsoft Corporation. All rights reserved.

C:\Obento-Test\Documents>git status
HEAD detached from 222c4a5
Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   ../v_0.1.3/app/__pycache__/main.cpython-312.pyc
        deleted:    ../v_0.1.3/app/_example.db
        modified:   ../v_0.1.3/app/database/example.db
        modified:   ../v_0.1.3/app/main.py

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        ../v_0.1.3/app/database/_example.db

no changes added to commit (use "git add" and/or "git commit -a")

C:\Obento-Test\Documents>git switch main
error: Your local changes to the following files would be overwritten by checkout:
        v_0.1.3/app/__pycache__/main.cpython-312.pyc
        v_0.1.3/app/database/example.db
        v_0.1.3/app/main.py
Please commit your changes or stash them before you switch branches.
Aborting

C:\Obento-Test\Documents>git branch -a
* (HEAD detached from 222c4a5)
  main
  正常系時刻修正前
  remotes/origin/HEAD -> origin/main
  remotes/origin/main

C:\Obento-Test\Documents>git rev-parse HEAD
b0942dbfb7861e618679600ba3545492dfcb4838

C:\Obento-Test\Documents>git checkout
M       v_0.1.3/app/__pycache__/main.cpython-312.pyc
D       v_0.1.3/app/_example.db
M       v_0.1.3/app/database/example.db
M       v_0.1.3/app/main.py

C:\Obento-Test\Documents>git rev-parse HEAD
b0942dbfb7861e618679600ba3545492dfcb4838

C:\Obento-Test\Documents>git status
HEAD detached from 222c4a5
Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   ../v_0.1.3/app/__pycache__/main.cpython-312.pyc
        deleted:    ../v_0.1.3/app/_example.db
        modified:   ../v_0.1.3/app/database/example.db
        modified:   ../v_0.1.3/app/main.py

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        ../v_0.1.3/app/database/_example.db

no changes added to commit (use "git add" and/or "git commit -a")

C:\Obento-Test\Documents>git rm _example.db
fatal: pathspec '_example.db' did not match any files

C:\Obento-Test\Documents>git rm ../v_0.1.3/app/_example.db
rm 'v_0.1.3/app/_example.db'

C:\Obento-Test\Documents>git status
HEAD detached from 222c4a5
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        deleted:    ../v_0.1.3/app/_example.db

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   ../v_0.1.3/app/__pycache__/main.cpython-312.pyc
        modified:   ../v_0.1.3/app/database/example.db
        modified:   ../v_0.1.3/app/main.py

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        ../v_0.1.3/app/database/_example.db


C:\Obento-Test\Documents>git rm -r --cached __pycache__
fatal: pathspec '__pycache__' did not match any files

C:\Obento-Test\Documents>git rm -r --cached ../v_0.1.3/app/__pycache__/main.cpython-312.pyc
rm 'v_0.1.3/app/__pycache__/main.cpython-312.pyc'

C:\Obento-Test\Documents>echo "__pycache__/" >> .gitignore

C:\Obento-Test\Documents>git switch main
error: Your local changes to the following files would be overwritten by checkout:
        v_0.1.3/app/database/example.db
        v_0.1.3/app/main.py
Please commit your changes or stash them before you switch branches.
error: The following untracked working tree files would be removed by checkout:
        v_0.1.3/app/__pycache__/main.cpython-312.pyc
Please move or remove them before you switch branches.
Aborting

C:\Obento-Test\Documents>git checkout main
error: Your local changes to the following files would be overwritten by checkout:
        v_0.1.3/app/database/example.db
        v_0.1.3/app/main.py
Please commit your changes or stash them before you switch branches.
error: The following untracked working tree files would be removed by checkout:
        v_0.1.3/app/__pycache__/main.cpython-312.pyc
Please move or remove them before you switch branches.
Aborting

C:\Obento-Test\Documents>git add .

C:\Obento-Test\Documents>git commit -m "WIP: 作業中の変更をコミット"
[detached HEAD ead0154] WIP: 作業中の変更をコミット
 3 files changed, 1 insertion(+)
 create mode 100644 Documents/.gitignore
 delete mode 100644 v_0.1.3/app/__pycache__/main.cpython-312.pyc
 delete mode 100644 v_0.1.3/app/_example.db

C:\Obento-Test\Documents>git checkout main
error: Your local changes to the following files would be overwritten by checkout:
        v_0.1.3/app/database/example.db
        v_0.1.3/app/main.py
Please commit your changes or stash them before you switch branches.
Aborting

C:\Obento-Test\Documents>git stash push -m "作業中の変更を一時保存"
Saved working directory and index state On (no branch): 作業中の変更を一時保存

C:\Obento-Test\Documents>git checkout main
Updating files: 100% (3293/3293), done.
Previous HEAD position was ead0154 WIP: 作業中の変更をコミット
Switched to branch 'main'
Your branch is up to date with 'origin/main'.

C:\Obento-Test\Documents>git add .

C:\Obento-Test\Documents>git commit -m "mainに保存した"
On branch main
Your branch is up to date with 'origin/main'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        ../v_0.1.3/

nothing added to commit but untracked files present (use "git add" to track)

C:\Obento-Test\Documents>git add ../v_0.1.3/

C:\Obento-Test\Documents>git commit -m "v_0.1.3/をmainに保存した"
[main 1216133] v_0.1.3/をmainに保存した
 2 files changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 v_0.1.3/app/__pycache__/main.cpython-312.pyc
 create mode 100644 v_0.1.3/app/database/_example.db

C:\Obento-Test\Documents>git push origin main
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 8 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (8/8), 7.94 KiB | 3.97 MiB/s, done.
Total 8 (delta 1), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To https://github.com/Kei-Okuma1216/Obento-Test.git
   222c4a5..1216133  main -> main

C:\Obento-Test\Documents>git stash pop
CONFLICT (modify/delete): v_0.1.3/app/main.py deleted in Updated upstream and modified in Stashed changes.  Version Stashed changes of v_0.1.3/app/main.py left in tree.
warning: Cannot merge binary files: v_0.2.0/app/example.db (Updated upstream:v_0.2.0/app/example.db vs. Stashed changes:v_0.1.3/app/database/example.db)
Auto-merging v_0.2.0/app/example.db
CONFLICT (content): Merge conflict in v_0.2.0/app/example.db
On branch main
Your branch is up to date with 'origin/main'.

Unmerged paths:
  (use "git restore --staged <file>..." to unstage)
  (use "git add/rm <file>..." as appropriate to mark resolution)
        deleted by us:   ../v_0.1.3/app/main.py
        both modified:   ../v_0.2.0/app/example.db

no changes added to commit (use "git add" and/or "git commit -a")
The stash entry is kept in case you need it again.

C:\Obento-Test\Documents>git stash pop
v_0.1.3/app/main.py: needs merge
v_0.2.0/app/example.db: needs merge
error: could not write index
The stash entry is kept in case you need it again.

C:\Obento-Test\Documents>git restore
fatal: you must specify path(s) to restore

C:\Obento-Test\Documents>



#### A.
### **💡 `v_0.1.3` フォルダ内の `.py` ファイルが消えた理由と復元方法**
あなたの `git` コマンドの実行履歴を分析すると、`.py` ファイルが消えた可能性のある操作がいくつかあります。

---

## **🔍 `.py` ファイルが消えた可能性のある原因**
1. **`git checkout main` で `HEAD detached` の状態から `main` に切り替えた**
   - `HEAD` が分離状態（detached HEAD）だったため、`checkout` で **一部のファイルが `main` の状態に戻った** 可能性がある。

2. **`git stash pop` による競合発生**
   - `stash` した変更を `pop` したとき、  
     `v_0.1.3/app/main.py` が `deleted by us` になり、  
     **手動で復元しない限り `main.py` が削除された状態になった**。

3. **`git add ../v_0.1.3/` で `v_0.1.3/app/main.py` 以外の `.py` ファイルが `untracked` のまま**
   - `.py` ファイルが `git` に登録されていなかった可能性がある。

---

## **🛠 `.py` ファイルを復元する方法**
### **✅ 1️⃣ `git log` で `.py` ファイルの変更履歴を確認**
まず、過去の `.py` ファイルの状態を確認します。

```sh
git log --name-status -- v_0.1.3/app/
```
🔹 **`main.py` や他の `.py` ファイルが最後にコミットされたバージョンを探す。**

---

### **✅ 2️⃣ `git checkout` で `.py` ファイルを復元**
もし、特定のコミットで `.py` ファイルが存在するなら、そのバージョンを復元できます。

1. `git log` で `.py` が存在していた最新のコミット `XXXXXXX` を見つける
2. そのコミットから `.py` ファイルを復元：
   ```sh
   git checkout XXXXXXX -- v_0.1.3/app/main.py
   ```

---

### **✅ 3️⃣ `git reflog` で過去の状態を探す**
もし `git log` で見つからない場合、  
`git reflog` を使って「過去の作業状態」を確認できます。

```sh
git reflog
```
🔹 `stash` する前や `commit` する前の状態を探す。  
🔹 例えば `stash` する前の `HEAD@{3}` に `.py` がある場合：
```sh
git checkout HEAD@{3} -- v_0.1.3/app/main.py
```

---

### **✅ 4️⃣ `.py` ファイルが untracked（未追跡）なら `git ls-files` で探す**
```sh
git ls-files --others --exclude-standard
```
🔹 ここに `.py` ファイルが表示されるなら、Git が追跡していないだけなので：
```sh
git add v_0.1.3/app/*.py
git commit -m "復元: 失われた .py ファイル"
```

---

## **🎯 まとめ**
| 状況 | 解決策 |
|------|--------|
| `.py` ファイルが最新のコミットにあった | `git checkout XXXXXXX -- v_0.1.3/app/main.py` |
| `stash` する前の状態を探したい | `git reflog` で過去の `HEAD@{X}` を確認 |
| `git` に未追跡の `.py` があるか調べる | `git ls-files --others --exclude-standard` |

---

✅ まずは `git log --name-status -- v_0.1.3/app/` で **`.py` ファイルの最後のコミット履歴** を確認してください！ 🚀

</details>

<details><summary>テスト実施書類をつくりたい。チェックを入れたら自動的に保存されるmdファイルは可能ですか？githubに保存したmdファイルのみ可能ですか？</summary>

#### Q.
テスト実施書類をつくりたい。チェックを入れたら自動的に保存されるmdファイルは可能ですか？githubに保存したmdファイルのみ可能ですか？

#### A.
### **✅ チェックを入れたら自動的に保存される Markdown（.md）ファイルは可能か？**  
結論として、**チェックを入れるだけで自動保存される Markdown ファイルを作ることは可能ですが、方法によります。**  

**📌 主な選択肢：**
1. **GitHub の Issues / Projects / Wiki / Gist（オンラインで管理）**
2. **ローカルアプリ（Obsidian / VSCode + Markdown 拡張機能）**
3. **カスタム Web アプリ（HTML + JavaScript + GitHub API）**
4. **Google Docs / Notion など、Markdown 変換可能なツールを使う**

---

## **🛠 1️⃣ GitHub の Markdown でチェックボックスを管理**
GitHub では、 **Issues / Projects / Wiki / Gist** で Markdown をサポートしており、チェックを入れると **ブラウザ上で自動保存** されます。

### **✅ 例: GitHub Issues でチェックリストを管理**
1. **GitHub のリポジトリで Issue を作成**
2. **Markdown 形式でチェックリストを記述**
   ```md
   - [ ] テストケース A
   - [x] テストケース B（完了）
   - [ ] テストケース C
   ```
3. **チェックを入れると、GitHub が自動保存**

**👉 ただし、 `.md` ファイルとしてローカルに自動保存されるわけではない。**

---

## **🛠 2️⃣ ローカルの Markdown エディタ + 自動保存**
ローカルで Markdown ファイルを編集し、**チェックを入れるたびに自動保存する方法** もあります。

### **✅ Obsidian（無料・ローカル）**
- **Obsidian** は **Markdown をリアルタイムで保存** できるエディタ。
- `- [ ]` 形式のチェックボックスを自動認識し、クリックで `- [x]` に変更可能。
- **デフォルトで変更を自動保存**。

👉 **手軽にローカルで管理したいなら、Obsidian がベスト！** 🚀

---

## **🛠 3️⃣ カスタム Web アプリで自動保存**
もし **ブラウザ上でチェックを入れるだけで `.md` を自動保存したい** なら、以下の方法もあります。

### **✅ JavaScript + GitHub API を使う**
1. **HTML + JavaScript でチェックリストを作成**
2. **チェックを入れると、自動で `.md` ファイルを GitHub に Push**
3. **ローカルファイルにも保存できる（ブラウザの FileSystem API を利用）**

**📌 参考コード（HTML + JavaScript でチェックリストを自動保存）**
```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>Markdown チェックリスト</title>
</head>
<body>
    <h2>テストチェックリスト</h2>
    <ul id="checklist">
        <li><input type="checkbox" id="test1"> テストケース A</li>
        <li><input type="checkbox" id="test2" checked> テストケース B</li>
    </ul>
    <button onclick="saveMarkdown()">保存</button>

    <script>
        function saveMarkdown() {
            let mdContent = "";
            document.querySelectorAll("#checklist li").forEach(li => {
                let checkbox = li.querySelector("input");
                let checked = checkbox.checked ? "[x]" : "[ ]";
                mdContent += `- ${checked} ${li.textContent.trim()}\n`;
            });

            // ローカルに保存
            let blob = new Blob([mdContent], { type: "text/markdown" });
            let a = document.createElement("a");
            a.href = URL.createObjectURL(blob);
            a.download = "test_checklist.md";
            a.click();
        }
    </script>
</body>
</html>
```
👉 **このコードをブラウザで開くと、チェックを入れて「保存」ボタンを押すだけで `.md` がダウンロードされる！** 🎉

---

## **🛠 4️⃣ Google Docs / Notion を Markdown でエクスポート**
もし **クラウド上で管理しつつ、Markdown 形式で `.md` に変換** したいなら、以下のツールも有効。

### **✅ Google Docs**
1. Google Docs でチェックリストを作成
2. `File > Download > Markdown (.md)` で `.md` に変換して保存

### **✅ Notion**
1. Notion でチェックリストを作成
2. `Export > Markdown` で `.md` に保存

👉 **クラウドでチーム管理するなら、Google Docs や Notion も便利！**

---

## **🎯 まとめ**
| 方法 | 自動保存 | `.md` ファイルとして保存 | 特徴 |
|------|---------|----------------|------|
| **GitHub Issues / Wiki / Gist** | ✅ | ❌（`.md` ではなく Web で保存） | GitHub で手軽に管理できる |
| **Obsidian / VSCode + 拡張機能** | ✅ | ✅（ローカルに `.md` 保存） | ローカルで管理するならベスト |
| **カスタム Web アプリ（HTML + JS）** | ✅ | ✅（チェックを入れたら `.md` 保存） | カスタマイズ可能 |
| **Google Docs / Notion** | ✅ | ✅（エクスポートで `.md` 保存） | チーム向け |

✅ **ローカルなら Obsidian / VSCode、ブラウザなら GitHub Issues or カスタム Web アプリがおすすめ！** 🚀

</details>

<details>
<summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

<details><summary>aaa</summary>

#### Q.


#### A.


</details>

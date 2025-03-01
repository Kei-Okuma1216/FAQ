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

<details><summary>aaa</summary>

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

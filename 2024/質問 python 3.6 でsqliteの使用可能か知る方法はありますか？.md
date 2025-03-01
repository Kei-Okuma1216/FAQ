#### Q.
- 
python 3.6 でsqliteの使用可能か知る方法はありますか？

#### A.
- 
Python 3.6では、標準ライブラリとしてSQLiteのサポートが含まれています。ですので、Python 3.6を使用している限り、追加のインストールなしでSQLiteを使用することができます。

SQLiteを使用するためのサンプルコードを以下に示します。これを実行してみて、正常に動作すればSQLiteが使用可能であることが確認できます。

```python
import sqlite3

# SQLiteデータベースに接続（データベースが存在しない場合は自動的に作成されます）
conn = sqlite3.connect('test.db')

# カーソルオブジェクトを作成
cursor = conn.cursor()

# テーブルを作成するためのSQL文
create_table_query = '''
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    age INTEGER
)
'''

# テーブルを作成
cursor.execute(create_table_query)

# データを挿入
insert_data_query = '''
INSERT INTO users (name, age)
VALUES ('Alice', 30), ('Bob', 25)
'''

cursor.execute(insert_data_query)

# データをコミット（保存）
conn.commit()

# データを取得して表示
cursor.execute('SELECT * FROM users')
rows = cursor.fetchall()
for row in rows:
    print(row)

# 接続を閉じる
conn.close()
```

このコードを実行すると、`test.db`というSQLiteデータベースファイルが作成され、「users」テーブルが作成され、いくつかのデータが挿入され、最後にそのデータが表示されます。

他にも質問や確認したいことがあれば、いつでもお知らせください！😊


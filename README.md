[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Soliton-Analytics-Team/FullTextSearch_Pgroonga/blob/main/FullTextSearch_Pgroonga.ipynb)

# Colabで全文検索（その5：PGroonga編）

各種全文検索ツールをColabで動かしてみるシリーズです。全7回の予定です。今回はPGroongaです。PGroongaはGroongaをPostgreSQLで使えるようにした拡張機能です。PostgreSQLのネイティブな全文検索に対してどれぐらい優位性があるのでしょうか。

処理時間の計測はストレージのキャッシュとの兼ね合いがあるので、2回測ります。2回目は全てがメモリに載った状態での性能評価になります。ただ1回目もデータを投入した直後なので、メモリに載ってしまっている可能性があります。

## 準備

まずは検索対象のテキストを日本語wikiから取得して、Google Driveに保存します。（※ Google Driveに約１GBの空き容量が必要です。以前のデータが残っている場合は取得せず再利用します。）

Google Driveのマウント


```python
from google.colab import drive
drive.mount('/content/drive')
```

    Mounted at /content/drive


jawikiの取得とjson形式に変換。90分ほど時間がかかります。他の全文検索シリーズでも同じデータを使うので、他の記事も試す場合は wiki.json.bz2 を捨てずに残しておくことをおすすめします。


```shell
%%time
%cd /content/
import os
if not os.path.exists('/content/drive/MyDrive/wiki.json.bz2'):
    !wget https://dumps.wikimedia.org/jawiki/latest/jawiki-latest-pages-articles.xml.bz2
    !pip install wikiextractor
    !python -m wikiextractor.WikiExtractor --no-templates --processes 4 --json -b 10G -o - jawiki-latest-pages-articles.xml.bz2 | bzip2 -c > /content/drive/MyDrive/wiki.json.bz2
```

    /content
    CPU times: user 1.77 ms, sys: 121 µs, total: 1.89 ms
    Wall time: 3.58 ms


json形式に変換されたデータを確認


```python
import json
import bz2

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    for n, line in enumerate(fin):
        data = json.loads(line)
        print(data['title'].strip(), data['text'].replace('\n', '')[:40], sep='\t')
        if n == 5:
            break
```

    アンパサンド	アンパサンド（&amp;, ）は、並立助詞「…と…」を意味する記号である。ラテン
    言語	言語（げんご）は、広辞苑や大辞泉には次のように解説されている。『日本大百科事典』
    日本語	 日本語（にほんご、にっぽんご）は、日本国内や、かつての日本領だった国、そして日
    地理学	地理学（ちりがく、、、伊：geografia、）は、。地域や空間、場所、自然環境
    EU (曖昧さ回避)	EU
    国の一覧	国の一覧（くにのいちらん）は、世界の独立国の一覧。対象.国際法上国家と言えるか否


## PostgreSQLのインストール


```shell
!sudo apt update
!sudo apt install postgresql
```

ローカルから無条件でアクセスできるようにconfファイルを書き換えます。


```shell
!sed -i -e 's/peer/trust/g' /etc/postgresql/10/main/pg_hba.conf
```

PostgreSQLを起動します。


```shell
!service postgresql start
```

     * Starting PostgreSQL 10 database server
       ...done.


## PGroongaのインストール


```shell
!sudo apt install -y software-properties-common
!sudo add-apt-repository -y universe
!sudo add-apt-repository -y ppa:groonga/ppa
!sudo apt update
!sudo apt install -y -V postgresql-10-pgroonga
```

## DBの作成

データベースを作成します


```shell
!sudo -u postgres -H psql --command 'CREATE DATABASE db'
```

    CREATE DATABASE


PGroongaを導入します。


```shell
!sudo -u postgres -H psql -d db --command 'CREATE EXTENSION pgroonga'
```

    CREATE EXTENSION


## データーのインポート

テーブルを作成します。


```shell
!sudo -u postgres -H psql -d db --command 'CREATE TABLE wiki_jp (title text, body text)'
```

    CREATE TABLE


データを50万件読み込みます。


```python
import psycopg2
import bz2
import json
from tqdm.notebook import tqdm

db = psycopg2.connect(database="db", user="postgres")
cursor = db.cursor()

cursor.execute('drop table if exists wiki_jp')
cursor.execute('create table wiki_jp(title text, body text)')

limit = 500000
insert_wiki = 'insert into wiki_jp (title, body) values (%s, %s);'

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    n = 0
    for line in tqdm(fin, total=limit*1.5):
        data = json.loads(line)
        title = data['title'].strip()
        body = data['text'].replace('\n', '')
        if len(title) > 0 and len(body) > 0:
            cursor.execute(insert_wiki, (title, body))
            n += 1
        if n == limit:
            break
db.commit()
db.close()
```

    /usr/local/lib/python3.7/dist-packages/psycopg2/__init__.py:144: UserWarning: The psycopg2 wheel package will be renamed from release 2.8; in order to keep installing from binary please use "pip install psycopg2-binary" instead. For details see: <http://initd.org/psycopg/docs/install.html#binary-install-from-pypi>.
      """)



      0%|          | 0/750000.0 [00:00<?, ?it/s]


登録件数を確認します。


```shell
!echo "select count(*) from wiki_jp;" | sudo -u postgres psql db
```

     count  
    --------
     500000
    (1 row)
    


## インデックス無しの検索

検索時間を測定するためのオプションを追加した検索コマンドをファイルに書き込みます。


```shell
%%writefile command.txt
\timing
select * from wiki_jp where body &@ '日本語';
```

    Writing command.txt


1回目


```shell
%%time
!sudo -u postgres psql db < command.txt | tail -3
```

    (17006 rows)
    
    Time: 34130.679 ms (00:34.131)
    CPU times: user 335 ms, sys: 40.1 ms, total: 375 ms
    Wall time: 42.5 s


2回目


```shell
%%time
!sudo -u postgres psql db < command.txt | tail -3
```

    (17006 rows)
    
    Time: 34811.219 ms (00:34.811)
    CPU times: user 345 ms, sys: 49.1 ms, total: 394 ms
    Wall time: 43.6 s


pg_bigmを入れてインデックス無しで測定した場合より3倍時間がかかっています。PostgreSQLのバージョンの違いによるものかもしれません。pg_bigmを入れるために最新版のソースからビルドしたため、あちらはversion 14.1ですが、こちらは10.19です。それでは14にと思いましたが、ColabのUbuntuはバージョン18で、その場合、pgroongaはpostgresql-10にしか対応していないとのことで、諦めました。ColabのUbuntuのバージョンが変わったら、試しましょう。


```shell
!psql -V
```

    psql (PostgreSQL) 10.19 (Ubuntu 10.19-0ubuntu0.18.04.1)


*参考のため、titleのみを抽出する場合も測定します。*


```shell
%%writefile command2.txt
\timing
select title from wiki_jp where body &@ '日本語';
```

    Writing command2.txt



```shell
%%time
!sudo -u postgres psql db < command2.txt | tail -3
```

    (17006 rows)
    
    Time: 33133.625 ms (00:33.134)
    CPU times: user 258 ms, sys: 40.4 ms, total: 299 ms
    Wall time: 33.3 s


内部の検索処理の時間はほぼ変わりありません。Wall timeに違いがありますが、これは検索結果のサイズの違いにより、それをパイプで転送する時間に差が出ているものと思われます。

## インデックス有りの検索

インデックスを作成します。


```shell
%%time
!sudo -u postgres -H psql -d db --command 'CREATE INDEX pgroonga_body_index ON wiki_jp using pgroonga (body)'
```

    CREATE INDEX
    CPU times: user 3.35 s, sys: 416 ms, total: 3.77 s
    Wall time: 7min 9s


1回目


```shell
%%time
!sudo -u postgres psql db < command.txt | tail -3
```

    (17006 rows)
    
    Time: 1809.901 ms (00:01.810)
    CPU times: user 88.6 ms, sys: 19.2 ms, total: 108 ms
    Wall time: 10.6 s


2回目


```shell
%%time
!sudo -u postgres psql db < command.txt | tail -3
```

    (17006 rows)
    
    Time: 1363.482 ms (00:01.363)
    CPU times: user 85.5 ms, sys: 17.1 ms, total: 103 ms
    Wall time: 10.3 s


インデックスを使わない検索よりも内部の検索時間で25倍、データ転送を含めた全体の処理でも半分の時間で検索できています。ただ、PostgreSQLのpg_bigramを使った全文検索とほぼ同じ検索時間なので、このクエリではPGroongaを使う利点はなさそうです。

参考にtitleのみを抽出した場合


```shell
%%time
!sudo -u postgres psql db < command2.txt | tail -3
```

    (17006 rows)
    
    Time: 209.340 ms
    CPU times: user 9.55 ms, sys: 5.2 ms, total: 14.8 ms
    Wall time: 415 ms


さらに内部の検索時間で6倍、全体の処理でも20倍速くなっています。また、pg_ngrmよりも内部の検索で9倍、全体では20倍速くなっています。使い方によってはPGroongaが威力を発揮するようです。

データベースの検索時間は大きなデータの取得時間が大半を占めるため、よくデータを絞ってからデータを取得するのが良い戦略になります。

## PostgreSQLの停止


```shell
!service postgresql stop
```

     * Stopping PostgreSQL 10 database server
       ...done.


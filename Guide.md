これまでの内容を、GitHubの `README.md` としてそのままコピー＆ペーストして使えるMarkdown形式でまとめました。

右上のコピーボタン（またはテキストの選択）でコピーしてご活用ください。

```markdown
# Python `requests` ライブラリとWeb通信の裏側

Pythonの `requests` ライブラリの機能概要と、Webスクレイピング時に直面する「ブロック機能（Bot対策）」の仕組みについてのまとめです。

## 1. `requests` でできること総まとめ

`requests` はHTTP通信に関わるほぼすべてのタスクを直感的に実行できるライブラリです。

### 基本的なHTTPメソッド
* **GET:** データの取得（WebページやAPIから）
* **POST:** データの送信・登録（フォーム送信など）
* **PUT / PATCH:** 既存データの更新
* **DELETE:** データの削除

### リクエストの高度なカスタマイズ
* **クエリパラメータ:** `?key=value` を辞書形式で自動生成
* **ヘッダーのカスタマイズ:** User-Agentの偽装やAPIキーの付与
* **データの送信:** フォームデータ、JSONデータ（自動変換）、ファイルのアップロード（マルチパート送信）
* **Cookieの送信:** 任意のCookieを付与してリクエスト可能

### レスポンスの柔軟な処理
* **ステータス確認:** 200(成功)、404(未検出)などの確認と、エラー時の例外処理
* **データ取得:** テキスト(HTML等)、バイナリ(画像・PDF等)、JSON（辞書型へ自動パース）
* **ヘッダー/Cookie取得:** サーバーからの返答ヘッダーや新規Cookieの保存

### セッションとネットワーク設定
* **Sessionオブジェクト:** ログイン状態（Cookie）の維持、接続の使い回し（パフォーマンス向上）
* **セキュリティと設定:** 認証（Basic/Digest等）、タイムアウト設定、プロキシ経由、SSL証明書検証のスキップ、リダイレクト制御

---

## 2. User-Agent（ユーザーエージェント）の偽装

「プログラム（Bot）ではなく、人間の使う一般的なブラウザからのアクセスである」とサーバーに認識させるための必須テクニックです。

### なぜ必要なのか？
デフォルトのUser-Agent（例: `python-requests/2.31.0`）のままだと、多くのWebサイトで「サーバーに負荷をかける自動プログラム」とみなされ、アクセスをブロック（拒否）されるためです。

### 具体的な実装例
`headers` 引数に、一般的なブラウザのUser-Agentを指定します。

```python
import requests
import time

url = '[https://example.com](https://example.com)'

# 一般的なGoogle ChromeのUser-Agentを指定
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
}

# リクエストの送信
response = requests.get(url, headers=headers)
print(response.status_code)

# サーバーに負荷をかけないよう待機（マナー）
time.sleep(1)

```

---

## 3. Webサイト側のBot対策（ブロック手法）

User-Agent以外にも、Webサイト側は以下の指標でプログラムを検知・ブロックしています。

| ブロック手法 | 概要 | `requests` での対応 |
| --- | --- | --- |
| **IPアドレス制限** | 短時間の大量アクセス（レートリミット）や、データセンター・クラウド系IP（AWS等）からのアクセス拒否。 | アクセス間隔を空ける、プロキシを利用する。 |
| **JavaScript実行チェック** | ブラウザがJSを実行できるかチェックし、裏で計算問題を解かせる。 | **不可。** `requests` はJSを実行できない。 |
| **CAPTCHA** | 「信号機の画像を選ぶ」などのパズルや画像認証（reCAPTCHA等）。 | **不可。** 人間の手動入力が必要。 |
| **行動パターン分析** | 1.0秒間隔など規則的すぎるアクセスや、マウス挙動の不在を検知。 | アクセス間隔をランダムにする等の工夫が必要。 |

---

## 4. ブロックが設定される場所（インフラの階層）

これらの防御システムは、インフラの異なるレイヤー（階層）で実装されています。

1. **CDN / WAF (Web Application Firewall)**
* **代表格:** Cloudflare, AWS WAF, Akamai
* **特徴:** 防御の最前線。最も強力なBot検知（JSチェック、CAPTCHA、高度なAI分析）が行われる。ここを `requests` で突破するのは非常に困難。


2. **Webサーバーソフトウェア**
* **代表格:** Nginx, Apache
* **特徴:** 中間層。簡易的なIPブロックやUser-Agentの拒否、アクセス頻度制限（レートリミット）を設定。


3. **アプリケーション（バックエンド）**
* **代表格:** Django, Ruby on Rails, PHP
* **特徴:** 最後尾。ビジネスロジックに基づく制限（例：「ログイン5回失敗で凍結」）を行う。



## 5. `requests` の限界と強力なBot対策（なぜ弾かれるのか？）

`requests` はあくまで「HTTP通信を行うためのシンプルなツール」であるため、JavaScriptの計算問題以外にも以下のような壁に直面するとブロックされてしまいます。

* **TLS指紋（JA3 Fingerprint）の不一致:** 通信の初期段階（暗号化のすり合わせ）のクセを見られ、「User-AgentはChromeなのに、通信の暗号化のクセがPythonだ」と見破られます。現代のスクレイピングにおける最大の壁の一つです。
* **CAPTCHA（画像認証・パズル）:** 視覚的な認識と物理的な操作（クリックやドラッグ）が必要なため、画面やマウスを持たない `requests` では物理的に突破できません。
* **ブラウザ指紋（Browser Fingerprinting）:** パソコンにインストールされているフォントや、グラフィックボードの描画のクセ（Canvas指紋）など、ブラウザ経由で得られるハードウェア情報を要求されると応答できません。
* **ユーザーの行動分析:** マウスの軌跡やスクロールなど「人間らしい動き」がないため、URLに対して一瞬でデータを要求する `requests` はBotと判定されやすくなります。

---

## 6. `requests` で突破できない場合の代替手段（ライブラリ）

Cloudflareなどの強力なBot対策が敷かれているサイトや、JavaScriptで動的にデータが生成されるサイトからデータを取得する場合、用途に合わせて以下のライブラリへの乗り換え（または併用）を検討します。

### ① `curl_cffi` （通信の指紋を偽装したい場合）
* **概要:** `requests` とほぼ同じコードの書き方ができつつ、ChromeやSafariなどの**「TLS指紋（通信のクセ）」を完璧に偽装できる**強力なライブラリです。
* **適しているケース:** 画面操作やJavaScriptの実行は不要だが、CloudflareなどのWAFに「Pythonからの通信だ」と見破られて弾かれる場合。本物のブラウザを立ち上げないため、非常に高速で軽量です。

### ② `Playwright` / `Selenium` （実際のブラウザを操作したい場合）
* **概要:** 裏側で本物のブラウザ（Chromeなど）を立ち上げ、プログラムから自動操作するライブラリです。現在は、動作が速く待機処理などが賢い **`Playwright`** が主流になりつつあります。
* **適しているケース:** サイトがJavaScriptで描画される場合（SPA）や、ログインボタンのクリック、ページのスクロールなど、実際のユーザーと同じ操作が必要な場合。
* **注意点:** 本物のブラウザを動かすため、`requests` に比べて処理が重く、メモリやCPUを多く消費します。

### ③ `undetected-chromedriver` （Seleniumがバレる場合）
* **概要:** `Selenium` をベースにしつつ、Bot検知システム（CloudflareやDatadomeなど）に「自動操作されていること」を検知されないよう、徹底的にブラウザの内部変数を偽装・調整した特殊なパッケージです。
* **適しているケース:** 通常のSeleniumやPlaywrightを使っても、「ブラウザが自動制御されています」というフラグを検知されてブロックされてしまう、非常に防御力の高いサイトを相手にする場合。


これまでのREADMEの続きとして追加できる、強力なBot対策を突破するための実践的なコード例をまとめました。

右上のコピーボタンからコピーして、前回の内容の末尾に追記してご活用ください。

```markdown
## 7. 強力なBot対策を突破する具体的な実装例

標準の `requests` や `Selenium` では弾かれてしまう場合の、具体的な回避策とコード例です。

### ① `curl_cffi` を使ったTLS指紋の偽装
**「通信のクセ（TLS指紋）」を本物のブラウザと完全に一致させる**ことで、Cloudflareなどの強力な検知をすり抜けます。ブラウザを起動しないため、非常に高速・軽量に動作します。

**インストール:**
```bash
pip install curl_cffi

```

**実装例:**
使い方は標準の `requests` とほぼ同じですが、`impersonate` パラメータで偽装したいブラウザを指定するだけです。

```python
# 標準のrequestsではなく、curl_cffiのrequestsを使用
from curl_cffi import requests

# 自分のTLS指紋を確認できるテストサイト
url = "[https://tls.browserleaks.com/json](https://tls.browserleaks.com/json)"

# impersonate="chrome120" を指定し、通信のクセからUser-Agentまで
# すべてChrome 120と完全に一致させる（他にも "safari_ios" などが指定可能）
response = requests.get(url, impersonate="chrome120")

print(f"ステータスコード: {response.status_code}")
print("取得したデータ:", response.text[:200])

```

---

### ② `undetected-chromedriver` を使った検知回避（ブラウザ自動操作）

JavaScriptの実行や画面操作（クリック等）がどうしても必要な場合に使用する、**検知回避特化型のSelenium**です。ブラウザの内部変数（`navigator.webdriver` など）から「自動操作の痕跡」を消し去ります。

**インストール:**

```bash
pip install selenium undetected-chromedriver

```

**実装例:**
通常の `selenium.webdriver.Chrome` の代わりに `undetected_chromedriver.Chrome` を使用します。

```python
import undetected_chromedriver as uc
import time

# Windows環境などでのマルチプロセス・エラーを防ぐため、必ず if文 の中で実行
if __name__ == '__main__':
    
    # オプションの設定（必要に応じて追加）
    options = uc.ChromeOptions()
    # options.add_argument('--headless') # 画面を表示しない場合（検知リスクは少し上がる）

    print("検知回避パッチを適用したChromeを起動しています...")
    driver = uc.Chrome(options=options)

    try:
        # 強力なBotチェックが導入されているテストサイトにアクセス
        url = "[https://nowsecure.nl](https://nowsecure.nl)"
        driver.get(url)
        
        # Cloudflare等の検証画面（チェックボックスなど）を通過するまで待機
        time.sleep(8)
        
        # 突破できたか確認するためにスクリーンショットを保存
        driver.save_screenshot("result.png")
        print("現在のページタイトル:", driver.title)
        
        # ページ全体のHTMLを取得する場合
        # html = driver.page_source

    finally:
        # 最後に必ずブラウザを閉じる
        driver.quit()

```

---

## 8. 現代のWebスクレイピング基本戦略

Bot対策が厳しいサイトからデータを取得する場合、以下の順番でアプローチするのが現在の定石です。

1. **まずは `curl_cffi` を試す**
* 圧倒的に速く、メモリも消費しません。APIの直接叩き出しや、初回アクセス時のHTML内に目的のデータが含まれている場合はこれで十分です。


2. **ダメなら `undetected-chromedriver` を使う**
* ページを開いた後にJavaScriptで非同期にデータが読み込まれるサイト（SPA）や、マウス操作が必要な高度なJSチャレンジがある場合は、こちらで実際のブラウザを動かします。処理は重くなりますが、最も人間に近い挙動を再現できます。



> **⚠️ 注意事項:** > どのような技術を使っても、相手のサーバーに過度な負荷をかける行為（短時間での大量アクセスなど）は絶対に避け、適切なスリープ時間（`time.sleep()`）を設けるなど、良識あるクローリング・スクレイピングを心がけてください。

```

これで、`requests` の基礎から、限界を迎えた時の対処法、そして実際の突破用コードまでが一気通貫で揃った強力なチートシート（README）が完成しました！

GitHubのドキュメントや、ご自身の備忘録としてぜひお役立てください。他にも気になる技術やエラーがあればいつでもご質問くださいね。

## 11. `aiohttp` の弱点と、最強の非同期スクレイピング手法

非同期処理ライブラリである `aiohttp` は非常に高速ですが、**「TLS指紋（通信のクセ）がPythonである」という弱点は `requests` と同じ**です。さらに、処理が速すぎるゆえにセキュリティシステム（WAF）の検知に引っかかりやすく、非常にブロックされやすいという問題があります。

この問題を解決し、「圧倒的なスピード」と「Cloudflare等のBot検知回避」を両立させる現代の最強手法が、**`curl_cffi` の非同期モード（`AsyncSession`）** です。

### `curl_cffi` の非同期モードとは？
`curl_cffi` は、標準の `requests` の代替になるだけでなく、**`aiohttp` の代替にもなります。** これを使えば、「通信のクセをChromeに偽装した状態」のまま、数百件の並列通信を爆速で行うことができます。

**実装例（非同期 ＋ TLS指紋偽装 ＋ 並列制限）：**

```python
import asyncio
# aiohttpの代わりに、curl_cffiのAsyncSessionをインポート
from curl_cffi.requests import AsyncSession
import time

# 1件のURLにアクセスする関数
async def fetch(session, url, semaphore):
    async with semaphore:
        try:
            # 💡 ここが最強のポイント！非同期なのに impersonate="chrome120" が使える
            response = await session.get(url, impersonate="chrome120")
            return response.status_code
        except Exception as e:
            return f"Error: {e}"

async def main():
    target_url = "[https://tls.browserleaks.com/json](https://tls.browserleaks.com/json)" # Bot検知テストサイト
    urls = [target_url for _ in range(50)] # テストとして50件
    
    # サーバーへの負荷と検知を避けるため、同時接続数を10に制限
    semaphore = asyncio.Semaphore(10)
    
    print("Chromeに偽装した非同期リクエストを開始します...")
    start_time = time.time()

    # AsyncSession を使って非同期の通信経路を作成
    async with AsyncSession() as session:
        tasks = [fetch(session, url, semaphore) for url in urls]
        results = await asyncio.gather(*tasks)

    end_time = time.time()
    success_count = results.count(200)
    
    print(f"処理完了: {len(results)}件")
    print(f"成功(ステータス200): {success_count}件")
    print(f"所要時間: {end_time - start_time:.2f} 秒")

if __name__ == "__main__":
    import sys
    if sys.platform == 'win32':
        asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())
    asyncio.run(main())



🚨 高速化とブロック対策の最終チェックリスト
非同期で大量のスクレイピングを行う場合、以下の3つを必ず組み合わせてブロック（およびIP BAN）を防ぎます。

TLS指紋の偽装: curl_cffi の impersonate を使う（上記コード）。

アクセス頻度の制限: asyncio.Semaphore で同時アクセス数を絞り、必要に応じて await asyncio.sleep(1) などで人間らしい「間」を作る。

プロキシの利用（ローテーション）: 同一IPからの大量アクセスによるBANを防ぐため、商用のローテーティング・プロキシ（リクエスト毎にIPが変わるサービス）を session.get(url, proxies={"http": "...", "https": "..."}) のように設定する。

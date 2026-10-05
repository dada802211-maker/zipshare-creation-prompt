# zipshareをゼロから作成するプロンプト

以下の区切り線より下を、開発AIへの指示としてコピーしてください。既存コードを渡さずに、確認した完成版の機能・画面構成・運用方法を再現するためのプロンプトです。コードの文字単位の一致や画面の完全一致を保証するものではありません。

---

あなたはフロントエンドとPHPバックエンドを実装できるWebエンジニアです。空のディレクトリから、日本語のZIP共有アプリ「zipshare」を完成させてください。設計説明だけで終わらず、実際に動作する全ファイル、起動スクリプト、デプロイ用ビルド、README、結合テストを作成してください。モックAPIやTODOで機能を代用しないでください。

## 1. 目的・技術構成

複数のファイルを1つのZIPにまとめるか、既存ZIPをアップロードし、公開範囲を指定して共有するアプリです。

- 想定開発先：Windows / Laragonの `C:\laragon\www\multiple-uploads`。空の作業フォルダから実装すること。既存ファイルがある場合は破壊せず、空の別フォルダに作成すること。
- フロント：React 19、TypeScript 5.9、Vite 8、通常のCSS。React DOM、ESLintも設定する。
- バックエンド：PHP 8.2以上。PHPフレームワークやComposer依存は不要。
- DB：SQLite、PDOを使用。PHP拡張は `pdo_sqlite`、`zip`、`mbstring`。
- Node.js 22.12以上、npmを使用し、依存関係のlockfileを作成する。
- APIと画面は同一オリジン。認証はPHPセッションとCookieを使用する。
- UI・通知・エラーメッセージ・READMEは日本語。
- 初期ユーザーやサンプル投稿を自動作成しない。
- ドメインのルートに配置する前提。サブディレクトリURL対応は不要。

## 2. アカウント

- ユーザー登録、ログイン、ログアウトを実装する。
- 登録項目は表示名、メールアドレス、パスワード。
- 表示名は必須・最大60文字。メールは必須・最大254文字で、形式検証、小文字化、重複禁止。
- パスワードは10〜72バイト。日本語は複数バイトになることを画面に説明し、サーバーでバイト数を検証する。
- `password_hash` / `password_verify` を使用する。
- 登録成功時はそのユーザーでログイン済みにする。
- 登録・ログイン・ログアウト時にセッションIDとCSRFトークンを更新する。
- 他ユーザーの選択候補を取得するAPIはログイン必須。自分以外のID・表示名だけを返し、メールやパスワードハッシュを含めない。

## 3. ZIP登録

ログインユーザーだけが新規登録できる。フォーム内で次の2モードを切り替える。

1. 「複数ファイルをZIPにする」：1〜20ファイルを選択し、サーバーで1つのZIPを生成する。
2. 「ZIPをアップロード」：既存ZIPを1個だけ選択し、そのまま保存する。

共通の条件：

- アップロードするファイルの合計は100MiB以下（100 × 1024 × 1024バイト）。画面上は100MBと表示してよい。
- クライアントとサーバーの両方で個数・合計容量を検証する。サーバーでは実ファイルサイズ、アップロードエラー、`is_uploaded_file` も確認する。
- ZIPモードでは拡張子と `ZipArchive::CHECKCONS` による妥当性を検証する。ZIPを展開・実行しない。
- 個別ファイルのZIP内の名前はパス部分を取り除き、制御文字を置換する。空名などは安全な名前にする。
- ZIP内の同名ファイルは大文字小文字を考慮した重複チェックを行い、`2_名前`、`3_名前` のように連番を付ける。既存の連番名とも衝突しないこと。
- 実体の保存名は `bin2hex(random_bytes(24)) + '.zip'` とする。48桁の暗号学的乱数を使い、ユーザー指定名とは分離する。
- 保存先は公開ディレクトリ外の `storage/archives/`。
- ZIP保存とDB登録が途中で失敗した場合はトランザクションをロールバックし、その作成処理で生成した不完全なZIPを片付ける。

登録メタデータ：

- タイトル：必須、最大120文字。
- 説明：任意、最大5000文字。
- ダウンロード時のファイル名：必須、入力は最大150文字。末尾に `.zip` がなければ補完する。制御文字、スラッシュ、バックスラッシュを拒否する。日本語名に対応する。
- ダウンロードできる人：`public` / `members` / `selected`。
- 登録ユーザー：ログイン情報からサーバーで自動設定し、フォーム入力やリクエストによる偽装を許可しない。
- 保存済みZIPの容量と登録日時を自動記録する。

## 4. 公開範囲と権限

一覧のタイトル・説明・登録者・ダウンロード名・容量・日時は、未ログインでも表示する。公開範囲が制限する対象はZIPのダウンロードであり、投稿の一覧表示ではない。

| 値 | フォームの表示 | ダウンロード権限 |
| --- | --- | --- |
| public | 誰でもダウンロード可能 | 未ログインを含め全員 |
| members | 登録ユーザーのみ | ログイン済みの全ユーザー |
| selected | 選択したユーザーのみ | 投稿者本人と、投稿者が選択したユーザー |

- selectedでは、他ユーザーの名前とIDをチェックボックスで表示し、複数選択できるようにする。
- 自分以外の実在ユーザーを1人以上選ぶこと。空配列、配列以外、不正ID、存在しないID、自分のID、真偽値などをサーバーで拒否し、重複は除去する。
- 候補読み込み中・失敗・候補なしの表示と、失敗時の再試行を用意する。
- selectedで候補を取得できていない場合や誰も選択していない場合は保存できない。
- 編集時には選択済みユーザーを復元し、対象の追加・解除を可能にする。
- public/membersへ変更した場合は選択ユーザーとの関連を削除する。
- 一覧APIは各投稿の `can_download` を返す。`allowed_user_ids` はその投稿の所有者にだけ返す。
- 未ログインで限定ZIPのダウンロードボタンを押した場合は、ログインが必要という通知と認証ダイアログを表示する。
- ログイン済みで権限がない場合はボタンを無効化し、「ダウンロード権限がありません」と表示する。
- UIの状態だけに頼らず、APIで毎回権限を検証する。

## 5. 編集・削除

- 投稿者本人だけがタイトル、説明、ダウンロード名、公開範囲、選択対象ユーザーを変更できる。
- ZIP本体の差し替えと登録ユーザーの変更は不可。編集フォームにファイル入力を表示しない。
- 削除ボタン名は「登録情報を削除」とする。
- 削除前に確認ダイアログで、一覧から消えてダウンロードできなくなることと、ZIP本体はサーバーに保管されることを伝える。
- 削除はDBの投稿行と関連する権限行だけを対象とする。登録済みZIP本体を削除しない。
- 削除後は一覧に表示せず、IDでダウンロードしても404とする。画面からの復元機能は不要。
- 非所有者による編集・削除APIの呼び出しは403にする。

## 6. 画面とデザイン

穏やかなグリーン系のシンプルな日本語UIを、通常のCSSで実装する。

- 背景 `#f6f8f5`、本文 `#25392f`、主要ボタン `#355e42`、白いカード、薄い緑灰色の罫線。角丸8〜12px、控えめな影、十分な余白。
- フォントは `'Noto Sans JP', 'Yu Gothic UI', system-ui, sans-serif`。外部フォント配信への依存は必須にしない。
- 本文の最大幅は1160px程度、基本文字サイズ14px。PCではカード3列、850px以下で2列、580px以下で1列。
- 白いヘッダーの左に緑の角丸四角の「Z」と「zipshare.」。右に「ログイン / ユーザー登録」、ログイン後は「表示名 さん」と「ログアウト」。
- ヒーロー左側に以下の文言を表示する。
  - `YOUR FILES, TOGETHER.`
  - `まとめて届ける。` 改行 `かんたんに共有する。`
  - `複数のファイルを、ひとつのZIPに。` 改行 `必要な人に、必要な資料を届けましょう。`
  - `＋ ファイルを登録`
- ヒーロー右側にはDOC・IMGの紙とZIPを重ねたイラストをHTML/CSSで作成する。外部画像は不要。スマートフォンでは非表示にする。
- ヒーローの下に全アーカイブ数・一般公開数と「ファイルの共有を、もっとシンプルに。」を表示する。
- 新規登録・編集フォームはページ内で統計欄の下、一覧の上に表示する。
- 一覧見出しは「LIBRARY」「共有アーカイブ」と件数バッジ。右側に検索欄。
- タイトル・説明・登録者を対象に大文字小文字を区別せず検索する。
- タブは「すべて」「一般公開」、ログイン時のみ「自分の登録」。検索とタブ絞り込みを併用できる。
- 新しい投稿を先頭にする。
- カードにはZIPアイコン、公開範囲バッジ、タイトル、説明、ダウンロード名、投稿者、日付、容量、ダウンロードボタンを表示する。日付は日本語形式、容量はMBで小数2桁。
- 説明未入力時は「説明はありません。」。長いファイル名や説明で横幅を崩さない。
- 所有者だけに「編集」「登録情報を削除」を表示する。
- 新規登録のファイル選択部分は大きい破線枠と上向き矢印を使い、クリックでファイルを選択する。選択個数と名前を表示する。モード変更時には選択をクリアする。
- 認証は中央のモーダルでログイン／新規登録を切り替える。
- 読み込み中、初期読み込み失敗と再試行、データなし、検索該当なしを区別する。
- 通信中は該当操作を無効化して二重送信を防ぐ。
- 登録、認証、編集、削除、ダウンロード、通信エラーは右下トーストに集約する。約6秒で消し、閉じるボタンを付ける。
- ダウンロード成功の通知は「ZIPをブラウザーに渡しました。保存状況をご確認ください。」とし、端末への保存完了と断定しない。
- フッターは「zipshare.」「ひとつにまとめて、つながる。」。
- フォームラベル、フォーカス表示、ダイアログのrole/aria属性、通知のaria-liveを設定する。

## 7. API契約

入口は `/api?action=処理名`。JSON応答はUTF-8、エラーは `{ "message": "日本語の説明" }`。変更操作はPOST限定で、`X-CSRF-Token` を必須とする。

| action | メソッド | 入出力 |
| --- | --- | --- |
| session | GET | `{ user, csrf }`。未ログインのuserはnull |
| register | POST | JSONでname/email/password。成功時user/csrf/message |
| login | POST | JSONでemail/password。成功時user/csrf/message |
| logout | POST | セッション更新。csrf/message |
| users | GET | ログイン必須。`{ users: [{id,name}] }` |
| list | GET | `{ archives: [...] }`、新しいID順 |
| create | POST | multipart。files[]、mode、title、description、download_name、visibility、selected時allowed_user_ids[]。成功201 |
| update&id=ID | POST | JSONでtitle/description/download_name/visibility、selected時allowed_user_ids配列 |
| delete&id=ID | POST | 所有者限定。ZIPを保持してDB登録情報を削除 |
| download&id=ID | GET | 権限検証後にZIP配信 |

ユーザー型は `id, name, email`。一覧のアーカイブ型は `id, user_id, title, description, visibility, download_name, size, created_at, user_name, can_download, allowed_user_ids?` とする。サーバー上の保存名やパスは返さない。

入力不正400、未認証401、権限・CSRF違反403、対象なし404、メソッド違反405、メール重複409、予期しないエラー500を使い分ける。内部例外の詳細はログに記録し、レスポンスには出さない。

フロントのAPIクライアントでCSRFトークンを保持・更新する。FormDataのContent-Typeはブラウザーに任せる。非JSONレスポンスや通信失敗も日本語のエラーに変換する。ダウンロードはfetchで応答を検証し、Blob URLとdownload属性で保存を開始し、使用後にURLを解放する。

ZIPレスポンスは `application/zip`、Content-Length、`X-Content-Type-Options: nosniff`、`Cache-Control: private, no-store` を付ける。Content-DispositionではASCIIフォールバックと `filename*=UTF-8''...` を使用し、日本語名に対応する。配信前にセッションの書き込みロックを解放する。

## 8. DB・保存先・保護

- `storage/app.sqlite`、`storage/archives/`、`storage/sessions/`、`storage/tmp/` を使用する。必要なフォルダ・DB・テーブルは自動作成する。
- バックエンドは `APP_STORAGE` 環境変数による保存先変更に対応する。
- users：id、name、email UNIQUE、password（ハッシュ）。
- archives：id、user_id（users参照）、title、description、visibility（3種類のCHECK制約）、stored_name UNIQUE、download_name、size、created_at。
- archive_users：archive_id、user_idの複合主キー。archive_idは削除時CASCADE。
- SQLiteは外部キーを有効にし、busy_timeoutを5000msにする。SQLはプリペアドステートメントを使用する。
- public/membersだけを許可する旧archivesテーブルが存在する場合、投稿を保持したままselected対応へトランザクション内で移行する。新規DBにも対応する。
- セッションCookieはHttpOnly、SameSite=Lax、path=/、HTTPS時Secure。セッションのstrict modeを有効にする。
- CSRFトークンは暗号学的乱数で生成し、`hash_equals` で比較する。
- `storage/`、`.git/`、PHPソース、テスト、設定ファイルをHTTPで直接公開しない。
- PHP内蔵サーバーのルーターは `/api`、トップページ、ビルド済みフロントの静的ファイルだけを配信する。realpathで公開範囲を検証する。
- Apacheでは `public/` だけをDocumentRootとし、index.phpからルーターへ渡す。.htaccessでmod_rewriteを設定し、ディレクトリ一覧を無効化する。storageには直接アクセスを拒否する.htaccessを置く。

## 9. ファイル構成

責務を分け、最低限次の構成を作成する。

```text
front/
  package.json
  package-lock.json
  index.html
  vite.config.ts
  tsconfig*.json
  eslint.config.js
  scripts/dev.mjs
  src/
    main.tsx
    App.tsx
    App.css
    index.css
    api/client.ts
    components/AuthForm.tsx
    components/ArchiveForm.tsx
    types/archive.ts
php/
  bootstrap.php
  index.php
  archives.php
  router.php
public/
  index.php
  .htaccess
storage/
  .htaccess
scripts/build-deploy.mjs
tests/smoke.py
start.ps1
package.json
.gitignore
README.md
```

## 10. 開発・起動・配布

- frontで `npm run build` は `tsc -b && vite build`、`npm run lint` はESLintを実行する。
- `start.ps1` はプロジェクト直下でPHPを `127.0.0.1:8011` に起動し、ビルド済み画面とAPIを配信する。
- upload_tmp_dirはプロジェクト内のstorage/tmpにし、`upload_max_filesize=100M`、`post_max_size=110M`、`max_file_uploads=20`、`display_errors=0`、`log_errors=1` を設定する。
- start.ps1ではAPP_STORAGEをこのプロジェクトのstorageへ固定し、終了時に元の環境変数を復元する。
- frontで `npm run dev` を実行すると、dev.mjsがPHPとViteをまとめて起動する。PHPの設定と保存先はstart.ps1と揃える。
- 8011番ポートの使用を先にチェックする。PHPの起動を確認してからViteを起動し、起動失敗やタイムアウト時は終了する。Ctrl+Cや子プロセス終了時には両方停止し、プロセスを残さない。
- Viteは `/api` を `http://127.0.0.1:8011` にプロキシする。preview単体ではAPIを利用できないことをREADMEに記載する。
- ルートの `npm run build:deploy` でフロントをビルドし、成功後に実行PCのローカル日時で `release/YYYY-MM-DD/HH-mm-ss-SSS/` を作る。
- リリースには `front/dist/`、`php/`、publicからコピーした `public_html/`、`storage/.htaccess`、`DEPLOY.txt` を入れる。
- 開発用DB、アップロードZIP、セッション、node_modulesは含めない。
- 同日・同時刻でも既存リリースを上書きしない。`.reserved` で名前を予約し、`.building` に生成してから完了名へrenameする。衝突時は連番を付ける。
- DEPLOY.txtに4ディレクトリを同じ親へ配置すること、DocumentRootはpublic_htmlであること、PHP設定、storage書き込み権限、更新時に既存storageを保持することを記載する。
- .gitignoreではnode_modules、dist、release、ログ、storage内の実データ、ZIPを除外し、storage/.htaccessは管理する。
- READMEに必要環境、初回セットアップ、開発・ビルド・起動コマンド、Laragon VirtualHost例、API、検証、停止中にDBとarchivesをセットでバックアップする方法を説明する。
- 外部公開時のHTTPS設定と、メール確認・パスワード再発行・ウイルススキャン・アカウント容量制限・ログイン回数制限は実装対象外であること、削除後もZIPの保管容量管理が必要なことをREADMEに記載する。

## 11. 検証と完成条件

Python標準ライブラリだけで実行できる `tests/smoke.py` を作る。一時ディレクトリ・一時DB・空きポートでPHPを起動し、既存のユーザーやZIPに触れずにHTTP経由で検証する。終了時はサーバーと一時環境を片付ける。

最低限の検証対象：

- 登録、ログイン、ログアウト、パスワードのハッシュ保存、CSRF拒否。
- 複数ファイルのZIP化、既存ZIP登録、偽ZIP拒否、同名ファイルの保持、日本語ダウンロード名。
- 未ログインの投稿拒否と、一覧の閲覧許可。
- publicは誰でも、membersはログイン済み全員、selectedは投稿者と選択ユーザーだけが取得できること。
- 複数ユーザー選択、対象解除、selectedから他の公開範囲への変更、不正な選択IDの拒否。
- 非所有者にallowed_user_idsが返らないこと。
- 投稿者のみ編集・削除できること。
- 登録情報削除後もZIP本体が残り、一覧から消え、ダウンロードは404になること。
- storage、PHPソース、.git、testsを直接HTTP取得できないこと。

実装後、実行可能な環境では以下を実行し、問題があれば修正する。

```powershell
npm --prefix front run build
npm --prefix front run lint
python tests/smoke.py
Get-ChildItem php/*.php,public/*.php | ForEach-Object { php -l $_.FullName }
npm run build:deploy
```

リリースに実データが混入していないことも確認する。ブラウザーを使える場合はPCとスマートフォン幅で画面を確認する。未実行の検証は成功したと報告しない。

最後に、完成した機能、実行した検証と結果、起動コマンド、配布フォルダの場所、残っている環境上の制約があれば簡潔に報告してください。

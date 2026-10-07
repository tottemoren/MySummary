# 02. リクエストの流れと Spring Boot の基本の仕組み

[← 目次に戻る](README.md)

Spring Boot の4層（Controller・Service・Entity・Repository）の**周りにある4つの仕組み**を整理する。
どの言語・フレームワークにも同じ役割のものがあるので、ここを押さえると他の言語も読める。

※ 「Rebuildでの例」は 2026-10-07 時点の Rebuild（RebuildJava）のコード。

---

## 全体の位置関係

```
お客さんの注文（リクエスト）
   ↓
① ルーティング        … どの担当者に回すか決める
   ↓
② Filter / Interceptor … 全員に共通のチェック（入店時の確認）
   ↓
Controller → Service → Repository → DB
   ※ ③ DI … 各担当者に必要な道具を店が配っておく仕組み
   ※ ④ マイグレーション … DBの棚の構造を変えるときの工事記録
```

---

## ① ルーティング（`@GetMapping` など）

**「このURLにこの種類のリクエストが来たら、このメソッドが担当する」という対応表。**
たとえ：受付係の振り分け。

```java
@RestController
@RequestMapping("/stories")      // このクラスの担当は /stories から始まるURL
public class StoryController {

    @PostMapping                 // POST /stories が来たら
    public Story createStory(...)

    @GetMapping                  // GET /stories が来たら
    public List<Story> getStories()
}
```

- `@` から始まるものは**アノテーション**。「このメソッドにはこういう役割がある」とSpringに伝える目印
- クラスの `@RequestMapping` とメソッドの `@PostMapping` を**つなげたもの**が実際のURL
- 同じURLでも、メソッド（GET/POST）によって担当が変わる

**他の言語**: Rails は `routes.rb`、Laravel は `routes/api.php` に**1つのファイルにまとめて書く**。Springはメソッドの上に1つずつ書く。やっていることは同じ。

**Rebuildでの例**: `StoryController`。URLが `/api/...` で始まるもの（`/api/folders`）と始まらないもの（`/stories`、`/dialogues`、`/login`）が混在している。**URLの命名ルールをそろえることもAPI設計の一部**。

---

## ② Filter / Interceptor（他の言語では「ミドルウェア」）

**どのリクエストにも共通して行う処理を、Controllerの手前でまとめて行う仕組み。**
たとえ：入口のスタッフ。どの料理を頼むお客さんでも入店時に確認する。

よく使う例：

- **ログイン確認**（していない人はここで止める）
- **ログ記録**（誰が、いつ、どのURLに）
- **処理時間の計測**

```java
// イメージ：すべてのリクエストで、Controllerより先に実行される
public boolean preHandle(HttpServletRequest request, ...) {
    if (ログインしていない) {
        return false;  // ここで止める。Controllerまで行かない
    }
    return true;       // 次（Controller）へ進む
}
```

これがないと、全Controllerの先頭に「ログイン確認」を書くことになり、**1つ書き忘れただけでセキュリティの穴**になる。

Filter と Interceptor の違い（Springの外側か内側か）は細かい話なので、まずは**「Controllerの手前で共通処理をするもの」**と覚えればよい。

**Rebuildでの例**: `config/SecurityConfig.java` の `SecurityFilterChain`。Spring Security が用意した Filter の並び（チェーン）。
ただし `anyRequest().permitAll()`（全員通してよい）なので、**入口のスタッフはいるが全員素通り**の状態。→ [04 セキュリティ](04_セキュリティの基本.md)

---

## ③ DI（依存性の注入、`@Autowired`）

**クラスが必要とする他のクラスを、自分で `new` せず、外から渡してもらう仕組み。**
たとえ：料理人が自分で包丁を作らず、店が用意した包丁を渡してもらう。

```java
// DIなし：自分で作る
public class ArticleController {
    private ArticleService service = new ArticleService();
}

// DIあり（コンストラクタ方式）：Springが作って渡してくれる
public class ArticleController {
    private final ArticleService service;

    public ArticleController(ArticleService service) {
        this.service = service;
    }
}
```

**なぜそうするのか**

- **差し替えが簡単**：テストでは本物のDBにつながない偽物を渡せる
- **1つを使い回せる**：アプリ全体で1つだけ作って共有できる

**書き方は2種類**

| 書き方 | 例 | 評価 |
|---|---|---|
| フィールドに `@Autowired` | `@Autowired private StoryService storyService;` | 古い書き方 |
| **コンストラクタで受け取る** | 上のコード | **今の主流**。`final` にでき、テストで偽物を渡しやすい。コンストラクタが1つなら `@Autowired` は省略可 |

**他の言語**: Laravel には似た仕組み（サービスコンテナ）がある。Go は**自分の手で引数で渡す**。Rails にはほぼなく、規約で自動的につながる。

**Rebuildでの例**: `StoryController`・`DialogueController` はフィールド `@Autowired`、`FolderController`・`UserController` はコンストラクタ方式で、**2種類が混在**している。
また `FolderController` は Service を通さず Repository を直接使っている（Story は3段、Folder は2段）。**どこまで層を分けるかに唯一の正解はなく、チームでそろえることが大事**。

---

## ④ マイグレーション（Flyway / Liquibase）

**DBの構造（テーブルや列）の変更を、番号付きのファイルとして記録し、順番に適用する仕組み。**
Flyway や Liquibase はそのためのツール名。たとえ：店の改装の工事記録。

```
V1__create_articles.sql   … 記事テーブルを作る
V2__add_tags.sql          … タグの列を追加する
V3__create_comments.sql   … コメントテーブルを作る
```

Flyway は「このDBにはV2まで適用済み」と記録しておき、**まだのもの（V3）だけ**を実行する。

**なぜ必要か**

- 自分のPC・チームメンバーのPC・本番サーバーの**DB構造をそろえられる**
- 「いつ、どんな変更をしたか」の**履歴が残る**（GitのDB版）
- 手作業で本番DBを変えると、ミスや「誰が何をしたか分からない」状態になる

**他の言語**: Rails・Laravel には最初からあり、`db/migrate/` や `database/migrations/` に日時付きファイルが並ぶ。Go は外部ツール（golang-migrate など）。

**Rebuildでの例**: Flyway は未導入。代わりに `application.properties` の `spring.jpa.hibernate.ddl-auto=update`。
これは**起動のたびに Entity を見て、テーブルを自動で作ったり列を追加したりする**設定。開発初期は楽だが：

- **履歴が残らない**
- **追加しかしない**（列の削除・名前変更は反映されず、古い列が残る）
- **本番では危険**（起動しただけで本番DBの構造が変わる）

→ バックエンドを本番に出す前に、Flyway などに切り替えるのが一般的。

---

## まとめ

| 仕組み | ひとことで | たとえ | Rebuildの状態 |
|---|---|---|---|
| ① ルーティング | URLと担当メソッドの対応表 | 受付係 | 使用中（URL命名が不統一） |
| ② Filter / Interceptor | 全リクエスト共通の事前チェック | 入口での確認 | あるが全員通している |
| ③ DI | 必要な部品を外から渡してもらう | 店が包丁を配る | 使用中（書き方が2種類） |
| ④ マイグレーション | DB構造の変更履歴を順番に適用 | 改装の工事記録 | 未導入（`ddl-auto` で代用） |

Spring Boot ではアノテーションを付けるだけで動くので、**意識しなくても①〜③はすでに使っている**。仕組みが見えにくかっただけ。

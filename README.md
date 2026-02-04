---
title: Spring Boot と Flyway で実現する DB マイグレーション - DB を壊さず開発から運用まで行う方法
---

# Spring Boot と Flyway で実現する DB マイグレーション - DB を壊さず開発から運用まで行う方法

## Spring Initializr

[これ](https://start.spring.io/#!type=maven-project&language=java&platformVersion=4.0.2&packaging=jar&configurationFileFormat=yaml&jvmVersion=21&groupId=dev.mikoto2000.springboot&artifactId=flyway&name=flyway&description=Demo%20project%20for%20Spring%20Boot%20and%20Flyway&packageName=dev.mikoto2000.springboot.flyway&dependencies=devtools,lombok,web,data-jdbc,h2,flyway)


## DB 接続情報の作成

`application.yaml` に、データソース定義を追加します。

マイグレーションを体験するためには、 Spring Boot アプリケーションの起動ごとに DB の初期化がされてしまうと困ります。
そのため今回は、 H2 DB を利用し、 `./target/workshopdb.mv.db` という名前の DB ファイルを作成し、そこでデータを管理します。

```yaml
spring:
  application:
    name: flyway

  # 追加ここから
  datasource:
    url: ${DATASOURCE_URL:jdbc:h2:file:./target/workshopdb}
    username: ${DATASOURCE_USERNAME:sa}
    password: ${DATASOURCE_PASSWORD:}
    driver-class-name: org.h2.Driver
  # 追加ここまで
```

H2 DB は、デフォルトでユーザー名 `sa`, パスワード無しでログインできるため、そのように設定しています。


## 起動確認

この時点で正常に起動できるか確認しましょう。

次のコマンドを実行し、 Spring Boot アプリケーションが正常に起動することを確認してください。

```sh
./mvnw spring-boot:run
```


## H2 コンソールの確認

Spring Boot アプリケーションを起動した状態で `https://localhost:8080/h2-console` に接続すると、 H2 コンソールを確認できます。

H2 コンソールでは、 H2 が管理している DB の状態が確認できます。
今回は、 DB マイグレーションの結果をこの H2 コンソールを用いて確認していきます。

H2 コンソールのログイン画面が開くので、 `application.yaml` に記載したものと同じ接続情報を入力し、 `Connect` ボタンを押してください。


## 初期スキーマの作成

`src/main/resources/db/migration` に DDL を格納することで、 Spring Boot アプリケーション起動時に Flyway が走り、マイグレーションを行ってくれます。
(`src/main/resources/db/migration` は Flyway のデフォルト探索パス)

今回は `ACCOUNT` テーブルを作ってみましょう。

`src/main/resources/db/migration/V20260203.01__add_account_table.sql`:

```sql
CREATE TABLE ACCOUNT (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(50),
  password VARCHAR(255)
);
```

これだけで、マイグレーションの準備は完了です。


## Spring Boot アプリケーションの起動とスキーマ確認

それではもう一度 Spring Boot アプリケーションを実行してみましょう。

```sh
./mvnw spring-boot:run
```

起動ログに、次のようにマイグレーションの結果が表示されます。

```
2026-02-04T10:45:41.784Z  INFO 170450 --- [flyway] [  restartedMain] o.f.c.i.s.JdbcTableSchemaHistory         : Schema history table "PUBLIC"."flyway_schema_history" does not exist yet
2026-02-04T10:45:41.786Z  INFO 170450 --- [flyway] [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 1 migration (execution time 00:00.006s)
2026-02-04T10:45:41.789Z  INFO 170450 --- [flyway] [  restartedMain] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table "PUBLIC"."flyway_schema_history" ...
2026-02-04T10:45:41.808Z  INFO 170450 --- [flyway] [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema "PUBLIC": << Empty Schema >>
2026-02-04T10:45:41.811Z  INFO 170450 --- [flyway] [  restartedMain] o.f.core.internal.command.DbMigrate      : Migrating schema "PUBLIC" to version "20260203.01 - add account table"
2026-02-04T10:45:41.821Z  INFO 170450 --- [flyway] [  restartedMain] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema "PUBLIC", now at version v20260203.01 (execution time 00:00.003s)
2026-02-04T10:45:41.914Z  INFO 170450 --- [flyway] [  restartedMain] o.s.b.h.a.H2ConsoleAutoConfiguration     : H2 console available at '/h2-console'. Database available at 'jdbc:h2:file:./
```

ログ上では成功しているので、 `http://localhost:8080/h2-console` にアクセスし、テーブル定義が存在するか確認してみましょう。

テーブル一覧にテーブル `ACCOUNT` が存在していることを確認してください。

また、テーブル `flyway_schema_history` が存在していることに気付いた方もいるかもしれません。
このテーブルは、 Flyway が行ったマイグレーションの履歴を管理するためのテーブルです。
消したり手動で変更したりしないようにしましょう。

このように、適用したマイグレーションファイルを Flyway が管理するため、
マイグレーションが完了したら、作成したマイグレーションファイルを編集しないようにしましょう。

間違いや仕様変更でスキーマが変更されたら、新しいマイグレーションファイルを追加して対応していきます。


## スキーマの更新ファイル作成

それではさっそく、「仕様変更でテーブル `ACCOUNT` にカラム `ROLE` が必要になった」という想定で、
新しいマイグレーションファイルを作成してみましょう。

`src/main/resources/db/migration/V20260203.02__add_role_column_to_account_table.sql`:

```sql
ALTER TABLE ACCOUNT ADD COLUMN role VARCHAR(50);
```

ALTER 文でカラム `ROLE` を追加するマイグレーションファイルを新規作成しました。


## Spring Boot アプリケーションの起動と更新したスキーマの確認

それではもう一度 Spring Boot アプリケーションを実行し、
H2 コンソールでカラムが追加されているか確認してみましょう。

Spring Boot 起動時に Flyway が自動実行されるため、新しいマイグレーションが適用されます。

カラムが追加されていることが確認できれば OK です。


## 並行開発に向けたマイグレーション運用例紹介

TODO: draft に作ってレビュー後 migration に移動させてマージする運用の説明


## まとめ


---
title: 'Prisma + Supabaseで起こるエラー prepared statement "s0" already existsの対処法'
emoji: "⛔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "supabase", "prisma", "vercel"]
published: true
---

## 概要

Next.js + Prisma 環境で Supabase のデータベースを使用し、Vercel にデプロイした際に発生する Prisma クライアントエラー code: "42P05", message: "prepared statement \"s0\" already exists" の対処法を紹介します。

## エラーの詳細

ローカル環境で問題なく Supabase からのデータ取得ができたにも関わらず、Vercel にデプロイすると以下のエラーが表示され Supabase からのデータ取得に失敗しました。

```
Error [PrismaClientUnknownRequestError]:
Invalid `prisma.departure.findMany()` invocation:


Error occurred during query execution:
ConnectorError(ConnectorError { user_facing_error: None, kind: QueryError(PostgresError { code: "42P05", message: "prepared statement \"s0\" already exists", severity: "ERROR", detail: None, column: None, hint: None }), transient: false })
    at async r (.next/server/app/api/[api名]) {
  clientVersion: '6.10.1'
}
```

## 原因

Prisma は同じクエリを効率的に実行するため、一度 SQL をコンパイルして「prepared statement」を作成し、サーバー側でキャッシュします。

このエラーは、既に存在する`s0`という名前のステートメントがキャッシュに残っている状態で、再度同じ名前で作成を試みたために発生しています。

## 解決手順

Supabase プロジェクトのデータベースを再起動することでキャッシュを消去し解消することができます。
手順は以下の通りです。

1. Supabase ダッシュボードで対象プロジェクトを開く。
2. 左メニューから **Settings > General** を選択。
3. **Restart Project** の項目で **Fast database reboot** をクリック。
4. 再起動が完了するまで待機する。

再起動後、正常にデータ取得が行えるようになります。

## 参考リンク

- [Stack Overflow: Prisma Postgres error "prepared statement 's0' already exists" の回答](https://stackoverflow.com/questions/71026259/prisma-postres-error-prepared-statement-s0-already-exists)
- [Prisma docs: prepared-statement-caching](https://www.prisma.io/docs/orm/overview/databases/postgresql#prepared-statement-caching)

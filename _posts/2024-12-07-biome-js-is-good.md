---
layout: post
title: "Biome.jsめっちゃいいぞ"
date: 2024-12-07 09:00:00 +0900
categories: car
---

この記事は、[貴族会 Advent Calendar 2024](https://qiita.com/advent-calendar/2024/kizokukai) 7 日目の記事です。

たまには技術の話をしよう

## 導入のきっかけ

現在私は、マッチングプラットフォームを運営する企業に参画している。

主な仕事は、Ruby on Rails でできたレガシーシステムをモダンフロントエンドに置き換えていくことである。

その中で、ESLint の v9 から始まった flatConfig に苦しめられ、逃げるように Biome.js を入れてみたところいいことしかなく、この記事はその紹介である。

## 1. 設定がシンプル

- ESLint の v9 が登場し、FlatConfig のみをサポート対象にすると発表
  - しかしいろんなライブラリがそこに追従できておらず、ユーザー側で頑張る必要あり
- とんでもなく複雑に
- それと比較すると Biome.js の config は非常にシンプル
- formatter と linter を兼ねているので Prettier の config が不要なのもヨシ

## 2. Rust 製なので爆速

- Rust 製のツールって速いらしいね
  - あの C 言語より速いらしい
  - 私は難しいことはわからん
- どれくらい速いかというと…
  - 直近のの ci 結果に基づく
  - 対象ファイルは 48 ファイル
    - ESLint(8s) + Prettier(3s) = 11s
    - Biome.js = 1s(133ms)
  - このファイル量でこの差がつくなら、中長期目線でかなり良いはず

## 3. TypeScript や TSX を標準サポート

- 最近の Web フロント開発では、React + TypeScript は当たり前
  - Vue.js…？知らない子です
  - <https://2023.stateofjs.com/ja-JP/libraries/front-end-frameworks/>
- ESLint では plugin を入れて対応する必要がある
- 正直環境構築なんて年に数回しかしないから毎回忘れる
- Biome.js では標準サポート

## 4. いいね数爆増中

[![Star History Chart](https://api.star-history.com/svg?repos=eslint/eslint,biomejs/biome,prettier/prettier&type=Date)](https://star-history.com/#eslint/eslint&biomejs/biome&prettier/prettier&Date)

## 5. 細かいところに手が届く

- どう直すといいか教えてくれたり
- schema 情報があるので Config の補完が効いたり

## 最後に

とにかくいいことづくしで、今のところ欠点が見当たりません。
ぜひ導入してみてください。

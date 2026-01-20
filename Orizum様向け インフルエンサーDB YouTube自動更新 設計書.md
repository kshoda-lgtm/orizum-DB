# Orizum様向け インフルエンサーDB YouTube自動更新 設計書

**バージョン:** 1.0
**作成日:** 2026年1月20日
**作成者:** Manus AI

## 1. 概要

本ドキュメントは、スプレッドシートで管理されるインフルエンサーDBのデータ鮮度を維持するため、YouTubeの主要指標を自動で更新する機能（Phase 1）の設計を定義するものです。

## 2. 目的

-   一度登録されたインフルエンサーの指標（登録者数など）が時間経過と共に古くなることを防ぎ、DBの価値を維持する。
-   手動での情報更新作業を撤廃し、運用コストを削減する。
-   営業担当が常に最新のデータに基づいてアプローチ判断を行えるようにする。

## 3. システムアーキテクチャ（推奨構成）

自動更新は、サーバーレス環境で実行されるスクリプトによって実現することを推奨します。これにより、インフラ管理の手間を最小限に抑え、スケーラビリティを確保できます。

-   **実行環境:** Google Cloud Functions や AWS Lambda などのサーバーレスプラットフォーム
-   **トリガー:** Google Cloud Scheduler や Amazon EventBridge を使用し、毎日定時にスクリプトを起動
-   **使用API:**
    -   **Google Sheets API:** スプレッドシートの読み書きに使用 [1]
    -   **YouTube Data API v3:** YouTubeのチャンネル情報取得に使用 [2]
-   **開発言語:** Python (推奨) または Node.js

## 4. 更新対象の指標

スプレッドシート定義書に基づき、以下の指標を自動更新の対象とします。

| 列名 (日本語) | 列名 (英語) | API取得項目 | 説明 |
| :--- | :--- | :--- | :--- |
| チャンネルID | `youtube_channel_id` | `Channel.id` | チャンネルURLから解決し、初回取得時に一度だけ書き込む。以降のAPI呼び出しのキーとして使用。 |
| 登録者数 | `subscriber_count` | `Channel.statistics.subscriberCount` | チャンネルの現在の登録者数。 |
| 取得ジャンル | `youtube_genre` | `Channel.topicDetails.topicCategories` | YouTubeが公式に分類しているトピックカテゴリ。複数取得できる場合がある。 |

## 5. 更新プロセスフロー

自動更新スクリプトは、以下の順序で処理を実行します。

1.  **スプレッドシート読み込み:** Google Sheets API を使用して、DBシートから全レコードの `channel_url` と `youtube_channel_id` を読み込む。
2.  **チャンネルIDの解決 (初回のみ):** レコードに `youtube_channel_id` がまだ存在しない場合、`channel_url` を基にYouTube Data APIを呼び出し、チャンネルIDを特定してシートに書き込む。
3.  **バッチ処理:** 全ての `youtube_channel_id` をリスト化し、YouTube Data API の `Channels: list` エンドポイントを呼び出す。APIは一度に最大50件のIDをまとめて処理できるため、効率的に情報を取得する。
4.  **データ抽出:** APIのレスポンスから `subscriber_count` と `youtube_genre` を抽出する。
5.  **スプレッドシート更新:** 抽出したデータを、対応するレコードの各列に書き込む。
6.  **ログ記録:** 各レコードの更新結果（成功または失敗）を `update_status`, `update_error`, `last_updated_at` 列に記録する。

## 6. 更新スケジュール

-   **頻度:** **毎日**
-   **実行時刻:** 日本時間 (JST) の深夜から早朝（例: 午前3:00）を推奨。システムの負荷が低く、日中の営業活動に影響を与えないため。

## 7. エラーハンドリングとログ記録

安定した運用のため、予期せぬエラーに適切に対処し、その内容を記録します。

| エラーケース | `update_status` | `update_error` に記録する内容 | 対処法 |
| :--- | :--- | :--- | :--- |
| チャンネルが見つからない | `not_found` | `Channel not found` | チャンネルが削除されたか、URLが間違っている可能性。手動での確認を促す。 |
| APIの利用上限超過 | `failed` | `API quota exceeded` | APIのクォータ上限に達した場合。クォータの引き上げ申請を検討。 |
| 不正なURL | `failed` | `Invalid URL format` | `channel_url` の形式が不正な場合。手動での修正を促す。 |
| その他のAPIエラー | `failed` | APIから返されたエラーメッセージ | ネットワークの問題や一時的なAPIの不調。次回の実行で解消される可能性が高い。 |

## 8. 費用に関する考慮事項 (YouTube Data API)

ご懸念のあったAPI利用料について、YouTube Data APIには無料の利用枠（クォータ）が設定されています。

-   **クォータの単位:** APIの各操作には「ユニット」というコストが設定されています。
-   **無料利用枠:** 1日あたり **10,000ユニット** まで無料で利用可能です [3]。
-   **今回のコスト計算:**
    -   チャンネル情報を取得する `Channels: list` のコストは **1ユニット** です。
    -   現在約1,000件のチャンネルがあり、毎日1回更新する場合、1日の消費コストは **約1,000ユニット** となります。

**結論として、現在の想定運用では無料利用枠を大幅に下回るため、API利用に関する追加費用は発生しない見込みです。** 将来的に対象チャンネル数が1万件に近づいた場合に、クォータの引き上げを検討する必要があります。

## 9. 参照

[1] Google Sheets API. Google for Developers. [https://developers.google.com/sheets/api](https://developers.google.com/sheets/api)
[2] YouTube Data API Overview. Google for Developers. [https://developers.google.com/youtube/v3/getting-started](https://developers.google.com/youtube/v3/getting-started)
[3] YouTube Data API (v3) - Quota and Usage. Google for Developers. [https://developers.google.com/youtube/v3/getting-started#quota](https://developers.google.com/youtube/v3/getting-started#quota)

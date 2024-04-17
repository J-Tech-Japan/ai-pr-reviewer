# AI-based PR reviewer and summarizer

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Overview

J-Tech-Japan/ai-pr-reviewerは、Azure OpenAIを使用するように修正されたCodeRabbit `ai-pr-reviewer`です。自分で展開したモデルを利用するため、より安全です。

CodeRabbit ai-pr-reviewerは、OpenAIの`gpt-3.5-turbo`および`gpt-4`モデルを使用したGitHubプルリクエストのためのAIベースのコードレビューアおよび要約ツールです。GitHubアクションとして使用するように設計されており、すべてのプルリクエストで実行およびコメントをレビューするように構成できます。

## Reviewer Features:

- **プルリクエストの要約**: プルリクエストの変更の要約とリリースノートを生成します。
- **行ごとのコード変更の提案**: 変更を行ごとにレビューし、コード変更の提案を提供します。
- **継続的でインクリメンタルなレビュー**: プルリクエスト内の各コミットでレビューを実行します。プルリクエスト全体の一度限りのレビューではありません。
- **高い費用効率とノイズの低減**: インクリメンタルレビューはOpenAIのコストを節約し、コミット間の変更されたファイルとプルリクエストのベースを追跡することでノイズを低減します。
- **サマリーには「軽量」モデルを使用**: 「軽量」の要約モデル（例：`gpt-3.5-turbo`）と「重量」のレビューモデル（例：`gpt-4`）と一緒に使用するように設計されています。_徹底的なコードレビューには強力な推論能力が必要なため、最適な結果を得るためには、「重量」モデルとして`gpt-4`を使用してください。_
- **チャットボット**: コードの行やファイル全体のコンテキストでボットとの会話をサポートし、コンテキストの提供、テストケースの生成、コードの複雑さの低減に役立ちます。
- **レビューのスキップ**: 初期設定では、単純な変更（例：誤字修正）や変更の大部分が良好に見える場合に、詳細なレビューをスキップします。`review_simple_changes`と`review_comment_lgtm`を`true`に設定することで無効にできます。
- **カスタマイズ可能なプロンプト**: `system_message`、`summarize`、`summarize_release_notes`プロンプトを調整して、レビュープロセスの特定の側面に焦点を当てたり、レビュー目標を変更したりできます。

このツールを使用するには、提供されたYAMLファイルをリポジトリに追加し、`GITHUB_TOKEN`や`OPENAI_API_KEY`などの必要な環境変数を設定する必要があります。使用方法、例、貢献、FAQについての詳細については、以下を参照してください。

- [Overview](#overview)
- [Professional Version of CodeRabbit](#professional-version-of-coderabbit)
- [Reviewer Features](#reviewer-features)
- [Install instructions](#install-instructions)
- [Conversation with CodeRabbit](#conversation-with-coderabbit)
- [Examples](#examples)
- [Contribute](#contribute)
- [FAQs](#faqs)

## Install instructions

`ai-pr-review` はGitHub Actionで動作します。以下のファイルをリポジトリに追加してください。`.github/workflows/ai-pr-reviewer.yml`

```yaml
name: Code Review

permissions:
  contents: read
  pull-requests: write

on:
  pull_request:
  pull_request_review_comment:
    types: [created]

concurrency:
  group:
    ${{ github.repository }}-${{ github.event.number || github.head_ref ||
    github.sha }}-${{ github.workflow }}-${{ github.event_name ==
    'pull_request_review_comment' && 'pr_comment' || 'pr' }}
  cancel-in-progress: ${{ github.event_name != 'pull_request_review_comment' }}

jobs:
  review:
    if: github.actor != 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - uses: J-Tech-Japan/ai-pr-reviewer@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          AZURE_OPENAI_API_KEY: ${{ Secrets.AZURE_OPENAI_API_KEY }}
          AZURE_OPENAI_API_INSTANCE_NAME: ${{ Secrets.AZURE_OPENAI_API_INSTANCE_NAME }}
          AZURE_OPENAI_API_DEPLOYMENT_NAME: ${{ Secrets.AZURE_OPENAI_API_DEPLOYMENT_NAME }}
          AZURE_OPENAI_API_VERSION: '2023-07-01-preview'
        with:
          debug: true
          review_comment_lgtm: false
          openai_light_model: gpt-4
          openai_heavy_model: gpt-4
          language: ja-JP
```

#### Environment variables

- `GITHUB_TOKEN`: これはすでにGitHubアクション環境で利用可能です。これはプルリクエストにコメントを追加するために使用されます。
- `AZURE_OPENAI_API_KEY`: Azure OpenAI APIと認証するために使用します。GitHub Actionのシークレットにキーを追加してください。
- `AZURE_OPENAI_API_INSTANCE_NAME`: Azure OpenAI APIインスタンスにアクセスするために使用します。GitHub Actionのシークレットにインスタンス名を追加してください。
- `AZURE_OPENAI_API_DEPLOYMENT_NAME`: Azure OpenAI APIモデルを推論するために使用します。GitHub Actionのシークレットにデプロイメント名を追加してください。
- `AZURE_OPENAI_API_VERSION`: Azure OpenAI APIバージョンにアクセスするために使用します。

具体的な値については、Langchainの設定を参照してください。
https://js.langchain.com/docs/integrations/text_embedding/azure_openai

### Models: `gpt-4` and `gpt-3.5-turbo`

サマリーのような軽いタスクには`gpt-3.5-turbo`を使用し(`openai_light_model` in configuration)、より複雑なレビューやコメントタスクには`gpt-4`を使用することをお勧めします (`openai_heavy_model` in configuration)。

費用: `gpt-3.5-turbo`は非常に安価です。`gpt-4`は桁違いに高価ですが、優れた結果が得られます。通常、`gpt-4`ベースのレビューとコメントでは、1日あたり20ドルを20人の開発者チームに費やしています。

### Prompts & Configuration

See: [action.yml](./action.yml)

Tip: `system_message`の値を設定することで、ボットのパーソナリティを変更できます。たとえば、ドキュメントやブログ記事をレビューする場合は、次のプロンプトを使用できます。


<details>
<summary>Blog Reviewer Prompt</summary>

```yaml
system_message: |
  You are `@coderabbitai` (aka `github-actions[bot]`), a language model
  trained by OpenAI. Your purpose is to act as a highly experienced
  DevRel (developer relations) professional with focus on cloud-native
  infrastructure.

  Company context -
  CodeRabbit is an AI-powered Code reviewer.It boosts code quality and cuts manual effort. Offers context-aware, line-by-line feedback, highlights critical changes,
  enables bot interaction, and lets you commit suggestions directly from GitHub.

  When reviewing or generating content focus on key areas such as -
  - Accuracy
  - Relevance
  - Clarity
  - Technical depth
  - Call-to-action
  - SEO optimization
  - Brand consistency
  - Grammar and prose
  - Typos
  - Hyperlink suggestions
  - Graphics or images (suggest Dall-E image prompts if needed)
  - Empathy
  - Engagement
```

</details>

## Conversation with CodeRabbit

このアクションによって作成されたレビューコメントにリプライすることで、diffコンテキストに基づいた応答を取得できます。さらに、コメントでボットをタグ付けして会話に招待することができます（`@coderabbitai`）。


Example:

> @coderabbitai Please generate a test plan for this file.

Note: A review comment is a comment made on a diff or a file in the pull
request.

### Ignoring PRs

PRを無視して欲しいときもあるはずです。例えば、このアクションを使用してドキュメントをレビューしている場合、ドキュメントのみを変更するPRを無視することができます。PRを無視するには、PRの説明に次のキーワードを追加します。

```text
@coderabbitai: ignore
```

## Examples

Some of the reviews done by ai-pr-reviewer

![PR Summary](./docs/images/PRSummary.png)

![PR Release Notes](./docs/images/ReleaseNotes.png)

![PR Review](./docs/images/section-1.png)

![PR Conversation](./docs/images/section-3.png)

## Contribute

### Developing

> First, you'll need to have a reasonably modern version of `node` handy, tested
> with node 17+.

Install the dependencies

```bash
$ npm install
```

Build the typescript and package it for distribution

```bash
$ npm run build && npm run package
```

## FAQs

### Review pull requests from forks

GitHub Actions limits the access of secrets from forked repositories. To enable
this feature, you need to use the `pull_request_target` event instead of
`pull_request` in your workflow file. Note that with `pull_request_target`, you
need extra configuration to ensure checking out the right commit:

```yaml
name: Code Review

permissions:
  contents: read
  pull-requests: write

on:
  pull_request_target:
    types: [opened, synchronize, reopened]
  pull_request_review_comment:
    types: [created]

concurrency:
  group:
    ${{ github.repository }}-${{ github.event.number || github.head_ref ||
    github.sha }}-${{ github.workflow }}-${{ github.event_name ==
    'pull_request_review_comment' && 'pr_comment' || 'pr' }}
  cancel-in-progress: ${{ github.event_name != 'pull_request_review_comment' }}

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: coderabbitai/ai-pr-reviewer@latest
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        with:
          debug: false
          review_simple_changes: false
          review_comment_lgtm: false
```

See also:
https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request_target

### Inspect the messages between OpenAI server

Set `debug: true` in the workflow file to enable debug mode, which will show the
messages

### Disclaimer

- Your code (files, diff, PR title/description) will be sent to OpenAI's servers
  for processing. Please check with your compliance team before using this on
  your private code repositories.
- OpenAI's API is used instead of ChatGPT session on their portal. OpenAI API
  has a
  [more conservative data usage policy](https://openai.com/policies/api-data-usage-policies)
  compared to their ChatGPT offering.
- This action is not affiliated with OpenAI.

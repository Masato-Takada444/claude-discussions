# Phase 1 実装チケット詳細仕様

## T-01: [スパイク] Railsからのmastra呼び出し検証 (HTTP API経由)

### 概要
mastraワークフローエンジンとRailsアプリケーションの統合方法を検証し、技術的実現可能性を確認する。

### 受け入れ条件
- [ ] mastraをNode.jsサーバーとして起動できる
- [ ] RailsからHTTP経由でmastraのワークフローを実行できる
- [ ] CSVデータの受け渡しが正しく動作する
- [ ] エラーハンドリングの基本パターンが確立されている
- [ ] 実装方法のドキュメントが作成されている

### 技術詳細
```ruby
# app/services/mastra_client.rb の例
class MastraClient
  include HTTParty
  base_uri ENV['MASTRA_API_URL'] || 'http://localhost:4000'

  def execute_workflow(workflow_id, payload)
    response = self.class.post("/workflows/#{workflow_id}/execute", 
      body: payload.to_json,
      headers: { 'Content-Type' => 'application/json' }
    )
    
    case response.code
    when 200..299
      JSON.parse(response.body)
    else
      raise MastraError, "Workflow execution failed: #{response.body}"
    end
  end
end
```

### タスク
1. mastraのセットアップとHello Worldワークフローの作成
2. Node.js APIサーバーの実装（Express.js）
3. Rails側のHTTPクライアント実装
4. 統合テストの作成
5. ドキュメント作成

### 見積もり
- 工数: 2-3日
- 優先度: 最高（技術的リスクの解消）

---

## T-02: [インフラ] AWS基本インフラ構築 (VPC, S3, IAM)

### 概要
tokium-agent-infra/modules/expense-inspect/にTerraformモジュールを作成し、必要なAWSリソースを構築する。

### 受け入れ条件
- [ ] VPCとネットワーク設定が完了している
- [ ] S3バケットが作成され、適切なライフサイクルポリシーが設定されている
- [ ] IAMロールとポリシーが最小権限の原則に従って設定されている
- [ ] Terraformコードがtokium-agent-infraの規約に従っている
- [ ] terraform planが正常に実行される

### 技術詳細
```hcl
# modules/expense-inspect/s3.tf
resource "aws_s3_bucket" "inspection_files" {
  bucket = "${var.environment}-expense-inspect-files"
  
  lifecycle_rule {
    enabled = true
    
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 365
      storage_class = "GLACIER"
    }
    
    expiration {
      days = 1825  # 5年
    }
  }
}

# modules/expense-inspect/iam.tf
resource "aws_iam_role" "expense_inspect_task" {
  name = "${var.environment}-expense-inspect-task"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}
```

### タスク
1. Terraformモジュール構造の作成
2. ネットワークリソース（VPC、サブネット）の定義
3. S3バケットとライフサイクルポリシーの設定
4. IAMロールとポリシーの作成
5. 出力値の定義

### 見積もり
- 工数: 1-2日
- 優先度: 高（後続タスクのブロッカー）

---

## T-03: [バックエンド] Rails 7 + ActiveJob (DelayedJob) セットアップ

### 概要
Rails 7アプリケーションの基本構造を作成し、ActiveJobとDelayed Jobを設定する。

### 受け入れ条件
- [ ] Rails 7.1アプリケーションが作成されている
- [ ] Delayed Jobがインストールされ、ActiveJobバックエンドとして設定されている
- [ ] inspectionsテーブルのマイグレーションが作成されている
- [ ] 基本的なモデルとバリデーションが実装されている
- [ ] テスト環境が設定されている

### 技術詳細
```ruby
# Gemfile
gem 'rails', '~> 7.1.0'
gem 'delayed_job_active_record'
gem 'pg'
gem 'aws-sdk-s3'

# config/application.rb
config.active_job.queue_adapter = :delayed_job

# db/migrate/xxx_create_inspections.rb
class CreateInspections < ActiveRecord::Migration[7.1]
  def change
    create_table :inspections, id: :uuid do |t|
      t.string :tokium_user_id, null: false
      t.string :status, null: false, default: 'accepted'
      t.string :original_filename, null: false
      t.string :input_file_path, null: false
      t.string :output_file_path
      t.text :notification_emails, null: false
      t.json :error_details
      t.datetime :completed_at
      t.timestamps
    end
    
    add_index :inspections, :tokium_user_id
    add_index :inspections, :status
    add_index :inspections, :created_at
  end
end

# app/models/inspection.rb
class Inspection < ApplicationRecord
  STATUSES = %w[accepted processing completed failed].freeze
  
  validates :tokium_user_id, presence: true
  validates :status, inclusion: { in: STATUSES }
  validates :original_filename, presence: true
  validates :input_file_path, presence: true
  validates :notification_emails, presence: true
  
  serialize :notification_emails, Array
end
```

### タスク
1. Rails 7.1アプリケーションの作成
2. 必要なGemのインストール
3. データベース設定
4. マイグレーションファイルの作成
5. モデルとバリデーションの実装
6. 基本的なテストの作成

### 見積もり
- 工数: 1日
- 優先度: 高（基盤となるため）

---

## T-04: [開発環境] Docker Composeによるローカル開発環境構築

### 概要
Rails、PostgreSQL、Redis（Delayed Job用）、mastra（Node.js）を含む統合開発環境を構築する。

### 受け入れ条件
- [ ] docker-compose.ymlが作成されている
- [ ] 全サービスが正常に起動する
- [ ] ホットリロードが動作する
- [ ] ローカルS3（MinIO）が設定されている
- [ ] セットアップ手順がREADMEに記載されている

### 技術詳細
```yaml
# docker-compose.yml
version: '3.8'

services:
  rails:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - bundle:/usr/local/bundle
    environment:
      DATABASE_URL: postgresql://postgres:password@db:5432/expense_inspect_development
      REDIS_URL: redis://redis:6379/0
      MASTRA_API_URL: http://mastra:4000
      AWS_ACCESS_KEY_ID: minioadmin
      AWS_SECRET_ACCESS_KEY: minioadmin
      S3_ENDPOINT: http://minio:9000
    depends_on:
      - db
      - redis
      - minio
      - mastra

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
    volumes:
      - postgres:/var/lib/postgresql/data

  redis:
    image: redis:7

  delayed_job:
    build: .
    command: bundle exec rake jobs:work
    volumes:
      - .:/app
      - bundle:/usr/local/bundle
    environment:
      DATABASE_URL: postgresql://postgres:password@db:5432/expense_inspect_development
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - db
      - redis

  mastra:
    build: ./mastra
    ports:
      - "4000:4000"
    volumes:
      - ./mastra:/app
      - node_modules:/app/node_modules
    environment:
      NODE_ENV: development

  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio:/data

volumes:
  postgres:
  bundle:
  node_modules:
  minio:
```

### タスク
1. Dockerfileの作成（Rails用、mastra用）
2. docker-compose.ymlの作成
3. 環境変数の設定
4. 初期化スクリプトの作成
5. READMEの作成

### 見積もり
- 工数: 1日
- 優先度: 高（開発効率に直結）

---

## T-05: [CI/CD] 基本的なCI/CDパイプライン構築

### 概要
GitHub ActionsまたはAWS CodePipelineを使用して、テスト実行とデプロイの自動化を設定する。

### 受け入れ条件
- [ ] プルリクエスト時にテストが自動実行される
- [ ] mainブランチへのマージ時にステージング環境へ自動デプロイされる
- [ ] Terraformの変更がある場合はterraform planが実行される
- [ ] セキュリティスキャンが実行される
- [ ] ビルド成果物がECRにプッシュされる

### 技術詳細
```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Ruby
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: '3.2'
        bundler-cache: true
    
    - name: Setup database
      env:
        DATABASE_URL: postgresql://postgres:password@localhost:5432/test
      run: |
        bundle exec rails db:create
        bundle exec rails db:schema:load
    
    - name: Run tests
      env:
        DATABASE_URL: postgresql://postgres:password@localhost:5432/test
      run: bundle exec rspec
    
    - name: Run security scan
      run: bundle exec brakeman

  build:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-1
    
    - name: Login to ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Build and push Docker image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/expense-inspect:$IMAGE_TAG .
        docker push $ECR_REGISTRY/expense-inspect:$IMAGE_TAG
```

### タスク
1. GitHub Actionsワークフローの作成
2. テスト実行ジョブの設定
3. セキュリティスキャンの追加
4. Docker イメージビルドとECRプッシュ
5. デプロイジョブの設定

### 見積もり
- 工数: 1-2日
- 優先度: 中（早期に設定することで継続的な品質担保）

---

## 実装順序と依存関係

```mermaid
graph LR
    T01[T-01: mastra検証] --> T04[T-04: Docker環境]
    T02[T-02: AWSインフラ] --> T05[T-05: CI/CD]
    T03[T-03: Rails設定] --> T04
    T04 --> T05
```

## Day 1-2のアクションプラン

1. **Day 1 AM**: T-01とT-03を並行して開始
   - mastra検証チーム: mastraのセットアップとAPIサーバー作成
   - Railsチーム: Rails 7アプリケーションの初期設定

2. **Day 1 PM**: T-02の開始
   - インフラチーム: Terraformモジュールの作成

3. **Day 2 AM**: T-04の実装
   - 全チーム: Docker環境の構築と統合テスト

4. **Day 2 PM**: T-01の結果レビューとT-05の開始
   - 技術判断: mastra統合の可否判定
   - CI/CDパイプラインの設定
# インフラストラクチャ設計

## 1. ディレクトリ構造

```
tokium-agent-infra/
├── environments/
│   ├── dev/
│   ├── stg/
│   └── prod/
└── modules/
    ├── expense-approval/   # 既存
    ├── travel/            # 既存
    └── expense-inspect/   # 新規追加
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        ├── s3.tf         # S3バケット（CSV保存）
        ├── ecs.tf        # ECSタスク定義（Rails + mastra）
        ├── rds.tf        # RDS（PostgreSQL）
        ├── alb.tf        # Application Load Balancer
        └── iam.tf        # IAMロール・ポリシー
```

## 2. 主要リソース

### 2.1 S3バケット

```hcl
# modules/expense-inspect/s3.tf
resource "aws_s3_bucket" "inspections" {
  bucket = "${var.environment}-expense-inspect-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_lifecycle_configuration" "inspections" {
  bucket = aws_s3_bucket.inspections.id
  
  rule {
    id     = "archive-old-data"
    status = "Enabled"
    
    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 365
      storage_class = "GLACIER"
    }
  }
}
```

### 2.2 RDS（PostgreSQL）

```hcl
# modules/expense-inspect/rds.tf
resource "aws_db_instance" "postgres" {
  identifier     = "${var.environment}-expense-inspect"
  engine         = "postgres"
  engine_version = "15"
  instance_class = var.db_instance_class
  
  allocated_storage     = 20
  max_allocated_storage = 100
  storage_encrypted     = true
  
  db_name  = "expense_inspect"
  username = "postgres"
  password = random_password.db_password.result
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  skip_final_snapshot = var.environment != "prod"
  deletion_protection = var.environment == "prod"
}
```

### 2.3 ECSタスク定義

```hcl
# modules/expense-inspect/ecs.tf
resource "aws_ecs_task_definition" "app" {
  family                   = "${var.environment}-expense-inspect"
  requires_compatibilities = ["FARGATE"]
  network_mode            = "awsvpc"
  cpu                     = var.task_cpu
  memory                  = var.task_memory
  execution_role_arn      = aws_iam_role.ecs_execution.arn
  task_role_arn           = aws_iam_role.ecs_task.arn
  
  container_definitions = jsonencode([
    {
      name  = "rails"
      image = "${aws_ecr_repository.app.repository_url}:${var.app_version}"
      
      portMappings = [{
        containerPort = 3000
        protocol      = "tcp"
      }]
      
      environment = [
        {
          name  = "RAILS_ENV"
          value = var.environment
        },
        {
          name  = "DATABASE_URL"
          value = "postgresql://${aws_db_instance.postgres.username}:${random_password.db_password.result}@${aws_db_instance.postgres.endpoint}/${aws_db_instance.postgres.db_name}"
        },
        {
          name  = "S3_BUCKET"
          value = aws_s3_bucket.inspections.id
        }
      ]
      
      secrets = [
        {
          name      = "OPENAI_API_KEY"
          valueFrom = aws_ssm_parameter.openai_api_key.arn
        },
        {
          name      = "GOOGLE_MAPS_API_KEY"
          valueFrom = aws_ssm_parameter.google_maps_api_key.arn
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = data.aws_region.current.name
          "awslogs-stream-prefix" = "rails"
        }
      }
    }
  ])
}
```

### 2.4 IAMロール

```hcl
# modules/expense-inspect/iam.tf
resource "aws_iam_role_policy" "ecs_task_s3" {
  name = "${var.environment}-expense-inspect-s3"
  role = aws_iam_role.ecs_task.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "${aws_s3_bucket.inspections.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:ListBucket"
        ]
        Resource = aws_s3_bucket.inspections.arn
      }
    ]
  })
}
```

## 3. 環境変数

```hcl
# modules/expense-inspect/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnet_ids" {
  description = "Private subnet IDs"
  type        = list(string)
}

variable "public_subnet_ids" {
  description = "Public subnet IDs"
  type        = list(string)
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "task_cpu" {
  description = "ECS task CPU units"
  type        = string
  default     = "256"
}

variable "task_memory" {
  description = "ECS task memory (MB)"
  type        = string
  default     = "512"
}
```

## 4. 環境別設定

```hcl
# environments/dev/main.tf に追加
module "expense_inspect" {
  source = "../../modules/expense-inspect"
  
  environment        = var.environment
  vpc_id            = var.vpc_id
  private_subnet_ids = var.private_subnet_ids
  public_subnet_ids  = var.public_subnet_ids
  
  # 開発環境は最小構成
  db_instance_class = "db.t3.micro"
  task_cpu         = "256"
  task_memory      = "512"
}
```

## 5. MVP段階での簡略化

### 5.1 最小構成
- RDS: db.t3.micro（開発環境）
- ECS: 256 CPU / 512 MB メモリ
- ALB: なし（MVP段階では直接アクセス）

### 5.2 後から追加
- CloudFront（CDN）
- WAF（セキュリティ）
- Auto Scaling
- マルチAZ構成

## 6. セキュリティ考慮事項

### 6.1 ネットワーク
- RDSはプライベートサブネット
- セキュリティグループで必要最小限のアクセス

### 6.2 データ保護
- S3バケット暗号化
- RDS暗号化
- SSMパラメータストアでシークレット管理

### 6.3 アクセス制御
- IAMロールによる最小権限
- S3バケットポリシー

## 7. 運用・監視

### 7.1 ログ
- CloudWatch Logs（30日保持）
- S3アクセスログ（別バケット）

### 7.2 メトリクス
- ECSメトリクス
- RDSメトリクス
- カスタムメトリクス（処理時間など）

### 7.3 アラート
- エラー率
- レスポンスタイム
- リソース使用率
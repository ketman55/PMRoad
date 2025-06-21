# バックアップ・リストア手順書

**タイトル**: バックアップ・リストア手順書  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: operation-procedures.md, incident-response-procedures.md  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システムのデータ保護とビジネス継続性を確保するため、バックアップおよびリストア手順を定義することを目的とする。

### 1.2 適用範囲
本手順は、以下のデータに適用される：
- データベース（PostgreSQL, Redis）
- アプリケーションファイル
- システム設定ファイル
- ユーザーアップロードファイル

### 1.3 設計方針
- **3-2-1ルール**: 3つのコピー、2つの異なるメディア、1つのオフサイト
- **自動化**: 人的ミスを防ぐための自動バックアップ
- **暗号化**: バックアップデータの暗号化保護
- **テスト**: 定期的なリストアテストによる実効性確認

## 2. バックアップ戦略

### 2.1 RPO・RTO目標

| システム | RPO | RTO | 重要度 |
|---|---|---|---|
| 業務データベース | 1時間 | 2時間 | Critical |
| ユーザーファイル | 24時間 | 4時間 | High |
| システム設定 | 24時間 | 1時間 | Medium |
| ログデータ | 24時間 | 8時間 | Low |

### 2.2 バックアップ種別

#### 2.2.1 バックアップタイプ

| タイプ | 実行頻度 | 保持期間 | 用途 |
|---|---|---|---|
| **フルバックアップ** | 週次（日曜日） | 3ヶ月 | 完全復旧 |
| **増分バックアップ** | 日次 | 1ヶ月 | 日次復旧 |
| **差分バックアップ** | 時間次（業務時間） | 1週間 | 迅速復旧 |
| **トランザクションログ** | リアルタイム | 1週間 | ポイントインタイム復旧 |

#### 2.2.2 保存場所

```
┌─────────────────────────────────────────────────────────┐
│                  バックアップ階層                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │   Tier 1    │    │   Tier 2    │    │   Tier 3    │ │
│  │ローカルディスク│    │ NASストレージ │    │クラウドストレージ│ │
│  │             │    │             │    │             │ │
│  │ ・高速復旧用 │    │ ・中期保存用 │    │ ・長期保存用 │ │
│  │ ・1-3日分   │    │ ・1ヶ月分   │    │ ・3ヶ月分   │ │
│  │ ・SSD       │    │ ・RAID6     │    │ ・S3/Azure  │ │
│  └─────────────┘    └─────────────┘    └─────────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 3. データベースバックアップ

### 3.1 PostgreSQL バックアップ

#### 3.1.1 フルバックアップ（週次）

```bash
#!/bin/bash
# postgresql_full_backup.sh

# 設定
DB_HOST="localhost"
DB_PORT="5432"
DB_USER="backup_user"
DB_NAME="main_db"
BACKUP_DIR="/backup/postgresql/full"
RETENTION_DAYS=90
DATE=$(date +%Y%m%d_%H%M%S)

# バックアップディレクトリ作成
mkdir -p $BACKUP_DIR

# フルバックアップ実行
echo "$(date): Starting full backup for $DB_NAME"
pg_dump -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
    --format=custom \
    --compress=9 \
    --verbose \
    --file=$BACKUP_DIR/full_backup_${DATE}.dump

# バックアップ結果確認
if [ $? -eq 0 ]; then
    echo "$(date): Full backup completed successfully"
    
    # バックアップファイルサイズ確認
    BACKUP_SIZE=$(du -h $BACKUP_DIR/full_backup_${DATE}.dump | cut -f1)
    echo "$(date): Backup size: $BACKUP_SIZE"
    
    # チェックサム計算
    sha256sum $BACKUP_DIR/full_backup_${DATE}.dump > $BACKUP_DIR/full_backup_${DATE}.dump.sha256
    
    # メタデータ保存
    cat > $BACKUP_DIR/full_backup_${DATE}.meta << EOF
backup_type=full
database=$DB_NAME
start_time=$(date -d '-10 minutes' --iso-8601=seconds)
end_time=$(date --iso-8601=seconds)
size=$BACKUP_SIZE
status=completed
EOF

    # 古いバックアップ削除
    find $BACKUP_DIR -name "full_backup_*.dump" -mtime +$RETENTION_DAYS -delete
    find $BACKUP_DIR -name "full_backup_*.meta" -mtime +$RETENTION_DAYS -delete
    find $BACKUP_DIR -name "full_backup_*.sha256" -mtime +$RETENTION_DAYS -delete
    
    # ログ記録
    echo "$(date): Full backup process completed" >> /var/log/backup.log
    
else
    echo "$(date): Full backup failed with exit code $?"
    echo "$(date): Full backup failed" >> /var/log/backup.log
    
    # アラート送信
    mail -s "PostgreSQL Full Backup Failed" admin@example.com << EOF
PostgreSQL full backup failed on $(hostname)
Date: $(date)
Database: $DB_NAME
Please check the system immediately.
EOF
fi
```

#### 3.1.2 増分バックアップ（日次）

```bash
#!/bin/bash
# postgresql_incremental_backup.sh

# 設定
WAL_ARCHIVE_DIR="/backup/postgresql/wal"
BACKUP_DIR="/backup/postgresql/incremental"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d)

# WALファイル管理
echo "$(date): Starting WAL archive management"

# 圧縮してアーカイブ
find $WAL_ARCHIVE_DIR -name "*.wal" -mmin +60 | while read walfile; do
    gzip "$walfile"
done

# 古いWALファイル削除
find $WAL_ARCHIVE_DIR -name "*.wal.gz" -mtime +$RETENTION_DAYS -delete

# ベースバックアップ（pg_basebackup使用）
if [ $(date +%u) -eq 1 ]; then  # 月曜日
    echo "$(date): Creating base backup"
    pg_basebackup -h localhost -U backup_user -D $BACKUP_DIR/base_$DATE -Ft -z -P
    
    if [ $? -eq 0 ]; then
        echo "$(date): Base backup completed"
        
        # 古いベースバックアップ削除
        find $BACKUP_DIR -name "base_*" -type d -mtime +7 -exec rm -rf {} \;
    else
        echo "$(date): Base backup failed"
        mail -s "PostgreSQL Base Backup Failed" admin@example.com
    fi
fi

echo "$(date): Incremental backup process completed" >> /var/log/backup.log
```

#### 3.1.3 ポイントインタイムリカバリ用設定

```sql
-- postgresql.conf 設定
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/postgresql/wal/%f'
archive_timeout = 300  -- 5分
max_wal_senders = 3
```

### 3.2 Redis バックアップ

#### 3.2.1 RDB バックアップ

```bash
#!/bin/bash
# redis_backup.sh

REDIS_HOST="localhost"
REDIS_PORT="6379"
REDIS_PASSWORD=""
BACKUP_DIR="/backup/redis"
DATE=$(date +%Y%m%d_%H%M%S)

# バックアップディレクトリ作成
mkdir -p $BACKUP_DIR

# RDB バックアップ
echo "$(date): Starting Redis backup"
redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_PASSWORD BGSAVE

# バックアップ完了待機
while [ $(redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_PASSWORD LASTSAVE) -eq $(redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_PASSWORD LASTSAVE) ]; do
    sleep 1
done

# RDBファイルコピー
cp /var/lib/redis/dump.rdb $BACKUP_DIR/redis_backup_${DATE}.rdb

# 圧縮
gzip $BACKUP_DIR/redis_backup_${DATE}.rdb

# 古いバックアップ削除
find $BACKUP_DIR -name "redis_backup_*.rdb.gz" -mtime +7 -delete

echo "$(date): Redis backup completed" >> /var/log/backup.log
```

## 4. ファイルバックアップ

### 4.1 アプリケーションファイルバックアップ

#### 4.1.1 rsync によるバックアップ

```bash
#!/bin/bash
# application_file_backup.sh

# 設定
SOURCE_DIRS=(
    "/opt/application"
    "/var/www/uploads"
    "/etc/nginx"
    "/etc/ssl"
)
BACKUP_BASE="/backup/files"
REMOTE_HOST="backup-server"
REMOTE_USER="backup"
DATE=$(date +%Y%m%d)

# ローカルバックアップ
for source in "${SOURCE_DIRS[@]}"; do
    dir_name=$(basename $source)
    backup_dir="$BACKUP_BASE/local/$dir_name"
    
    echo "$(date): Backing up $source to $backup_dir"
    
    # 増分バックアップ
    rsync -av --delete --link-dest=$backup_dir/current \
        $source/ $backup_dir/backup_$DATE/
    
    # currentリンク更新
    rm -f $backup_dir/current
    ln -s backup_$DATE $backup_dir/current
    
    # 古いバックアップ削除（7日以上前）
    find $backup_dir -maxdepth 1 -name "backup_*" -type d -mtime +7 -exec rm -rf {} \;
done

# リモートバックアップ
echo "$(date): Syncing to remote backup server"
rsync -av --delete -e "ssh -i /home/backup/.ssh/id_rsa" \
    $BACKUP_BASE/local/ $REMOTE_USER@$REMOTE_HOST:/backup/remote/

if [ $? -eq 0 ]; then
    echo "$(date): Remote sync completed successfully"
else
    echo "$(date): Remote sync failed"
    mail -s "Remote Backup Sync Failed" admin@example.com
fi

echo "$(date): File backup process completed" >> /var/log/backup.log
```

#### 4.1.2 tar による圧縮バックアップ

```bash
#!/bin/bash
# compressed_backup.sh

# 設定
BACKUP_SOURCES="/opt/application /etc/nginx /etc/ssl"
BACKUP_DIR="/backup/compressed"
REMOTE_BUCKET="s3://backup-bucket"
DATE=$(date +%Y%m%d)
RETENTION_DAYS=30

# 圧縮バックアップ作成
echo "$(date): Creating compressed backup"
tar -czf $BACKUP_DIR/system_backup_${DATE}.tar.gz \
    --exclude="*.log" \
    --exclude="*.tmp" \
    --exclude="cache/*" \
    $BACKUP_SOURCES

# 暗号化
gpg --cipher-algo AES256 --compress-algo 2 --symmetric \
    --output $BACKUP_DIR/system_backup_${DATE}.tar.gz.gpg \
    $BACKUP_DIR/system_backup_${DATE}.tar.gz

# 元ファイル削除
rm $BACKUP_DIR/system_backup_${DATE}.tar.gz

# クラウドストレージにアップロード
aws s3 cp $BACKUP_DIR/system_backup_${DATE}.tar.gz.gpg \
    $REMOTE_BUCKET/compressed/

# 古いバックアップ削除
find $BACKUP_DIR -name "system_backup_*.tar.gz.gpg" -mtime +$RETENTION_DAYS -delete

echo "$(date): Compressed backup completed" >> /var/log/backup.log
```

## 5. リストア手順

### 5.1 PostgreSQL リストア

#### 5.1.1 フルリストア

```bash
#!/bin/bash
# postgresql_full_restore.sh

# 設定（復元時に指定）
BACKUP_FILE="$1"
DB_NAME="$2"
DB_HOST="localhost"
DB_PORT="5432"
DB_USER="postgres"

if [ -z "$BACKUP_FILE" ] || [ -z "$DB_NAME" ]; then
    echo "Usage: $0 <backup_file> <database_name>"
    exit 1
fi

echo "$(date): Starting PostgreSQL full restore"
echo "Backup file: $BACKUP_FILE"
echo "Target database: $DB_NAME"

# バックアップファイル存在確認
if [ ! -f "$BACKUP_FILE" ]; then
    echo "Error: Backup file not found: $BACKUP_FILE"
    exit 1
fi

# チェックサム確認
if [ -f "$BACKUP_FILE.sha256" ]; then
    echo "$(date): Verifying backup file integrity"
    if ! sha256sum -c "$BACKUP_FILE.sha256"; then
        echo "Error: Backup file integrity check failed"
        exit 1
    fi
    echo "$(date): Backup file integrity verified"
fi

# データベース停止
echo "$(date): Stopping application services"
systemctl stop application

# 既存データベース削除・再作成
echo "$(date): Recreating database"
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -c "DROP DATABASE IF EXISTS $DB_NAME;"
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -c "CREATE DATABASE $DB_NAME;"

# リストア実行
echo "$(date): Restoring database from backup"
pg_restore -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
    --verbose \
    --clean \
    --if-exists \
    "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "$(date): Database restore completed successfully"
    
    # データ整合性確認
    echo "$(date): Verifying data integrity"
    psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -c "\dt"
    
    # アプリケーション再起動
    echo "$(date): Starting application services"
    systemctl start application
    
    # 動作確認
    sleep 30
    curl -f http://localhost:8080/health
    
    if [ $? -eq 0 ]; then
        echo "$(date): Application health check passed"
        echo "$(date): Full restore completed successfully" >> /var/log/restore.log
    else
        echo "$(date): Application health check failed"
        exit 1
    fi
    
else
    echo "$(date): Database restore failed"
    exit 1
fi
```

#### 5.1.2 ポイントインタイムリカバリ

```bash
#!/bin/bash
# postgresql_pitr.sh

# 設定
BASE_BACKUP_DIR="$1"
TARGET_TIME="$2"
WAL_ARCHIVE_DIR="/backup/postgresql/wal"
DB_DATA_DIR="/var/lib/postgresql/data"

if [ -z "$BASE_BACKUP_DIR" ] || [ -z "$TARGET_TIME" ]; then
    echo "Usage: $0 <base_backup_dir> <target_time>"
    echo "Example: $0 /backup/postgresql/incremental/base_20240101 '2024-01-01 12:00:00'"
    exit 1
fi

echo "$(date): Starting Point-in-Time Recovery"
echo "Base backup: $BASE_BACKUP_DIR"
echo "Target time: $TARGET_TIME"

# PostgreSQL停止
systemctl stop postgresql

# 既存データディレクトリバックアップ
mv $DB_DATA_DIR $DB_DATA_DIR.backup.$(date +%Y%m%d_%H%M%S)

# ベースバックアップ復元
tar -xzf $BASE_BACKUP_DIR/base.tar.gz -C /var/lib/postgresql/

# recovery.conf作成
cat > $DB_DATA_DIR/recovery.conf << EOF
restore_command = 'gunzip < $WAL_ARCHIVE_DIR/%f.gz > %p'
recovery_target_time = '$TARGET_TIME'
recovery_target_timeline = 'latest'
EOF

# PostgreSQL起動
systemctl start postgresql

# リカバリ完了確認
echo "$(date): Waiting for recovery to complete"
while psql -c "SELECT pg_is_in_recovery();" | grep -q "t"; do
    echo "Recovery in progress..."
    sleep 5
done

echo "$(date): Point-in-Time Recovery completed"
echo "$(date): PITR completed successfully" >> /var/log/restore.log
```

### 5.2 ファイルリストア

#### 5.2.1 アプリケーションファイルリストア

```bash
#!/bin/bash
# application_file_restore.sh

# 設定
BACKUP_DATE="$1"
RESTORE_TARGET="$2"
BACKUP_BASE="/backup/files/local"

if [ -z "$BACKUP_DATE" ] || [ -z "$RESTORE_TARGET" ]; then
    echo "Usage: $0 <backup_date> <restore_target>"
    echo "Example: $0 20240101 /opt/application"
    exit 1
fi

BACKUP_DIR="$BACKUP_BASE/$(basename $RESTORE_TARGET)/backup_$BACKUP_DATE"

echo "$(date): Starting file restore"
echo "Backup directory: $BACKUP_DIR"
echo "Restore target: $RESTORE_TARGET"

# バックアップディレクトリ存在確認
if [ ! -d "$BACKUP_DIR" ]; then
    echo "Error: Backup directory not found: $BACKUP_DIR"
    exit 1
fi

# 現在のファイルをバックアップ
if [ -d "$RESTORE_TARGET" ]; then
    echo "$(date): Backing up current files"
    mv "$RESTORE_TARGET" "$RESTORE_TARGET.backup.$(date +%Y%m%d_%H%M%S)"
fi

# ファイルリストア
echo "$(date): Restoring files"
rsync -av "$BACKUP_DIR/" "$RESTORE_TARGET/"

# 権限・所有者復元
echo "$(date): Restoring permissions"
case "$RESTORE_TARGET" in
    "/opt/application")
        chown -R app:app "$RESTORE_TARGET"
        chmod -R 755 "$RESTORE_TARGET"
        ;;
    "/var/www/uploads")
        chown -R www-data:www-data "$RESTORE_TARGET"
        chmod -R 644 "$RESTORE_TARGET"
        ;;
    "/etc/nginx")
        chown -R root:root "$RESTORE_TARGET"
        chmod -R 644 "$RESTORE_TARGET"
        chmod 755 "$RESTORE_TARGET"
        ;;
esac

echo "$(date): File restore completed successfully"
echo "$(date): File restore completed for $RESTORE_TARGET" >> /var/log/restore.log
```

### 5.3 圧縮バックアップリストア

```bash
#!/bin/bash
# compressed_backup_restore.sh

# 設定
BACKUP_FILE="$1"
RESTORE_DIR="/tmp/restore"
GPG_PASSPHRASE_FILE="/etc/backup/gpg-passphrase"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

echo "$(date): Starting compressed backup restore"
echo "Backup file: $BACKUP_FILE"

# 復元ディレクトリ作成
mkdir -p $RESTORE_DIR

# 復号化
echo "$(date): Decrypting backup file"
gpg --batch --yes --passphrase-file $GPG_PASSPHRASE_FILE \
    --output $RESTORE_DIR/backup.tar.gz \
    --decrypt "$BACKUP_FILE"

# 展開
echo "$(date): Extracting backup file"
tar -xzf $RESTORE_DIR/backup.tar.gz -C $RESTORE_DIR

# 復元実行
echo "$(date): Restoring files to original locations"
rsync -av $RESTORE_DIR/opt/application/ /opt/application/
rsync -av $RESTORE_DIR/etc/nginx/ /etc/nginx/
rsync -av $RESTORE_DIR/etc/ssl/ /etc/ssl/

# 権限復元
chown -R app:app /opt/application
chown -R root:root /etc/nginx /etc/ssl

# 一時ファイル削除
rm -rf $RESTORE_DIR

echo "$(date): Compressed backup restore completed"
echo "$(date): Compressed backup restore completed" >> /var/log/restore.log
```

## 6. 災害復旧手順

### 6.1 完全災害復旧

#### 6.1.1 新環境構築とデータ復旧

```bash
#!/bin/bash
# disaster_recovery.sh

# 設定
DR_SITE_HOST="dr-server"
DR_SITE_USER="recovery"
BACKUP_BUCKET="s3://backup-bucket"
RECOVERY_DATE="$1"

if [ -z "$RECOVERY_DATE" ]; then
    echo "Usage: $0 <recovery_date>"
    echo "Example: $0 20240101"
    exit 1
fi

echo "$(date): Starting disaster recovery for date: $RECOVERY_DATE"

# 1. バックアップファイルのダウンロード
echo "$(date): Downloading backup files from cloud storage"
aws s3 sync $BACKUP_BUCKET/compressed/ /tmp/dr-restore/ --exclude "*" --include "*$RECOVERY_DATE*"
aws s3 sync $BACKUP_BUCKET/postgresql/ /tmp/dr-restore/ --exclude "*" --include "*$RECOVERY_DATE*"

# 2. システム環境構築
echo "$(date): Setting up system environment"
# Ansible playbookによる環境構築
ansible-playbook -i inventory/dr-hosts.yml playbooks/system-setup.yml

# 3. データベース復旧
echo "$(date): Restoring database"
./postgresql_full_restore.sh /tmp/dr-restore/full_backup_${RECOVERY_DATE}.dump main_db

# 4. アプリケーションファイル復旧
echo "$(date): Restoring application files"
./compressed_backup_restore.sh /tmp/dr-restore/system_backup_${RECOVERY_DATE}.tar.gz.gpg

# 5. 設定ファイル調整
echo "$(date): Adjusting configuration files"
# DR環境用設定への変更
sed -i "s/primary-db-server/dr-db-server/g" /opt/application/config/database.yml
sed -i "s/primary.example.com/dr.example.com/g" /etc/nginx/nginx.conf

# 6. サービス起動
echo "$(date): Starting services"
systemctl start postgresql
systemctl start application
systemctl start nginx

# 7. 動作確認
echo "$(date): Performing health checks"
sleep 60

# データベース接続確認
psql -c "SELECT version();"

# アプリケーション動作確認
curl -f http://localhost:8080/health

# Web応答確認
curl -f http://localhost/

if [ $? -eq 0 ]; then
    echo "$(date): Disaster recovery completed successfully"
    echo "$(date): DR completed for $RECOVERY_DATE" >> /var/log/dr.log
    
    # 成功通知
    mail -s "Disaster Recovery Completed" admin@example.com << EOF
Disaster recovery has been completed successfully.

Recovery Date: $RECOVERY_DATE
Completion Time: $(date)
DR Site: $DR_SITE_HOST

All services are now operational.
EOF

else
    echo "$(date): Disaster recovery failed"
    mail -s "Disaster Recovery Failed" admin@example.com
    exit 1
fi
```

## 7. バックアップ検証

### 7.1 定期検証

#### 7.1.1 自動バックアップ検証

```bash
#!/bin/bash
# backup_verification.sh

BACKUP_DIR="/backup/postgresql/full"
TEST_DB="backup_test"
LATEST_BACKUP=$(ls -t $BACKUP_DIR/full_backup_*.dump | head -1)

echo "$(date): Starting backup verification"
echo "Testing backup: $LATEST_BACKUP"

# テストデータベース作成
createdb $TEST_DB

# バックアップファイルのリストア
pg_restore -d $TEST_DB "$LATEST_BACKUP"

if [ $? -eq 0 ]; then
    # データ整合性確認
    ROW_COUNT=$(psql -d $TEST_DB -t -c "SELECT COUNT(*) FROM users;")
    TABLE_COUNT=$(psql -d $TEST_DB -t -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public';")
    
    echo "$(date): Backup verification results:"
    echo "  Tables: $TABLE_COUNT"
    echo "  Users: $ROW_COUNT"
    
    if [ $ROW_COUNT -gt 0 ] && [ $TABLE_COUNT -gt 0 ]; then
        echo "$(date): Backup verification PASSED"
        echo "$(date): Backup verification passed for $LATEST_BACKUP" >> /var/log/backup-verification.log
    else
        echo "$(date): Backup verification FAILED - Data integrity issue"
        mail -s "Backup Verification Failed" admin@example.com
    fi
    
    # テストデータベース削除
    dropdb $TEST_DB
    
else
    echo "$(date): Backup verification FAILED - Restore failed"
    mail -s "Backup Verification Failed" admin@example.com
fi
```

### 7.2 月次フルテスト

#### 7.2.1 完全リストアテスト

```bash
#!/bin/bash
# monthly_restore_test.sh

# テスト環境でのフル復旧テスト（月次）
TEST_ENV="test-server"
BACKUP_DATE=$(date -d "last sunday" +%Y%m%d)

echo "$(date): Starting monthly restore test"
echo "Target date: $BACKUP_DATE"

# テスト環境でのリストア実行
ssh $TEST_ENV "
    # アプリケーション停止
    sudo systemctl stop application nginx postgresql
    
    # データリストア
    ./postgresql_full_restore.sh /backup/postgresql/full/full_backup_${BACKUP_DATE}.dump test_db
    ./application_file_restore.sh $BACKUP_DATE /opt/application
    
    # サービス起動
    sudo systemctl start postgresql application nginx
    
    # 動作確認
    sleep 30
    curl -f http://localhost:8080/health
"

if [ $? -eq 0 ]; then
    echo "$(date): Monthly restore test PASSED"
    echo "$(date): Monthly restore test passed" >> /var/log/restore-test.log
else
    echo "$(date): Monthly restore test FAILED"
    mail -s "Monthly Restore Test Failed" admin@example.com
fi
```

## 8. 監視・アラート

### 8.1 バックアップ監視

#### 8.1.1 監視スクリプト

```bash
#!/bin/bash
# backup_monitoring.sh

BACKUP_DIRS=(
    "/backup/postgresql/full"
    "/backup/postgresql/incremental" 
    "/backup/files/local"
    "/backup/redis"
)

for dir in "${BACKUP_DIRS[@]}"; do
    # 最新バックアップファイルの確認
    LATEST_FILE=$(find $dir -type f -name "*$(date +%Y%m%d)*" | head -1)
    
    if [ -z "$LATEST_FILE" ]; then
        echo "WARNING: No backup found for today in $dir"
        mail -s "Backup Missing Warning" admin@example.com << EOF
No backup file found for today in: $dir
Please check the backup process immediately.
EOF
    else
        echo "OK: Backup found in $dir - $LATEST_FILE"
        
        # ファイルサイズ確認
        SIZE=$(du -h "$LATEST_FILE" | cut -f1)
        echo "  Size: $SIZE"
        
        # ファイル年齢確認
        AGE=$(find "$LATEST_FILE" -mtime +1)
        if [ -n "$AGE" ]; then
            echo "WARNING: Backup file is older than 24 hours"
        fi
    fi
done

echo "$(date): Backup monitoring completed" >> /var/log/backup-monitoring.log
```

---

**注意**: このバックアップ・リストア手順書は、データ保護の重要な手順を定義しています。手順の変更時は十分なテストを実施し、定期的な検証を行ってください。
# セキュリティ運用手順書

**タイトル**: セキュリティ運用手順書  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: security-design.md, incident-response-procedures.md  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システムのセキュリティを維持するための日常的な運用手順を定義し、セキュリティインシデントの予防と早期発見を実現することを目的とする。

### 1.2 適用範囲
本手順は、以下のセキュリティ運用に適用される：
- セキュリティ監視・分析
- 脆弱性管理
- アクセス制御管理
- インシデント対応
- セキュリティ教育・訓練

### 1.3 セキュリティ運用方針
- **継続監視**: 24時間365日のセキュリティ監視
- **予防重視**: 事前防御による被害最小化
- **迅速対応**: インシデント発生時の素早い初動対応
- **継続改善**: 脅威動向に応じた対策継続的改善

## 2. セキュリティ監視体制

### 2.1 監視体制

#### 2.1.1 役割分担

| 役割 | 責任者 | 主な業務 | 勤務時間 |
|---|---|---|---|
| **CISO** | セキュリティ責任者 | セキュリティ戦略・意思決定 | 営業時間 |
| **SOCアナリスト** | セキュリティ専門家 | 高度分析・インシデント対応 | 24時間体制 |
| **セキュリティエンジニア** | 技術担当者 | 技術的対応・設定変更 | 営業時間+オンコール |
| **運用エンジニア** | 運用担当者 | 基本監視・一次対応 | 24時間体制 |

#### 2.1.2 エスカレーション手順

```
┌─────────────────┐
│ Level 1         │ ← セキュリティアラート検知（即座）
│ 運用エンジニア   │
└─────────────────┘
         │ 15分以内/重要度High以上
         ▼
┌─────────────────┐
│ Level 2         │ ← 詳細分析・対応判断（30分以内）
│ SOCアナリスト    │
└─────────────────┘
         │ 1時間以内/Critical
         ▼
┌─────────────────┐
│ Level 3         │ ← 経営判断・対外対応（即座）
│ CISO            │
└─────────────────┘
```

### 2.2 監視対象・項目

#### 2.2.1 ネットワーク監視

**監視項目**
- 不正アクセス試行
- DDoS攻撃パターン
- 異常な通信量・パターン
- 未承認の通信プロトコル

**監視ツール設定**
```bash
# Suricata IDS設定例
# /etc/suricata/suricata.yaml

default-rule-path: /var/lib/suricata/rules
rule-files:
  - suricata.rules
  - emerging-threats.rules
  - local.rules

# カスタムルール例（/var/lib/suricata/rules/local.rules）
alert tcp any any -> $HOME_NET 22 (msg:"SSH Brute Force Attempt"; 
    flow:to_server,established; content:"SSH"; threshold:type both,track by_src,count 5,seconds 60; 
    sid:1000001; rev:1;)

alert http any any -> $HOME_NET any (msg:"SQL Injection Attempt"; 
    flow:to_server,established; content:"union select"; nocase; 
    sid:1000002; rev:1;)
```

#### 2.2.2 アプリケーション監視

**監視項目**
- 認証失敗パターン
- 異常なAPIアクセス
- 権限昇格試行
- データアクセス異常

```python
# アプリケーションセキュリティ監視例
import logging
from collections import defaultdict
from datetime import datetime, timedelta

class SecurityMonitor:
    def __init__(self):
        self.failed_logins = defaultdict(list)
        self.api_requests = defaultdict(list)
        
    def monitor_login_attempts(self, username, ip_address, success):
        """ログイン試行の監視"""
        current_time = datetime.now()
        
        if not success:
            self.failed_logins[ip_address].append(current_time)
            
            # 5分間に5回失敗した場合アラート
            recent_failures = [t for t in self.failed_logins[ip_address] 
                             if current_time - t < timedelta(minutes=5)]
            
            if len(recent_failures) >= 5:
                self.alert_brute_force(username, ip_address)
                
    def monitor_api_access(self, user_id, endpoint, method):
        """API アクセス監視"""
        current_time = datetime.now()
        self.api_requests[user_id].append({
            'time': current_time,
            'endpoint': endpoint,
            'method': method
        })
        
        # 1分間に100リクエスト以上の場合アラート
        recent_requests = [r for r in self.api_requests[user_id]
                          if current_time - r['time'] < timedelta(minutes=1)]
        
        if len(recent_requests) >= 100:
            self.alert_api_abuse(user_id)
            
    def alert_brute_force(self, username, ip_address):
        """ブルートフォース攻撃アラート"""
        logging.critical(f"Brute force attack detected: {username} from {ip_address}")
        # アラート送信処理
        
    def alert_api_abuse(self, user_id):
        """API乱用アラート"""
        logging.warning(f"API abuse detected: User {user_id}")
        # アラート送信処理
```

#### 2.2.3 システム監視

**監視項目**
- 管理者権限でのコマンド実行
- 重要ファイルの変更
- 異常なプロセス実行
- 不審なネットワーク接続

```bash
# auditd 設定例（/etc/audit/rules.d/security.rules）

# 重要ディレクトリの監視
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/ssh/sshd_config -p wa -k ssh_config_changes
-w /opt/application/config -p wa -k app_config_changes

# 管理者コマンドの監視
-w /bin/su -p x -k su_usage
-w /bin/sudo -p x -k sudo_usage
-w /usr/bin/passwd -p x -k passwd_usage

# ネットワーク設定変更の監視
-w /etc/hosts -p wa -k network_changes
-w /etc/resolv.conf -p wa -k dns_changes

# システムコール監視
-a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time_change
-a always,exit -F arch=b64 -S clock_settime -k time_change
```

## 3. 日常セキュリティ運用

### 3.1 日次セキュリティチェック

#### 3.1.1 セキュリティログ分析（毎日 9:00）

```bash
#!/bin/bash
# daily_security_check.sh

LOG_DATE=$(date +%Y%m%d)
REPORT_FILE="/var/log/security/daily_report_$LOG_DATE.txt"

echo "=== 日次セキュリティレポート $(date) ===" > $REPORT_FILE

# 1. 認証ログ分析
echo "## 認証関連" >> $REPORT_FILE
echo "### ログイン失敗" >> $REPORT_FILE
grep "Failed password" /var/log/auth.log | grep "$(date +%b' '%d)" | \
    awk '{print $11}' | sort | uniq -c | sort -nr >> $REPORT_FILE

echo "### 不正ユーザー試行" >> $REPORT_FILE
grep "Invalid user" /var/log/auth.log | grep "$(date +%b' '%d)" | \
    awk '{print $8, $10}' | sort | uniq -c | sort -nr >> $REPORT_FILE

# 2. Web アクセスログ分析
echo "## Webアクセス関連" >> $REPORT_FILE
echo "### 4xx/5xx エラー" >> $REPORT_FILE
awk '$9>=400 {print $1, $7, $9}' /var/log/nginx/access.log | \
    grep "$(date +%d/%b/%Y)" | sort | uniq -c | sort -nr | head -20 >> $REPORT_FILE

echo "### 不審なUser-Agent" >> $REPORT_FILE
grep -E "(bot|crawler|scanner)" /var/log/nginx/access.log | \
    grep "$(date +%d/%b/%Y)" | awk '{print $1, $12}' | sort | uniq -c | sort -nr >> $REPORT_FILE

# 3. システムログ分析
echo "## システム関連" >> $REPORT_FILE
echo "### Sudo使用履歴" >> $REPORT_FILE
grep "sudo:" /var/log/auth.log | grep "$(date +%b' '%d)" >> $REPORT_FILE

echo "### ファイル変更検知" >> $REPORT_FILE
aureport --file --start today --end today >> $REPORT_FILE

# 4. アラート生成
FAILED_LOGINS=$(grep "Failed password" /var/log/auth.log | grep "$(date +%b' '%d)" | wc -l)
if [ $FAILED_LOGINS -gt 100 ]; then
    echo "WARNING: High number of failed logins: $FAILED_LOGINS" >> $REPORT_FILE
    mail -s "Security Alert: High Failed Logins" security@example.com < $REPORT_FILE
fi

echo "日次セキュリティチェック完了: $(date)" >> /var/log/security/security_ops.log
```

#### 3.1.2 脆弱性スキャン（毎日 2:00）

```bash
#!/bin/bash
# daily_vulnerability_scan.sh

SCAN_DATE=$(date +%Y%m%d)
SCAN_TARGETS="localhost 192.168.1.100 192.168.1.101"
REPORT_DIR="/var/log/security/vulns"

mkdir -p $REPORT_DIR

for target in $SCAN_TARGETS; do
    echo "$(date): Scanning $target"
    
    # Nmap脆弱性スキャン
    nmap --script vuln -oN $REPORT_DIR/nmap_${target}_${SCAN_DATE}.txt $target
    
    # OpenVAS スキャン（スケジュール実行）
    if command -v omp &> /dev/null; then
        omp -u admin -w password --xml="<create_task>
            <name>Daily Scan $target</name>
            <config id=\"74db13d6-7489-11df-a3ec-002264764cea\"/>
            <target id=\"$target\"/>
        </create_task>"
    fi
    
    # 結果分析
    CRITICAL_VULNS=$(grep -c "Critical" $REPORT_DIR/nmap_${target}_${SCAN_DATE}.txt || echo 0)
    if [ $CRITICAL_VULNS -gt 0 ]; then
        echo "CRITICAL: $CRITICAL_VULNS critical vulnerabilities found on $target"
        mail -s "Critical Vulnerabilities Found" security@example.com \
            -a $REPORT_DIR/nmap_${target}_${SCAN_DATE}.txt
    fi
done

echo "$(date): Daily vulnerability scan completed" >> /var/log/security/security_ops.log
```

### 3.2 週次セキュリティ作業

#### 3.2.1 セキュリティパッチ適用確認（毎週日曜日）

```bash
#!/bin/bash
# weekly_patch_check.sh

echo "=== 週次パッチ確認 $(date) ==="

# 1. OS パッチ確認
echo "## OS パッチ状況"
if command -v apt &> /dev/null; then
    apt list --upgradable | grep -i security
elif command -v yum &> /dev/null; then
    yum check-update --security
fi

# 2. アプリケーションパッチ確認
echo "## アプリケーションパッチ状況"

# Docker イメージ更新確認
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.CreatedAt}}" | grep -v "CREATED"

# 手動更新が必要なコンポーネント確認
echo "## 手動確認が必要な項目"
echo "- PostgreSQL: $(psql --version)"
echo "- Nginx: $(nginx -v 2>&1)"
echo "- OpenSSL: $(openssl version)"

# 3. CVE データベース確認
echo "## 関連CVE確認"
# 使用ソフトウェアの既知脆弱性チェック
curl -s "https://cve.circl.lu/api/search/postgresql" | jq '.[]' | head -5

echo "$(date): Weekly patch check completed" >> /var/log/security/security_ops.log
```

#### 3.2.2 アクセス権限監査（毎週金曜日）

```bash
#!/bin/bash
# weekly_access_audit.sh

AUDIT_DATE=$(date +%Y%m%d)
AUDIT_REPORT="/var/log/security/access_audit_$AUDIT_DATE.txt"

echo "=== アクセス権限監査 $(date) ===" > $AUDIT_REPORT

# 1. システムユーザー確認
echo "## システムユーザー一覧" >> $AUDIT_REPORT
awk -F: '$3 >= 1000 {print $1, $3, $5, $6, $7}' /etc/passwd >> $AUDIT_REPORT

# 2. sudo 権限確認
echo "## sudo 権限ユーザー" >> $AUDIT_REPORT
grep -E '^%|^[^#]' /etc/sudoers /etc/sudoers.d/* >> $AUDIT_REPORT

# 3. SSH キー確認
echo "## SSH 公開鍵" >> $AUDIT_REPORT
for user in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    if [ -f "/home/$user/.ssh/authorized_keys" ]; then
        echo "User: $user" >> $AUDIT_REPORT
        cat "/home/$user/.ssh/authorized_keys" >> $AUDIT_REPORT
    fi
done

# 4. データベースユーザー確認
echo "## データベースユーザー" >> $AUDIT_REPORT
psql -c "SELECT usename, usesuper, usecreatedb FROM pg_user;" >> $AUDIT_REPORT

# 5. アプリケーション権限確認
echo "## アプリケーション権限" >> $AUDIT_REPORT
# アプリケーション固有の権限確認スクリプト実行
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
    http://localhost:8080/admin/users | jq '.[]' >> $AUDIT_REPORT

echo "$(date): Weekly access audit completed" >> /var/log/security/security_ops.log
```

### 3.3 月次セキュリティ作業

#### 3.3.1 包括的セキュリティ評価（毎月第1土曜日）

```bash
#!/bin/bash
# monthly_security_assessment.sh

ASSESSMENT_DATE=$(date +%Y%m)
REPORT_DIR="/var/log/security/monthly/$ASSESSMENT_DATE"

mkdir -p $REPORT_DIR

echo "=== 月次セキュリティ評価 $(date) ==="

# 1. ペネトレーションテスト
echo "## ペネトレーションテスト実行"
python3 /opt/security/pentest.py --target localhost --output $REPORT_DIR/pentest.json

# 2. 設定監査
echo "## 設定監査"
# CIS Benchmark チェック
if command -v lynis &> /dev/null; then
    lynis audit system --report-file $REPORT_DIR/lynis_report.dat
fi

# 3. ログ分析
echo "## 月次ログ分析"
# 過去1ヶ月のセキュリティインシデント集計
python3 /opt/security/log_analyzer.py --period 30 --output $REPORT_DIR/log_analysis.json

# 4. レポート生成
echo "## 総合レポート生成"
python3 /opt/security/report_generator.py \
    --pentest $REPORT_DIR/pentest.json \
    --lynis $REPORT_DIR/lynis_report.dat \
    --logs $REPORT_DIR/log_analysis.json \
    --output $REPORT_DIR/monthly_security_report.pdf

# 5. 結果通知
mail -s "Monthly Security Assessment Report" -a $REPORT_DIR/monthly_security_report.pdf \
    security@example.com ciso@example.com

echo "$(date): Monthly security assessment completed" >> /var/log/security/security_ops.log
```

## 4. インシデント対応手順

### 4.1 セキュリティインシデント分類

#### 4.1.1 インシデント分類基準

| レベル | 定義 | 例 | 対応時間 |
|---|---|---|---|
| **Critical** | 重大なセキュリティ侵害 | データ漏洩、システム乗っ取り | 15分以内 |
| **High** | 重要なセキュリティ事象 | マルウェア感染、不正アクセス | 1時間以内 |
| **Medium** | 軽微なセキュリティ事象 | 脆弱性発見、設定不備 | 4時間以内 |
| **Low** | 情報提供レベル | 疑わしい活動、警告 | 24時間以内 |

### 4.2 初期対応手順

#### 4.2.1 インシデント検知時対応

```bash
#!/bin/bash
# incident_response_initial.sh

INCIDENT_ID="$1"
INCIDENT_TYPE="$2"
SEVERITY="$3"

if [ -z "$INCIDENT_ID" ] || [ -z "$INCIDENT_TYPE" ] || [ -z "$SEVERITY" ]; then
    echo "Usage: $0 <incident_id> <incident_type> <severity>"
    exit 1
fi

INCIDENT_DIR="/var/log/security/incidents/$INCIDENT_ID"
mkdir -p $INCIDENT_DIR

echo "=== セキュリティインシデント初期対応 ===" | tee $INCIDENT_DIR/response.log
echo "インシデントID: $INCIDENT_ID" | tee -a $INCIDENT_DIR/response.log
echo "インシデント種別: $INCIDENT_TYPE" | tee -a $INCIDENT_DIR/response.log
echo "重要度: $SEVERITY" | tee -a $INCIDENT_DIR/response.log
echo "検知時刻: $(date)" | tee -a $INCIDENT_DIR/response.log

# 1. 証跡保全
echo "$(date): 証跡保全開始" | tee -a $INCIDENT_DIR/response.log

# システム状態スナップショット
ps aux > $INCIDENT_DIR/processes.txt
netstat -tulpn > $INCIDENT_DIR/network.txt
lsof > $INCIDENT_DIR/openfiles.txt
df -h > $INCIDENT_DIR/disk.txt
free -h > $INCIDENT_DIR/memory.txt

# ログファイル保存
cp /var/log/auth.log $INCIDENT_DIR/
cp /var/log/nginx/access.log $INCIDENT_DIR/
cp /var/log/nginx/error.log $INCIDENT_DIR/
cp /var/log/application/*.log $INCIDENT_DIR/

# 2. 影響範囲特定
echo "$(date): 影響範囲特定開始" | tee -a $INCIDENT_DIR/response.log

case $INCIDENT_TYPE in
    "malware")
        # マルウェア感染対応
        clamav-daemon --infected-files=$INCIDENT_DIR/infected_files.txt
        ;;
    "unauthorized_access")
        # 不正アクセス対応
        last -n 50 > $INCIDENT_DIR/login_history.txt
        w > $INCIDENT_DIR/current_users.txt
        ;;
    "data_breach")
        # データ漏洩対応
        audit_log_extract.sh > $INCIDENT_DIR/audit_extract.txt
        ;;
esac

# 3. 緊急連絡
echo "$(date): 緊急連絡実施" | tee -a $INCIDENT_DIR/response.log

if [ "$SEVERITY" = "Critical" ] || [ "$SEVERITY" = "High" ]; then
    # 緊急連絡先への通知
    mail -s "SECURITY INCIDENT [$SEVERITY] $INCIDENT_TYPE" security@example.com << EOF
セキュリティインシデントが発生しました。

インシデントID: $INCIDENT_ID
種別: $INCIDENT_TYPE
重要度: $SEVERITY
検知時刻: $(date)

詳細は以下のディレクトリで確認してください：
$INCIDENT_DIR

初期対応者: $(whoami)
EOF

    # Slack通知
    curl -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"🚨 SECURITY INCIDENT [$SEVERITY] $INCIDENT_TYPE - ID: $INCIDENT_ID\"}" \
        $SLACK_WEBHOOK_URL
fi

echo "$(date): 初期対応完了" | tee -a $INCIDENT_DIR/response.log
```

### 4.3 封じ込め対応

#### 4.3.1 悪意あるIPアドレス遮断

```bash
#!/bin/bash
# block_malicious_ip.sh

MALICIOUS_IP="$1"
REASON="$2"

if [ -z "$MALICIOUS_IP" ]; then
    echo "Usage: $0 <ip_address> [reason]"
    exit 1
fi

echo "$(date): Blocking malicious IP: $MALICIOUS_IP"

# 1. iptables による即座の遮断
iptables -I INPUT 1 -s $MALICIOUS_IP -j DROP
iptables -I OUTPUT 1 -d $MALICIOUS_IP -j DROP

# 2. fail2ban への追加
fail2ban-client set nginx-http-auth banip $MALICIOUS_IP

# 3. WAF での遮断設定
# ModSecurity設定更新
echo "SecRule REMOTE_ADDR \"@ipMatch $MALICIOUS_IP\" \"id:9001,phase:1,block,msg:'Malicious IP blocked'\"" \
    >> /etc/nginx/modsecurity/custom_rules.conf

# 4. ログ記録
echo "$(date): IP $MALICIOUS_IP blocked. Reason: $REASON" >> /var/log/security/blocked_ips.log

# 5. 通知
mail -s "Malicious IP Blocked" security@example.com << EOF
悪意あるIPアドレスを遮断しました。

IP Address: $MALICIOUS_IP
Reason: $REASON
Blocked Time: $(date)
Action: iptables + fail2ban + WAF

詳細はセキュリティログを確認してください。
EOF

echo "IP address $MALICIOUS_IP has been blocked successfully"
```

#### 4.3.2 感染システム隔離

```bash
#!/bin/bash
# isolate_infected_system.sh

INFECTED_HOST="$1"
INCIDENT_ID="$2"

if [ -z "$INFECTED_HOST" ]; then
    echo "Usage: $0 <infected_host> [incident_id]"
    exit 1
fi

echo "$(date): Isolating infected system: $INFECTED_HOST"

# 1. ネットワーク隔離
if [ "$INFECTED_HOST" = "localhost" ]; then
    # ローカルシステムの場合
    echo "WARNING: Isolating localhost - this will disconnect all network access"
    
    # 緊急隔離モード
    iptables -P INPUT DROP
    iptables -P OUTPUT DROP
    iptables -P FORWARD DROP
    
    # 管理用SSHアクセスのみ許可
    iptables -A INPUT -i lo -j ACCEPT
    iptables -A OUTPUT -o lo -j ACCEPT
    iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT
    iptables -A OUTPUT -p tcp --sport 22 -d 192.168.1.0/24 -j ACCEPT
    
else
    # リモートシステムの場合
    ssh $INFECTED_HOST "
        iptables -P INPUT DROP
        iptables -P OUTPUT DROP
        iptables -P FORWARD DROP
        iptables -A INPUT -i lo -j ACCEPT
        iptables -A OUTPUT -o lo -j ACCEPT
    "
fi

# 2. 疑わしいプロセス停止
SUSPICIOUS_PROCESSES="cryptominer malware backdoor"
for process in $SUSPICIOUS_PROCESSES; do
    if [ "$INFECTED_HOST" = "localhost" ]; then
        pkill -f $process
    else
        ssh $INFECTED_HOST "pkill -f $process"
    fi
done

# 3. 証跡保全
EVIDENCE_DIR="/var/log/security/incidents/$INCIDENT_ID/evidence"
mkdir -p $EVIDENCE_DIR

if [ "$INFECTED_HOST" = "localhost" ]; then
    # メモリダンプ取得
    dd if=/dev/mem of=$EVIDENCE_DIR/memory_dump.img bs=1M
    
    # プロセス情報保存
    ps auxf > $EVIDENCE_DIR/process_tree.txt
    lsof > $EVIDENCE_DIR/open_files.txt
    
else
    # リモートシステムからの証跡収集
    ssh $INFECTED_HOST "ps auxf" > $EVIDENCE_DIR/remote_processes.txt
    ssh $INFECTED_HOST "lsof" > $EVIDENCE_DIR/remote_openfiles.txt
fi

# 4. 通知
mail -s "System Isolated - $INFECTED_HOST" security@example.com << EOF
感染システムを隔離しました。

Host: $INFECTED_HOST
Incident ID: $INCIDENT_ID
Isolation Time: $(date)

実施された隔離措置:
- ネットワークアクセス遮断
- 疑わしいプロセス停止
- 証跡保全実施

詳細調査を開始してください。
EOF

echo "$(date): System $INFECTED_HOST isolated successfully" >> /var/log/security/isolation.log
```

## 5. フォレンジック調査

### 5.1 ログ分析

#### 5.1.1 自動ログ分析スクリプト

```python
#!/usr/bin/env python3
# forensic_log_analysis.py

import json
import re
import sys
from datetime import datetime, timedelta
from collections import defaultdict, Counter
import geoip2.database

class ForensicAnalyzer:
    def __init__(self):
        self.suspicious_patterns = [
            r'(?i)(union|select|insert|update|delete|drop)\s+.*\s+(from|into|table)',  # SQL Injection
            r'(?i)<script[^>]*>.*</script>',  # XSS
            r'(?i)(\.\./){2,}',  # Directory Traversal
            r'(?i)(cmd|exec|system|shell_exec|passthru)',  # Command Injection
            r'(?i)/etc/passwd',  # System File Access
        ]
        
        self.geoip_reader = None
        try:
            self.geoip_reader = geoip2.database.Reader('/usr/share/GeoIP/GeoLite2-Country.mmdb')
        except:
            print("Warning: GeoIP database not found")
    
    def analyze_web_logs(self, log_file):
        """Web アクセスログの分析"""
        results = {
            'suspicious_requests': [],
            'top_ips': Counter(),
            'error_patterns': Counter(),
            'geographic_distribution': Counter()
        }
        
        with open(log_file, 'r') as f:
            for line in f:
                # Nginxログフォーマット解析
                match = re.match(r'(\S+) - - \[([^\]]+)\] "(\S+) ([^"]*)" (\d+) (\d+) "([^"]*)" "([^"]*)"', line)
                if not match:
                    continue
                    
                ip, timestamp, method, uri, status, size, referer, user_agent = match.groups()
                
                # IP集計
                results['top_ips'][ip] += 1
                
                # 地理的分布（GeoIP利用）
                if self.geoip_reader:
                    try:
                        response = self.geoip_reader.country(ip)
                        results['geographic_distribution'][response.country.iso_code] += 1
                    except:
                        results['geographic_distribution']['Unknown'] += 1
                
                # エラーコード分析
                if int(status) >= 400:
                    results['error_patterns'][f"{status} {uri}"] += 1
                
                # 疑わしいパターン検出
                for pattern in self.suspicious_patterns:
                    if re.search(pattern, uri) or re.search(pattern, user_agent):
                        results['suspicious_requests'].append({
                            'ip': ip,
                            'timestamp': timestamp,
                            'method': method,
                            'uri': uri,
                            'status': status,
                            'user_agent': user_agent,
                            'pattern': pattern
                        })
        
        return results
    
    def analyze_auth_logs(self, log_file):
        """認証ログの分析"""
        results = {
            'failed_logins': defaultdict(list),
            'successful_logins': defaultdict(list),
            'brute_force_attempts': [],
            'unusual_login_times': []
        }
        
        with open(log_file, 'r') as f:
            for line in f:
                if 'Failed password' in line:
                    # 失敗ログイン解析
                    match = re.search(r'Failed password for (\w+) from (\S+)', line)
                    if match:
                        user, ip = match.groups()
                        timestamp = self.extract_timestamp(line)
                        results['failed_logins'][ip].append({
                            'user': user,
                            'timestamp': timestamp
                        })
                
                elif 'Accepted password' in line:
                    # 成功ログイン解析
                    match = re.search(r'Accepted password for (\w+) from (\S+)', line)
                    if match:
                        user, ip = match.groups()
                        timestamp = self.extract_timestamp(line)
                        results['successful_logins'][ip].append({
                            'user': user,
                            'timestamp': timestamp
                        })
                        
                        # 異常ログイン時間検知（営業時間外）
                        if timestamp.hour < 9 or timestamp.hour > 18:
                            results['unusual_login_times'].append({
                                'user': user,
                                'ip': ip,
                                'timestamp': timestamp
                            })
        
        # ブルートフォース攻撃検知
        for ip, attempts in results['failed_logins'].items():
            if len(attempts) > 10:  # 10回以上の失敗
                results['brute_force_attempts'].append({
                    'ip': ip,
                    'attempts': len(attempts),
                    'users_targeted': list(set([a['user'] for a in attempts]))
                })
        
        return results
    
    def extract_timestamp(self, log_line):
        """ログから日時を抽出"""
        # 簡易実装（実際はログフォーマットに応じて調整）
        try:
            match = re.search(r'(\w{3}\s+\d{1,2}\s+\d{2}:\d{2}:\d{2})', log_line)
            if match:
                time_str = match.group(1)
                # 年を追加して解析
                time_str = f"2024 {time_str}"
                return datetime.strptime(time_str, "%Y %b %d %H:%M:%S")
        except:
            pass
        return datetime.now()
    
    def generate_report(self, analysis_results, output_file):
        """分析結果レポート生成"""
        report = {
            'analysis_time': datetime.now().isoformat(),
            'summary': {
                'total_suspicious_requests': len(analysis_results.get('suspicious_requests', [])),
                'total_brute_force_attempts': len(analysis_results.get('brute_force_attempts', [])),
                'total_unusual_logins': len(analysis_results.get('unusual_login_times', []))
            },
            'details': analysis_results
        }
        
        with open(output_file, 'w') as f:
            json.dump(report, f, indent=2, default=str)
        
        print(f"Forensic analysis report saved to {output_file}")

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("Usage: python3 forensic_log_analysis.py <log_type> <log_file> [output_file]")
        print("Log types: web, auth")
        sys.exit(1)
    
    log_type = sys.argv[1]
    log_file = sys.argv[2]
    output_file = sys.argv[3] if len(sys.argv) > 3 else f"forensic_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
    
    analyzer = ForensicAnalyzer()
    
    if log_type == "web":
        results = analyzer.analyze_web_logs(log_file)
    elif log_type == "auth":
        results = analyzer.analyze_auth_logs(log_file)
    else:
        print("Unsupported log type")
        sys.exit(1)
    
    analyzer.generate_report(results, output_file)
```

## 6. セキュリティ教育・訓練

### 6.1 定期訓練

#### 6.1.1 インシデント対応訓練（四半期）

```bash
#!/bin/bash
# incident_response_drill.sh

DRILL_DATE=$(date +%Y%m%d)
DRILL_TYPE="$1"  # malware, breach, ddos

echo "=== インシデント対応訓練 $(date) ==="
echo "訓練種別: $DRILL_TYPE"

case $DRILL_TYPE in
    "malware")
        echo "## マルウェア感染対応訓練"
        echo "シナリオ: Webサーバーでマルウェア検知"
        
        # 模擬アラート生成
        echo "ALERT: Malware detected on web-server-01" | \
            mail -s "DRILL: Malware Detection" security@example.com
        
        # 対応手順確認
        echo "対応チェックリスト:"
        echo "[ ] システム隔離実施"
        echo "[ ] 証跡保全実施"  
        echo "[ ] マルウェア特定・除去"
        echo "[ ] 被害範囲調査"
        echo "[ ] 復旧手順実施"
        ;;
        
    "breach")
        echo "## データ漏洩対応訓練"
        echo "シナリオ: 不正アクセスによる個人情報漏洩"
        
        # 模擬ログ生成
        echo "$(date): Unauthorized access to customer database detected" >> /tmp/drill_alert.log
        
        echo "対応チェックリスト:"
        echo "[ ] アクセス遮断実施"
        echo "[ ] 漏洩範囲特定"
        echo "[ ] 関係機関への報告"
        echo "[ ] 顧客への通知準備"
        echo "[ ] 再発防止策策定"
        ;;
        
    "ddos")
        echo "## DDoS攻撃対応訓練"
        echo "シナリオ: Webサイトへの大規模DDoS攻撃"
        
        echo "対応チェックリスト:"
        echo "[ ] 攻撃パターン分析"
        echo "[ ] トラフィック制限実施"
        echo "[ ] CDN・WAF設定調整"
        echo "[ ] 上流ISPとの連携"
        echo "[ ] サービス継続性確保"
        ;;
esac

echo ""
echo "訓練完了後は以下を実施してください:"
echo "1. 対応時間の記録"
echo "2. 課題・改善点の洗い出し"
echo "3. 手順書の更新"
echo "4. 訓練レポートの作成"

echo "$(date): Incident response drill ($DRILL_TYPE) completed" >> /var/log/security/training.log
```

### 6.2 セキュリティ意識向上

#### 6.2.1 フィッシング訓練

```python
#!/usr/bin/env python3
# phishing_simulation.py

import smtplib
import random
from email.mime.text import MimeText
from email.mime.multipart import MimeMultipart
from datetime import datetime

class PhishingSimulation:
    def __init__(self):
        self.smtp_server = "localhost"
        self.smtp_port = 587
        self.sender_email = "security-training@example.com"
        
        self.templates = [
            {
                'subject': '【緊急】パスワード変更のお知らせ',
                'body': '''
お疲れ様です。

セキュリティ強化のため、パスワードの変更が必要です。
下記リンクより変更をお願いいたします。

[パスワード変更リンク]
http://fake-login.training.example.com/reset

※このメールは訓練用です。実際にクリックしないでください。
''',
                'risk_level': 'medium'
            },
            {
                'subject': '【重要】システムメンテナンスのお知らせ',
                'body': '''
システム管理者です。

緊急メンテナンスのため、一時的にログインが必要です。
下記よりアクセスしてください。

[システムアクセス]
http://fake-portal.training.example.com/login

※このメールは訓練用です。実際にクリックしないでください。
''',
                'risk_level': 'high'
            }
        ]
    
    def send_simulation_email(self, target_emails):
        """フィッシング訓練メール送信"""
        template = random.choice(self.templates)
        
        for email in target_emails:
            msg = MimeMultipart()
            msg['From'] = self.sender_email
            msg['To'] = email
            msg['Subject'] = template['subject']
            
            # 個人化
            body = template['body'].replace('[USER]', email.split('@')[0])
            msg.attach(MimeText(body, 'plain'))
            
            try:
                server = smtplib.SMTP(self.smtp_server, self.smtp_port)
                server.send_message(msg)
                server.quit()
                
                # ログ記録
                with open('/var/log/security/phishing_simulation.log', 'a') as f:
                    f.write(f"{datetime.now()}: Simulation email sent to {email}, template: {template['risk_level']}\n")
                    
            except Exception as e:
                print(f"Failed to send email to {email}: {e}")
    
    def track_clicks(self, log_file):
        """クリック率追跡"""
        clicked_users = set()
        
        # Webアクセスログからトレーニングサイトへのアクセスを検出
        with open(log_file, 'r') as f:
            for line in f:
                if 'training.example.com' in line:
                    # IPからユーザーを特定（簡易実装）
                    ip_match = re.search(r'(\d+\.\d+\.\d+\.\d+)', line)
                    if ip_match:
                        clicked_users.add(ip_match.group(1))
        
        return len(clicked_users)

if __name__ == "__main__":
    simulator = PhishingSimulation()
    
    # 対象ユーザーリスト
    target_users = [
        "user1@example.com",
        "user2@example.com", 
        "user3@example.com"
    ]
    
    print("フィッシング訓練を開始します...")
    simulator.send_simulation_email(target_users)
    print("訓練メールを送信しました。")
```

---

**注意**: このセキュリティ運用手順書は、システムのセキュリティ維持のための重要な手順を定義しています。実際の運用では最新の脅威情報に基づいて継続的に更新してください。
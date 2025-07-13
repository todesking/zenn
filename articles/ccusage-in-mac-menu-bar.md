---
title: "ccusageの出力をMacのメニューバーに表示する"
emoji: "📊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mac", "raycast", "xbar", "claudecode"]
published: true
---

RaycastかXbarというアプリを使うといいでしょう。

## Raycast

ランチャー+ファイラー+その他色々、プラグインで何でもできる便利アプリ。[公式ストア](https://www.raycast.com/nyatinte/ccusage)にプラグインが公開されてるので入れれば動く。動かない場合は設定画面でnpxの場所を指定するとよい

![](https://storage.googleapis.com/zenn-user-upload/3adc4319f15d-20250713.png)

![](https://storage.googleapis.com/zenn-user-upload/7e6b76843e33-20250713.png)

## [Xbar](https://xbarapp.com)

指定したスクリプトの出力をメニューバーに表示してくれる万能アプリ。公式ページにはClaude Code用のプラグインはなかったが、こんなもんAIに書かせればええんや。

```
`xbar/ccusage.1m.sh` として、ccusageコマンドの出力を表示するxbarプラグインを作りなさい。
xbarプラグインの仕様については https://raw.githubusercontent.com/matryer/xbar-plugins/refs/heads/main/CONTRIBUTING.md を参照せよ。
`npx ccusage daily --json` で取得できる本日のデータから、Input/Output tokensとCostを表示せよ。
```

このようなものができる。
![](https://storage.googleapis.com/zenn-user-upload/050f58c8ca4d-20250713.png)

スクリプトは以下。このままだと筆者の環境でしか動かない。npxコマンドの場所を指定する部分を、各自で自分の環境に合わせて調整してください。

```bash:~/Library/Application\ Support/xbar/plugins/ccusage.1m.sh
#!/usr/bin/env bash

# LICENSE: CC0

#  <xbar.title>ccusage</xbar.title>
#  <xbar.version>v1.0</xbar.version>
#  <xbar.author>todesking</xbar.author>
#  <xbar.desc>Display Claude Code daily usage statistics including tokens and cost</xbar.desc>

# Get current date in YYYYMMDD format
TODAY=$(date +%Y-%m-%d)

function exec_npx() {
  ASDF_NODE_VERSION=22.14.0 /opt/homebrew/bin/asdf exec npx "$@"
}

# Get ccusage data for today
JSON_OUTPUT=$(exec_npx ccusage daily --json 2>/dev/null)

if [ $? -ne 0 ] || [ -z "$JSON_OUTPUT" ]; then
    echo "CC Usage: Error"
    echo "---"
    echo "Failed to fetch usage data"
    exit 1
fi

# Extract today's data using jq
TODAY_DATA=$(echo "$JSON_OUTPUT" | jq --arg today "$TODAY" '.daily[] | select(.date == $today)')

if [ -z "$TODAY_DATA" ]; then
    echo "CC: No data"
    echo "---"
    echo "No usage data for today"
    exit 0
fi

# Extract values
TOTAL_TOKENS=$(echo "$TODAY_DATA" | jq -r '.totalTokens')
TOTAL_COST=$(echo "$TODAY_DATA" | jq -r '.totalCost')
INPUT_TOKENS=$(echo "$TODAY_DATA" | jq -r '.inputTokens')
OUTPUT_TOKENS=$(echo "$TODAY_DATA" | jq -r '.outputTokens')
CACHE_CREATION=$(echo "$TODAY_DATA" | jq -r '.cacheCreationTokens')
CACHE_READ=$(echo "$TODAY_DATA" | jq -r '.cacheReadTokens')

# Format numbers with scientific notation (k, m)
format_scientific() {
    local num=$1
    if [ "$num" -ge 1000000 ]; then
        printf "%.1fm" $(echo "scale=1; $num / 1000000" | bc)
    elif [ "$num" -ge 1000 ]; then
        printf "%.1fk" $(echo "scale=1; $num / 1000" | bc)
    else
        echo "$num"
    fi
}

# Format cost to 2 decimal places
COST_FORMATTED=$(printf "%.2f" "$TOTAL_COST")

# Menu bar display (compact)
echo "$(format_scientific $INPUT_TOKENS)/$(format_scientific $OUTPUT_TOKENS) \$$COST_FORMATTED"
echo "---"

# Dropdown details
echo "Claude Code Usage - $TODAY"
echo "---"
echo "Total Tokens: $(format_scientific $TOTAL_TOKENS)"
echo "Total Cost: \$$COST_FORMATTED"
echo "---"
echo "Input Tokens: $(format_scientific $INPUT_TOKENS)"
echo "Output Tokens: $(format_scientific $OUTPUT_TOKENS)"
echo "Cache Creation: $(format_scientific $CACHE_CREATION)"
echo "Cache Read: $(format_scientific $CACHE_READ)"
echo "---"
echo "Refresh | refresh=true"
```

おしまい。

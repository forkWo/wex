name: 更新 Spider

on:
  push:
    branches:
      - main
  workflow_dispatch:
  schedule:
    - cron: '0 */1 * * *'  # 每小时执行一次

jobs:
  update-spider:
    runs-on: ubuntu-latest

    steps:
      # Step 0: Delete previous runs
      - name: 删除任务流
        env:
          PAT: ${{ secrets.PAT }}
        run: |
          runs_to_delete=$(curl -s -H "Authorization: token $PAT" \
            https://api.github.com/repos/${{ github.repository }}/actions/runs?branch=${{ github.ref_name }} \
            | jq -r '.workflow_runs | map(select(.status == "completed")) | .[23:] | .[].id')

          for run_id in $runs_to_delete; do
            echo "Deleting run $run_id"
            curl -X DELETE -H "Authorization: token $PAT" \
            https://api.github.com/repos/${{ github.repository }}/actions/runs/$run_id
          done

      # Step 1: Checkout the repository
      - name: 查看 repository
        uses: actions/checkout@v2

      # Step 2: Setup Python
      - name: 设置 Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.x'

      # Step 3: Install necessary dependencies
      - name: 安装 Dependencies
        run: python -m pip install requests

      # Step 4: Get Spider URL and set environment variable
      - name: 获取 Spider 字段
        run: |
          SPIDER_URL=$(python -c "import requests; print(requests.get('https://9280.kstore.space/wex.json').json()['spider'])")
          echo "SPIDER_URL=$SPIDER_URL" >> $GITHUB_ENV

      # Step 4.5: Compare with existing Spider URL
      - name: 对比现有 Spider URL
        id: compare-urls
        env:
          SPIDER_JSON: ${{ secrets.SPIDER_JSON }}
        run: |
          # 获取现有的 Spider URL
          EXISTING_SPIDER_URL=$(python -c "import requests; print(requests.get('${{ env.SPIDER_JSON }}').json()['spider'].strip())")
          
          # 去除新获取的 Spider URL 中的换行符和空格
          SPIDER_URL=$(echo $SPIDER_URL | tr -d '\n' | tr -d ' ')
          EXISTING_SPIDER_URL=$(echo $EXISTING_SPIDER_URL | tr -d '\n' | tr -d ' ')

          # 输出两者以调试
          echo "New SPIDER_URL: $SPIDER_URL"
          echo "Existing SPIDER_URL: $EXISTING_SPIDER_URL"
          
          # 比较两者
          if [ "$SPIDER_URL" = "$EXISTING_SPIDER_URL" ]; then
            echo "continue=false" >> $GITHUB_ENV
            echo "对比结果一致。停止脚本."
          else
            echo "continue=true" >> $GITHUB_ENV
            echo "对比结果不同。继续脚本."
          fi

      # Step 5: Update Spider Field and upload to Kstore
      - name: 更新 Spider 字段、并上传 Kstore
        if: env.continue == 'true'
        env:
          KSTORE_TOKEN: ${{ secrets.KSTORE_TOKEN }}
          KSTORE_TOKEN2: ${{ secrets.KSTORE_TOKEN2 }}
          KSTORE_ID: ${{ secrets.KSTORE_ID }}
          WEX_JSON: ${{ secrets.WEX_JSON }}
        run: |
          # Fetch existing data
          response=$(curl -s ${{ env.WEX_JSON }})
          
          # Update only the 'spider' field with the new value
          updated_data=$(echo "$response" | python -c "import sys, json; data = json.load(sys.stdin); data['spider'] = '${SPIDER_URL}'; print(json.dumps(data, ensure_ascii=False))")

          # Save updated data to spider.json
          cd ${{ github.workspace }}
          echo "$updated_data" > spider.json
          
          # Upload the updated spider.json to Kstore
          curl -F "file=@spider.json" "https://upload.kstore.space/upload/${{ env.KSTORE_ID }}?access_token=${{ env.KSTORE_TOKEN2 }}"

          # Save updated data to wex.json
          echo "$updated_data" > wex.json
          
          # Upload the updated wex.json to Kstore
          curl -F "file=@wex.json" "https://upload.kstore.space/upload/${{ env.KSTORE_ID }}?access_token=${{ env.KSTORE_TOKEN }}"

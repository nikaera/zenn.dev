---
title: "GitHub Actions で Azure Functions にデプロイする"
---

Azure Functions へのデプロイも GitHub Actions で行えるようにするため、GitHub Actions から Azure Functions へのデプロイ時に必要となる関数アプリの発行プロファイルを下記からダウンロードします。

![関数アプリの発行プロファイルを取得する](https://i.gyazo.com/0eafcad37d032cb4e4203d63a272185a.png)
*関数アプリの発行プロファイルをダウンロードする*

発行プロファイルを取得したら、ダウンロードしたファイル内容をコピー&ペーストで GitHub の Secrets に登録します。

![2. GitHub リポジトリの Secrets の登録画面に遷移する](https://i.gyazo.com/4f209679f639f5866fd3213d99758078.png)
*2. GitHub リポジトリの Secrets の登録画面に遷移する*

![3. GitHub リポジトリの Secrets に発行プロファイルを登録する](https://i.gyazo.com/943fb9c48763c238afb47b4d986edc51.png)
*3. GitHub リポジトリの Secrets に発行プロファイルを登録する*

上記が完了したら Azure Functions へデプロイするための GitHub Actions のワークフローファイル `.github/workflows/deploy-azure.yml` を作成して、`main` ブランチに push します。

```yml:.github/workflows/deploy-azure.yml
# main ブランチに何らかの commit が追加されたら走らせる
on:
  push:
    branches:
      - main

env:
  AZURE_FUNCTIONAPP_NAME: azure-nestjs-sample
  AZURE_FUNCTIONAPP_PACKAGE_PATH: '.'
  NODE_VERSION: '10.x'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@master

    - name: Setup Node ${{ env.NODE_VERSION }} Environment
      uses: actions/setup-node@v1
      with:
        node-version: ${{ env.NODE_VERSION }}
    - name: 'Resolve Project Dependencies Using Npm'
      shell: bash
      run: |
        pushd './${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}'
        npm install
        npm run build --if-present
        popd
    - name: 'Run Azure Functions Action'
      uses: Azure/functions-action@v1
      id: fa
      with:
        app-name: ${{ env.AZURE_FUNCTIONAPP_NAME }}
        package: ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}
        publish-profile: ${{ secrets.AZURE_FUNCTIONAPP_PUBLISH_PROFILE }}

# For more samples to get started with GitHub Action workflows to deploy to Azure, refer to https://github.com/Azure/actions-workflow-samples
```

main ブランチに push 後、実際に GitHub Actions が動作しているか Actions タブから確認します。

![main ブランチを更新した後、GitHub Actions を確認する](https://i.gyazo.com/b5e30f42271bc526d4bfbafe1156d4ba.png)
*main ブランチが更新された後、GitHub Actions を確認する*

:::message
関数アプリのアプリケーション設定に `WEBSITE_RUN_FROM_PACKAGE` が設定されていると、デプロイに失敗する時があります。失敗したらアプリケーション設定から `WEBSITE_RUN_FROM_PACKAGE` を削除して再度 GitHub Actions を実行してみてください。
:::

最後にデプロイした先の API が正常に動作していそうか `curl` で確認してみます。デフォルトでは [HTTP Trigger の authLevel](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=csharp)は `anonymouse` になっているので URL 直打ちでアクセス可能です。

:::message
[PlayFab の CloudScript](https://docs.microsoft.com/ja-jp/gaming/playfab/features/automation/cloudscript-af/quickstart) から Azure Functions を実行する際の authLevel は `function` が推奨されています。また、CloudScript から Azure Functions を呼び出す際は必ず POST Method で HTTP リクエストされます。
:::

```bash
# POST events でデータ登録出来るか確認してみる
curl -X POST -H "Content-Type: application/json" -d '{"name": "test-event"}' https://azure-nestjs-sample.azurewebsites.net/api/events

{"_id":"5f963d6d2867990052b9bac8","name":"test-event","__v":0}
```

![Azure Cosmos DB にデータが正常に登録されていることを確認する](https://i.gyazo.com/05734976034a2955a0e23bc12c9cbf64.png)
*Azure Cosmos DB にデータが正常に登録されていることを確認する*

無事にデータが登録されていることが確認出来れば作業完了です。お疲れさまでした。

# Solution Server を後から有効にする（推奨手順）

`helm/values.yaml` の `components.operator1.solutionServer` に **コメント付きのひな形**を置いておき、MaaS などで **モデル名・Base URL が確定したあと**に編集して Git に push する運用です。

**API Key は Git に書きません。** クラスタ上の `kai-api-keys` Secret にだけ置きます。

---

## 前提で読むファイル

- **`helm/values.yaml`** … `solutionServer` ブロック（コメントの例付き）

---

## 手順（概要）

1. **初回**: `enabled: false` のままデプロイ（Konveyor Operator / Tackle のみ。Solution Server は未起動）
2. **値が確定したら**（モデル・URL・API Key）:
   - 先に **`kai-api-keys` Secret** を `konveyor-tackle` に作成
   - **`helm/values.yaml`** でコメントを外し、`enabled: true` と各フィールドを埋める
3. **Git に commit & push** → ArgoCD が同期 → Tackle CR 更新 → Operator が Kai を起動
4. 下記の **確認コマンド**で状態を見る

---

## 1. Secret を先に作成（API Key はここだけ）

```bash
oc create secret generic kai-api-keys -n konveyor-tackle \
  --from-literal=OPENAI_API_KEY='<API Key>' \
  --from-literal=OPENAI_API_BASE='https://your-maas.example.com/v1'
```

- `OPENAI_API_BASE` は OpenAI 互換 API（Parasol MaaS 等）のときに指定。公式 OpenAI だけなら省略可の場合あり。
- 既に同名 Secret がある場合は `oc delete secret kai-api-keys -n konveyor-tackle` してから作り直すか、`oc create secret ... --dry-run=client -o yaml | oc apply -f -` で更新方針を決める。

---

## 2. `helm/values.yaml` を編集

`components.operator1.solutionServer` を、リポジトリ内の例に沿って更新します。

- `enabled: true` に変更
- コメントアウトされていた行を外し、実際の **モデル名・Base URL** を入れる
- `existingSecretName: kai-api-keys` を有効にする（上で作った Secret 名と一致させる）
- `llmProvider` がコメントのみの場合は、`openai` 等を明示する

**Git に載せないもの:** `apiKey` フィールド（機密）。Secret + `existingSecretName` で足ります。

参照用スニペット（実際の値は環境に合わせる）:

```yaml
    solutionServer:
      enabled: true
      llmProvider: "openai"
      llmModel: "your-model-name"
      llmBaseUrl: "https://your-maas.example.com/v1"
      existingSecretName: kai-api-keys
```

---

## 3. push と同期

```bash
git add helm/values.yaml
git commit -m "Enable Konveyor Solution Server (llm model/url)"
git push
```

ArgoCD が自動同期するまで待つか、UI から **Refresh / Sync**。

---

## 4. 注入後の確認

```bash
oc get tackle tackle -n konveyor-tackle -o yaml | grep -A8 'kai_'
oc get secret kai-api-keys -n konveyor-tackle
oc get pods -n konveyor-tackle | grep kai
```

`KaiSolutionServerReady` が True になり、`kai-api` / `kai-db` の Pod が Running になればよい目安です。

---

## VS Code 拡張（参考）

`konveyor.solutionServer.url` には **tackle の Route（tackle-ui）の HTTPS URL** を指定します（専用の kai Route は通常不要）。

---

## 補足: 別の注入方法（任意）

| 方法 | 用途 |
|------|------|
| `argocd app set -p operator1.solutionServer...` | CLI で Application に Helm パラメータを足す。**親の App-of-Apps が子を再生成すると消える**ことがあるので、このリポジトリの運用では非推奨 |
| `oc patch application ...` | 上と同様。JSON が長くなりがち |
| Git で `helm/values.yaml` を更新（このドキュメントの手順） | **推奨**。記述がシンプルで、追跡しやすい |

---

## トラブルシューティング

- **kai Pod が出ない**: Secret が `konveyor-tackle` にあるか、`existingSecretName` と名前が一致しているか
- **Tackle の condition が False**: `oc describe tackle tackle -n konveyor-tackle` の message を確認
- **再同期を促す**: `oc patch tackle tackle -n konveyor-tackle --type=merge -p '{"metadata":{"annotations":{"konveyor.io/force-reconcile":"'$(date +%s)'"}}}'`

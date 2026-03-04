# 📚 OpenClaw Model Switching Guide: Raspberry Pi 5

**Languages / 語言 / 言語**
- [English](#-openclaw-model-switching-guide-raspberry-pi-5)
- [日本語 (Japanese)](#-openclaw-モデル切り替えガイドraspberry-pi-5-ラズベリーパイ5)
- [繁體中文 (Traditional Chinese)](#-openclaw-模型切換指南raspberry-pi-5-樹莓派5)

This guide provides two distinct deployment architectures. **Option 1 (Primary)** offloads the processing to the cloud for maximum speed and intelligence. **Option 2 (Alternative)** keeps your data 100% private by running an optimized local model.

This guide assumes you have already installed the base OpenClaw framework and assembled your hardware according to the Edge AI Architecture Report.

---

## ☁️ Option 1 (Primary Solution): Cloud Deployment (OpenAI GPT-5.3 Codex)

**Best for:** Lightning-fast responses, zero hardware strain on the Raspberry Pi, and utilizing OpenAI's most advanced reasoning model (GPT-5.3).

### ⚠️ Authentication Concept: Why You Should NOT Use an API Key
When setting up OpenAI in OpenClaw, **do not generate a standard OpenAI API Key.** 
* **The Problem:** OpenAI API keys are strictly pay-as-you-go based on token usage. An autonomous agent can rack up unexpected usage bills very quickly.
* **The Solution (ChatGPT OAuth):** OpenClaw supports **OpenAI Codex (ChatGPT OAuth)**. This securely links OpenClaw directly to your personal ChatGPT account, allowing your autonomous agent to pull entirely from your existing, flat-rate ChatGPT Plus subscription—meaning zero surprise API bills!

> 💡 **Tip: How to get a 1-Month Free Trial of ChatGPT Plus**
> If you do not have a Plus subscription, OpenAI frequently runs promotional campaigns offering a 1-month free trial in select regions. Log into your free ChatGPT account, click your profile, and select "Upgrade to Plus" to look for the "Claim free offer" prompt. *(Remember to cancel before 30 days if you do not want the auto-renewal charge).*

### Step-by-Step Implementation

#### Step 1: Stop the OpenClaw Gateway & Disable Local Inference
* **What it does:** Safely halts OpenClaw and permanently turns off the Ollama service.
* **Why it is necessary:** You cannot run the configuration wizard while OpenClaw is actively running. (Optional) Disabling Ollama frees up the Raspberry Pi's CPU and RAM to ensure maximum network stability.
```bash
openclaw gateway stop
sudo systemctl stop ollama
sudo systemctl disable ollama
```

#### Step 2: Run the OpenClaw Configuration Wizard
* **What it does:** Launches the interactive command-line setup tool.
* **Why it is necessary:** The built-in wizard safely manages the complex OAuth secure token exchange so you don't have to edit JSON files manually.
```bash
openclaw configure
```

#### Step 3: Navigate the Wizard to OpenAI Codex
* **What it does:** Selects the specific authentication routing for your ChatGPT account.
* **Why it is necessary:** This tells OpenClaw to bypass standard API key billing and use your subscription.
1. In the interactive menu, select **Models & Auth**.
2. Select **Add/Update Provider**.
3. Choose **OpenAI**.
4. Select **OpenAI Codex (ChatGPT OAuth)**.

#### Step 4: Authorize Your Account
* **What it does:** Connects the Raspberry Pi to your ChatGPT account via a secure, refreshable token (PKCE).
* **Why it is necessary:** Allows your Pi to act on your behalf without storing your actual password.
1. The wizard will output a secure URL (e.g., `https://auth.openai.com/oauth/authorize?...`).
2. Copy this URL and open it in a web browser on your PC or phone.
3. Log into your ChatGPT account and click **Authorize**.
4. The browser will be redirected to a brand new page with localhost url.
5. Copy and paste the browser url back to the wizard and press enter. 

#### Step 5: Set GPT-5.3 as the Primary Model
* **What it does:** Instructs OpenClaw's default agent to use the latest flagship model.
* **Why it is necessary:** By default, OpenClaw might fall back to an older model if not explicitly told otherwise.
1. Still inside the wizard, go to **Agents -> Default Agent -> Primary Model**.
2. Select `openai-codex/gpt-5.3` from the list.
3. Exit and save the wizard.

#### Step 6: Validate and Start the Gateway
* **What it does:** Checks the system for errors and boots your cloud-connected AI agent.
* **Why it is necessary:** Applies the new OAuth credentials and routing permanently.
```bash
openclaw doctor --fix
openclaw gateway start
```

---

## 🏗️ Option 2 (Alternative Solution): Local Deployment (Ollama + Qwen2.5:3B)

**Best for:** Maximum privacy, offline capabilities, and zero recurring cloud subscription costs. 

### Step-by-Step Implementation

#### Step 1: Upgrade NVMe Swap Memory to 16GB
* **What it does:** Replaces your existing 8GB virtual memory file with a massive 16GB file on your high-speed NVMe SSD.
* **Why it is necessary:** Loading local models requires copying huge gigabytes of data into RAM. 16GB of swap ensures the Raspberry Pi never suffers an Out-Of-Memory (OOM) crash during this process.
```bash
sudo swapoff -v /swapfile
sudo rm /swapfile
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```
check
```bash
free -h
```

#### Step 2: Create a Hardware-Optimized AI Model (`qwen-pi`)
* **Important notice:** Running local AI model for agentic workflow is only doable but not feasible. Limited resouces make it slow and useless. Total not recommed.
* **What it does:** Compiles a custom version of Qwen2.5:3B specifically throttled for the Raspberry Pi.
* **Why it is necessary:** By limiting the AI to 3 CPU threads (`num_thread`) and restricting its short-term memory (`num_ctx 4096`), we prevent the Pi's CPU from freezing and ensure network tasks don't time out.
```bash
sudo systemctl enable ollama
sudo systemctl start ollama
ollama pull qwen2.5-coder:0.5b

cat <<EOF > Modelfile
FROM qwen2.5-coder:0.5b
PARAMETER num_thread 3
PARAMETER num_ctx 4096
EOF

ollama create qwen-pi -f Modelfile
```

check the name of the modified qwen model:
```bash
ollama list
```

#### Step 3: Stop the Gateway
* **What it does:** Safely shuts down the OpenClaw service.
* **Why it is necessary:** Prevents configuration file corruption while editing.
```bash
openclaw gateway stop
```

#### Step 4: Update OpenClaw Configuration (Clean Schema)
* **What it does:** Points OpenClaw to your local Ollama instance using strictly validated JSON.
* **Why it is necessary:** This routing tells the system where to find the local model while respecting OpenClaw's strict schema rules (no invalid `timeout` keys).

Open the configuration file:
```bash
nano ~/.openclaw/openclaw.json
```
Replace the models and agents with following settings (*correct the model name with the model you run on ollama)
```json
"models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://127.0.0.1:11434/v1",
        "apiKey": "ollama-local",
        "api": "openai-completions",
        "models":[
          { "id": "Put your modifed AI model's name here", "name": "Put your modifed AI model's name here" }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": { "primary": "ollama/Put your modifed AI model's name here" },
      "models": { "ollama/qwen-pi": {} },
      "workspace": "/home/pi/.openclaw/workspace",
      "timeoutSeconds": 600
    }
  }
```
*(Save by pressing `Ctrl+O`, `Enter`, `Ctrl+X`)*.

#### Step 5: Validate and Start
* **What it does:** Runs the doctor to ensure the schema is perfect, then starts the bot.
* **Why it is necessary:** Ensures stability before running in the background.
```bash
openclaw doctor --fix
openclaw gateway start
```

#### Step 6: The "Pre-Warming" Trick (Crucial for Option 2)
* **What it does:** Forces Ollama to load the model into the 16GB swap *before* you ever message the Telegram bot.
* **Why it is necessary:** Because we cannot change OpenClaw's hardcoded 2-minute timeout, the Raspberry Pi will fail if it tries to cold-start the model upon receiving your first Telegram message. 
Run this command in the Pi terminal to pre-load the model into memory indefinitely:
```bash
curl http://127.0.0.1:11434/api/generate -d '{"model": "qwen-pi", "keep_alive": -1}'
```
*(Once this completes, your Telegram bot will reply flawlessly without timing out!)*

---

## 📝 Summary

You now have two fully functional pathways:
1. **Cloud Approach:** By utilizing OpenClaw's interactive `configure` tool to authenticate via **OpenAI Codex (ChatGPT OAuth)**, you eliminate all API costs by routing requests through your existing ChatGPT Plus subscription directly to GPT-5.3. 
2. **Local Approach:** By applying a pure, schema-compliant `openclaw.json` setup, expanding NVMe swap to 16GB, and running a background `keep_alive` curl command to pre-warm the model, you completely bypass the hardware "Cold Start" timeouts that crash standard local deployments.

---

# 📚 OpenClaw モデル切り替えガイド：Raspberry Pi 5 (ラズベリーパイ5)

このガイドでは、2つの異なる展開（導入や配置）アーキテクチャの方法を説明します。**オプション1（推奨・メインの解決策）**は、最高のスピードと賢さを実現するために、処理をクラウド（インターネット上のサーバー）に任せます。**オプション2（代替の解決策）**は、最適化されたローカルモデル（お使いの機器内で動くAIモデル）を動かすことで、データを完全にプライベート（非公開）な状態に保ちます。

このガイドは、すでにOpenClaw（オープンクロー）の基本システムをインストール済みで、Edge AI Architecture Report（エッジAIアーキテクチャレポート）に従ってハードウェアの組み立てが完了していることを前提としています。

---

## ☁️ オプション1（メインの解決策）: クラウド展開 (OpenAI GPT-5.3 Codex)

**こんな方におすすめ:** 非常に速い応答を求めている方、Raspberry Pi（ラズベリーパイ）本体に負荷をかけたくない方、そしてOpenAIの最新の推論モデル（GPT-5.3）を使用したい方。

### ⚠️ 認証の概念: APIキーを使用すべきではない理由
OpenClawでOpenAIを設定する際、**通常のOpenAI APIキー（従量課金制のパスワードのようなもの）は作成しないでください。**
* **問題点:** OpenAIのAPIキーは、トークン（文字数）の使用量に基づく完全な従量課金制です。自動で動くAIエージェントを使用すると、予想外の請求がすぐに膨れ上がる可能性があります。
* **解決策 (ChatGPT OAuth - オーオース):** OpenClawは**OpenAI Codex (ChatGPT OAuth)**をサポートしています。これにより、OpenClawをあなたの個人のChatGPTアカウントに直接かつ安全にリンクさせることができます。つまり、すでに支払っている定額制のChatGPT Plus（有料プラン）の枠内でAIを動かせるため、予期せぬAPIの追加請求がゼロになります！

> 💡 **ヒント: ChatGPT Plusの1ヶ月無料トライアルの取得方法**
> Plus（有料）サブスクリプションをお持ちでない場合、OpenAIは一部の地域で1ヶ月無料トライアルのキャンペーンを実施していることがあります。無料のChatGPTアカウントにログインし、プロフィールをクリックして「Upgrade to Plus（Plusにアップグレード）」を選択し、「Claim free offer（無料オファーを受け取る）」の案内があるか探してみてください。*（自動更新の請求を避けたい場合は、30日以内にキャンセルすることを忘れないでください）。*

### 導入の手順

#### ステップ1: OpenClaw Gateway（ゲートウェイ）の停止とローカル推論の無効化
* **これは何をするか:** OpenClawを安全に停止し、Ollama（オラマ：ローカルでAIを動かすためのソフトウェア）サービスを完全にオフにします。
* **なぜ必要か:** OpenClawが稼働中の状態では、設定ウィザード（対話型の設定ツール）を実行できません。（オプションとして）Ollamaを無効化することで、Raspberry PiのCPUとRAM（メモリ）を解放し、ネットワークの安定性を最大限に高めます。
```bash
openclaw gateway stop
sudo systemctl stop ollama
sudo systemctl disable ollama
```

#### ステップ2: OpenClaw設定ウィザードの実行
* **これは何をするか:** 対話型のコマンドライン設定ツールを起動します。
* **なぜ必要か:** 組み込みのウィザードが、複雑なOAuth（オーオース）の安全なトークン交換（認証の手続き）を管理してくれるため、手動でJSONファイル（設定ファイル）を編集する手間が省けます。
```bash
openclaw configure
```

#### ステップ3: ウィザードでOpenAI Codexへナビゲートする
* **これは何をするか:** ChatGPTアカウントの特定の認証ルートを選択します。
* **なぜ必要か:** これにより、標準のAPIキーの請求を回避し、代わりにあなたのサブスクリプションを使用するようOpenClawに指示します。
1. 対話型メニューで、**Models & Auth（モデルと認証）**を選択します。
2. **Add/Update Provider（プロバイダーの追加/更新）**を選択します。
3. **OpenAI**を選択します。
4. **OpenAI Codex (ChatGPT OAuth)**を選択します。

#### ステップ4: アカウントの承認
* **これは何をするか:** 安全で更新可能なトークン(PKCE)を介して、Raspberry PiをChatGPTアカウントに接続します。
* **なぜ必要か:** Raspberry Piが実際のパスワードを保存することなく、あなたの代わりに操作できるようにするためです。
1. ウィザードは安全なURLを出力します（例: `https://auth.openai.com/oauth/authorize?...`）。
2. このURLをコピーし、PCまたはスマートフォンのWebブラウザで開きます。
3. ChatGPTアカウントにログインし、**Authorize（承認する）**をクリックします。
4. ブラウザがlocalhost（ローカルホスト：自分のPCのアドレス）の新しいページにリダイレクト（転送）されます。
5. ブラウザのURLをコピーしてウィザードに貼り付け、Enterキーを押します。

#### ステップ5: GPT-5.3をメインモデルとして設定する
* **これは何をするか:** OpenClawのデフォルト（初期設定）エージェントに、最新の主力モデルを使用するよう指示します。
* **なぜ必要か:** 明示的に指定しない場合、OpenClawは古いモデルを優先して使用する可能性があるためです。
1. ウィザード内で、**Agents（エージェント） -> Default Agent（デフォルトエージェント） -> Primary Model（プライマリモデル）**へと進みます。
2. リストから `openai-codex/gpt-5.3` を選択します。
3. 終了して設定を保存します。

#### ステップ6: 検証とゲートウェイの起動
* **これは何をするか:** システムにエラーがないかチェックし、クラウド接続されたAIエージェントを起動します。
* **なぜ必要か:** 新しいOAuth（オーオース）の認証情報と通信経路を永続的に適用するためです。
```bash
openclaw doctor --fix
openclaw gateway start
```

---

## 🏗️ オプション2（代替の解決策）: ローカル展開 (Ollama + Qwen2.5:3B)

**こんな方におすすめ:** 最大限のプライバシー保護、オフライン（インターネット接続なし）での使用、そして継続的なクラウドの定額費用をかけたくない方。

### 導入の手順

#### ステップ1: NVMe Swap（スワップ）メモリを16GBにアップグレードする
* **これは何をするか:** 既存の8GBの仮想メモリファイル（ストレージをメモリの代わりとして使う仕組み）を、高速なNVMe SSD上の巨大な16GBのファイルに置き換えます。
* **なぜ必要か:** ローカルモデルを読み込むには、数ギガバイトもの膨大なデータをRAM（メモリ）にコピ​​ーする必要があります。16GBのスワップを確保することで、このプロセス中にRaspberry Piがメモリ不足(OOM: Out-Of-Memory)でクラッシュするのを防ぎます。
```bash
sudo swapoff -v /swapfile
sudo rm /swapfile
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```
確認コマンド:
```bash
free -h
```

#### ステップ2: ハードウェアに最適化されたAIモデル(`qwen-pi`)を作成する
* **重要な注意:** ローカルのAIモデルを自動ワークフロー（自律的な作業処理）で動かすことは「可能」ではありますが、実用的ではありません。機器の制限により動作が遅くなり役に立たないため、全く推奨されません。
* **これは何をするか:** Raspberry Pi用に特別に制限をかけたQwen2.5:3Bのカスタムバージョン（改良版）を作成します。
* **なぜ必要か:** AIを3つのCPUスレッド(`num_thread`)に制限し、短期記憶(`num_ctx 4096`)を制限することで、PiのCPUがフリーズ（固まること）するのを防ぎ、ネットワークの通信がタイムアウト（時間切れ）にならないようにします。
```bash
sudo systemctl enable ollama
sudo systemctl start ollama
ollama pull qwen2.5-coder:0.5b

cat <<EOF > Modelfile
FROM qwen2.5-coder:0.5b
PARAMETER num_thread 3
PARAMETER num_ctx 4096
EOF

ollama create qwen-pi -f Modelfile
```

変更されたqwenモデルの名前を確認します:
```bash
ollama list
```

#### ステップ3: ゲートウェイの停止
* **これは何をするか:** OpenClawサービスを安全にシャットダウンします。
* **なぜ必要か:** 編集中の設定ファイルの破損を防ぎます。
```bash
openclaw gateway stop
```

#### ステップ4: OpenClaw設定の更新（クリーンなスキーマ）
* **これは何をするか:** 厳重にチェックされたJSON（データ記述言語）を使用して、OpenClawをローカルのOllamaプログラムに接続します。
* **なぜ必要か:** この設定により、OpenClawの厳格なスキーマルール（データの構造ルール。例えば無効な`timeout`キーは不可など）を遵守しながら、システムにローカルモデルの場所を教えることができます。

設定ファイルを開きます:
```bash
nano ~/.openclaw/openclaw.json
```
モデルとエージェントの名前を以下の設定に置き換えてください（*モデル名はOllamaで実行している実際の名前に合わせて修正してください）
```json
"models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://127.0.0.1:11434/v1",
        "apiKey": "ollama-local",
        "api": "openai-completions",
        "models":[
          { "id": "ここに修正したAIモデルの名前を入力", "name": "ここに修正したAIモデルの名前を入力" }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": { "primary": "ollama/ここに修正したAIモデルの名前を入力" },
      "models": { "ollama/qwen-pi": {} },
      "workspace": "/home/pi/.openclaw/workspace",
      "timeoutSeconds": 600
    }
  }
```
*（キーボードの `Ctrl+O`、`Enter`、`Ctrl+X`の順に押して保存・終了します）*。

#### ステップ5: 検証と起動
* **これは何をするか:** doctor（診断機能）を実行して設定の構造が完璧であることを確認し、ボットを起動します。
* **なぜ必要か:** 裏側(バックグラウンド)で実行する前に安定性を確保するためです。
```bash
openclaw doctor --fix
openclaw gateway start
```

#### ステップ6: 「プレウォーミング（事前準備）」のトリック（オプション2に不可欠）
* **これは何をするか:** Telegramボットにメッセージを送る*前に*、Ollamaにモデルを16GBのスワップメモリに読み込ませます。
* **なぜ必要か:** OpenClawに固定されている2分間のタイムアウト時間（待ち時間の限界）を変更できないため、最初のメッセージを受け取ってからモデルをコールドスタート（ゼロからのゼロからの起動）させようとすると、Raspberry Piは失敗してしまいます。
以下のコマンドをPiのターミナル（黒い画面）で実行し、モデルを無期限にメモリに事前読み込みします。
```bash
curl http://127.0.0.1:11434/api/generate -d '{"model": "qwen-pi", "keep_alive": -1}'
```
*（これが完了すれば、Telegramボットはタイムアウトすることなく完璧に応答します！）*

---

## 📝 まとめ

現在、完全に機能する2つの方法があります：
1. **クラウド方式:** OpenClawの対話型`configure`（設定）ツールを利用し、**OpenAI Codex (ChatGPT OAuth - オーオース)**経由で認証することで、既存のChatGPT Plusサブスクリプションを利用してリクエストをGPT-5.3に直接転送し、APIの追加コストを完全になくすことができます。
2. **ローカル方式:** 構造が正確な`openclaw.json`設定を適用し、NVMeスワップを16GBに拡張し、さらにバックグラウンドで`keep_alive`つきのcurlコマンドを実行してモデルを事前準備（プレウォーム）することで、通常のローカル環境で発生しがちなハードウェアの「コールドスタート」（起動時の待ち時間による時間切れ）を完全に回避できます。

---

# 📚 OpenClaw 模型切換指南：Raspberry Pi 5 (樹莓派5)

本指南提供兩種截然不同的部署架構（安裝執行方式）。**選項1（主要方案）**將處理任務轉移到雲端，以獲得最快的速度和最高的智慧。**選項2（替代方案）**藉由執行經過最佳化的本地端模型（在裝置內執行的AI模型），讓您的資料維持100%隱私。

本指南假設您已經安裝好基礎的 OpenClaw 框架，並根據 Edge AI Architecture Report (邊緣AI架構報告) 完成了硬體組裝。

---

## ☁️ 選項1（主要方案）：雲端部署 (OpenAI GPT-5.3 Codex)

**最適合：** 追求極致的反應速度、不希望對 Raspberry Pi 造成任何硬體負擔，並想使用 OpenAI 最先進的推理模型 (GPT-5.3) 的使用者。

### ⚠️ 認證概念：為什麼不應該使用 API 金鑰
在 OpenClaw 中設定 OpenAI 時，**請不要產生標準的 OpenAI API 金鑰（一種依使用量計費的密碼）。**
* **問題點：** OpenAI API 金鑰完全是基於 Token（字數）用量的隨收隨付（用多少付多少）機制。一個自動運作的 AI 代理 (Agent) 很容易在短時間內累積出超乎預期的帳單。
* **解決方案 (ChatGPT OAuth - 開放授權)：** OpenClaw 支援 **OpenAI Codex (ChatGPT OAuth)**。這讓 OpenClaw 可以安全地直接連接到您個人的 ChatGPT 帳號，代表您的自動 AI 代理可以直接運用您現有的 ChatGPT Plus（付費方案）吃到飽額度，不會有任何意外的 API 帳單！

> 💡 **提示：如何取得 1 個月的 ChatGPT Plus 免費試用**
> 如果您沒有 Plus 訂閱，OpenAI 經常在特定地區推出 1 個月免費試用的促銷活動。您可以登入免費的 ChatGPT 帳號，點擊您的個人資料，選擇「Upgrade to Plus (升級至 Plus)」，看看是否有「Claim free offer (領取免費優惠)」的提示。*（請記得，如果您不想被自動扣款，請在 30 天內取消訂閱）。*

### 逐步實作

#### 步驟 1：停止 OpenClaw Gateway (閘道器) 並停用本地端推論
* **這是做什麼：** 安全地停止 OpenClaw 通訊，並永久關閉 Ollama（在本地端執行AI的軟體）服務。
* **為什麼需要：** 在 OpenClaw 正在運行的狀態下，無法執行設定精靈（互動式設定工具）。（選擇性）停用 Ollama 可釋放出 Raspberry Pi 的 CPU 和 RAM（記憶體），確保最高的網路穩定度。
```bash
openclaw gateway stop
sudo systemctl stop ollama
sudo systemctl disable ollama
```

#### 步驟 2：執行 OpenClaw 設定精靈
* **這是做什麼：** 啟動互動式的指令列設定工具。
* **為什麼需要：** 內建的設定精靈會安全地管理複雜的 OAuth 安全權杖交換（認證流程），讓您不需手動編輯 JSON 檔案（設定檔）。
```bash
openclaw configure
```

#### 步驟 3：在設定精靈中選擇 OpenAI Codex
* **這是做什麼：** 為您的 ChatGPT 帳號選擇特定的認證路徑。
* **為什麼需要：** 此舉將告知 OpenClaw 避開標準的 API 金鑰計費機制，轉而使用您的現有訂閱。
1. 在互動選單中，選擇 **Models & Auth (模型與認證)**。
2. 選擇 **Add/Update Provider (新增/更新提供者)**。
3. 選擇 **OpenAI**。
4. 選擇 **OpenAI Codex (ChatGPT OAuth)**。

#### 步驟 4：授權您的帳號
* **這是做什麼：** 透過安全的、可更新的權杖 (PKCE)，將 Raspberry Pi 連接到您的 ChatGPT 帳號。
* **為什麼需要：** 允許您的 Pi 代表您執行相關操作，而無需儲存您真實的密碼。
1. 設定精靈將會產生一串安全的 URL（例如：`https://auth.openai.com/oauth/authorize?...`）。
2. 複製該 URL，然後在 PC 或手機上的網頁瀏覽器中開啟它。
3. 登入您的 ChatGPT 帳號，並點擊 **Authorize (授權)**。
4. 瀏覽器將被重新導向（轉跳）到一個包含 localhost（本機網址）的新頁面。
5. 將瀏覽器的網址複製並貼回設定精靈，然後按下 Enter 鍵。

#### 步驟 5：將 GPT-5.3 設定為主要模型
* **這是做什麼：** 指示 OpenClaw 的預設代理使用最新的旗艦模型。
* **為什麼需要：** 預設情況下，如果沒有明確指定，OpenClaw 可能會退而求其次使用較舊的模型。
1. 仍在設定精靈中，前往 **Agents (代理) -> Default Agent (預設代理) -> Primary Model (主要模型)**。
2. 從列表中選擇 `openai-codex/gpt-5.3`。
3. 退出並儲存設定精靈。

#### 步驟 6：驗證與啟動閘道器
* **這是做什麼：** 檢查系統是否存在錯誤，並啟動與雲端連線的 AI 代理。
* **為什麼需要：** 確保新的 OAuth 憑證與通訊路徑永久生效。
```bash
openclaw doctor --fix
openclaw gateway start
```

---

## 🏗️ 選項2（替代方案）：本地端部署 (Ollama + Qwen2.5:3B)

**最適合：** 要求最高隱私、需離線執行，以及不想支付任何持續性雲端訂閱費用的使用者。

### 逐步實作

#### 步驟 1：將 NVMe Swap (置換空間) 記憶體升級到 16GB
* **這是做什麼：** 用位於高速 NVMe SSD 上的 16GB 大型檔案，取代原本 8GB 的虛擬記憶體檔案（將儲存空間當作記憶體使用的一種技術）。
* **為什麼需要：** 載入本地端模型需要複製好幾 GB 龐大的資料到 RAM（記憶體）中。配置 16GB 的 swap 可以確保 Raspberry Pi 在這個過程中永遠不會發生記憶體耗盡 (OOM: Out-Of-Memory) 當機。
```bash
sudo swapoff -v /swapfile
sudo rm /swapfile
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```
檢查指令：
```bash
free -h
```

#### 步驟 2：建立硬體最佳化的 AI 模型 (`qwen-pi`)
* **重要注意事項：** 讓本地端 AI 模型執行代理式工作流程 (Agentic Workflow) 雖然「可行」，但不實用。硬體的限制會讓它變得很慢且徒勞無功。完全不建議這麼做。
* **這是做什麼：** 編譯一個特別為 Raspberry Pi 降速與限制的客製化 Qwen2.5:3B 版本。
* **為什麼需要：** 將 AI 限制使用 3 個 CPU 執行緒 (`num_thread`) 並限制其短期記憶空間 (`num_ctx 4096`)，我們才能防止 Pi 的 CPU 凍結當機，並確保網路任務不會逾時（等待時間過長）。
```bash
sudo systemctl enable ollama
sudo systemctl start ollama
ollama pull qwen2.5-coder:0.5b

cat <<EOF > Modelfile
FROM qwen2.5-coder:0.5b
PARAMETER num_thread 3
PARAMETER num_ctx 4096
EOF

ollama create qwen-pi -f Modelfile
```

檢查修改後的 qwen 模型名稱：
```bash
ollama list
```

#### 步驟 3：停止閘道器
* **這是做什麼：** 安全地關閉 OpenClaw 服務。
* **為什麼需要：** 防止在編輯時損壞設定檔。
```bash
openclaw gateway stop
```

#### 步驟 4：更新 OpenClaw 設定 (清晰的 Schema)
* **這是做什麼：** 使用經過嚴格驗證的 JSON (一種資料格式)，將 OpenClaw 指向您本地端的 Ollama 程式。
* **為什麼需要：** 該設定指示系統在何處尋找本地端模型，同時遵守 OpenClaw 嚴格的綱要 (Schema) 規則（例如不能有不合規定的 `timeout` 鍵值）。

開啟設定檔：
```bash
nano ~/.openclaw/openclaw.json
```
將模型和代理名稱替換為以下設定（*請將模型名稱修正為您實際在 Ollama 上執行的名稱）
```json
"models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://127.0.0.1:11434/v1",
        "apiKey": "ollama-local",
        "api": "openai-completions",
        "models":[
          { "id": "在此輸入您修改後的AI模型名稱", "name": "在此輸入您修改後的AI模型名稱" }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": { "primary": "ollama/在此輸入您修改後的AI模型名稱" },
      "models": { "ollama/qwen-pi": {} },
      "workspace": "/home/pi/.openclaw/workspace",
      "timeoutSeconds": 600
    }
  }
```
*（依序按下鍵盤的 `Ctrl+O`、`Enter`、`Ctrl+X` 來儲存並離開檔案）*。

#### 步驟 5：驗證與啟動
* **這是做什麼：** 執行 doctor (診斷工具) 來確保結構完美無誤，隨後啟動機器人。
* **為什麼需要：** 確保在背景執行前，系統處於穩定狀態。
```bash
openclaw doctor --fix
openclaw gateway start
```

#### 步驟 6：「預熱」技巧（對選項2至關重要）
* **這是做什麼：** 在您第一次傳送 Telegram 訊息給機器人*之前*，強迫 Ollama 提前將模型載入 16GB 的 swap（置換空間）中。
* **為什麼需要：** 因為我們無法更改 OpenClaw 寫死的 2 分鐘逾時限制（等待時間上限），因此如果等到接收您的第一條訊息時才冷啟動（從零開始載入）模型，Raspberry Pi 就會因超時而失敗。
在 Pi 的終端機（黑底控制台）中執行此指令，將模型無限期地預先載入記憶體中：
```bash
curl http://127.0.0.1:11434/api/generate -d '{"model": "qwen-pi", "keep_alive": -1}'
```
*（一旦這個步驟完成，您的 Telegram 機器人就能夠完美回應，再也不會發生逾時當機了！）*

---

## 📝 總結

現在，您擁有兩條功能完善的途徑：
1. **雲端方法：** 透過利用 OpenClaw 互動式的 `configure` (設定) 工具，經由 **OpenAI Codex (ChatGPT OAuth - 開放授權)** 完成認證。您可將請求直接導向現有的 ChatGPT Plus 訂閱與 GPT-5.3 服務，完全消除額外的 API 成本。
2. **本地端方法：** 藉由套用符合標準且清晰的 `openclaw.json` 設定、將 NVMe swap 擴充至 16GB，並在背景執行帶有 `keep_alive` 的 curl 指令來預先載入模型，您可以徹底避開那些常導致標準本地端作業崩潰的硬體「冷啟動」（啟動載入時耗時過久導致的逾時）問題。
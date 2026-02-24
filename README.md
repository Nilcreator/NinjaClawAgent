# Language Versions
* [English](#english)
* [日本語 (Japanese)](#japanese)
* [繁體中文 (Traditional Chinese)](#chinese)

<a id="english"></a>

## Autonomous AI Agent based on Raspberry Pi 5 and OpenClaw

This report provides a production-grade implementation plan for deploying OpenClaw (an autonomous AI agent framework) on a Raspberry Pi 5 (8GB RAM model). Unlike a standard chatbot, this system has the ability to execute tools and maintain long-term memory.
This architecture includes three key optimizations to overcome the hardware limitations of the Raspberry Pi 5:
1. **I/O Throughput (Data Transfer Speed):** We will migrate the operating system (OS) to an NVMe SSD (a much faster type of storage drive) to eliminate the speed bottleneck caused by traditional SD cards.
2. **Thermal Management (Cooling):** We will implement an aggressive active cooling curve (adjusting the fan to spin faster at lower temperatures) to ensure the CPU (the brain of the computer) maintains peak performance during heavy AI inference tasks without overheating.
3. **Memory Management:** We will establish 8GB+ of NVMe Swap memory (virtual memory that uses your storage drive as extra RAM). This prevents "Out of Memory" (OOM) crashes when the system tries to load large AI models.

For the software, this setup uses cloud model: Google Gemini 2.5 Flash, leveraging its advanced reasoning capabilities. (but you can also install Ollama to run local model as an alternative) It also uses Telegram (a messaging app) to provide a safe and secure remote operation interface.
 
Below is the complete step-by-step implementation.

***

# Edge AI Architecture Report: Autonomous Agent Implementation on Raspberry Pi 5

## 1. Executive Summary
This report provides a production-grade, step-by-step implementation plan for deploying Openclaw with Gemini 2.5 Flash on a Raspberry Pi 5 (8GB). 
Because our AI workload explicitly requires a massive 8GB swap file located on the high-speed NVMe drive, we will bypass legacy tools and use standard Linux swap creation. Furthermore, because OpenClaw depends on native audio libraries that must compile from source on Node 22/ARM64 architectures, this guide includes specific compiler flag optimizations to ensure a flawless build process.

This architecture achieves maximum efficiency through:
1. **I/O Throughput:** Booting directly from NVMe storage using rpi-clone.
2. **Memory Management:** Creating a raw 8GB standard Linux swap file on the NVMe.
3. **Thermal Stability:** Configuring an aggressive active cooling curve via `config.txt`.
4. **Native Compilation Bypass:** Adjusting GCC strictness to safely compile dependencies.

________________

## 2. Hardware Assembly & Thermal Configuration
**Objective:** Assemble the hardware and configure firmware-level cooling to prevent thermal throttling.

**2.1 Component Assembly**
* **Active Cooler:** Attach the official Raspberry Pi Active Cooler. Connect the fan to the 4-pin "FAN" header.
* **NVMe HAT & SSD:** Attach your NVMe HAT. Insert the Hudison 260G NVMe SSD and secure it.

**2.2 Operating System Installation (Temporary SD Boot)**
1. Use Raspberry Pi Imager on your PC. Select **Raspberry Pi OS Lite (64-bit)**.
2. Select your 32GB MicroSD card.
3. Set hostname to `ai-agent`, enable SSH, and configure your user/password.
4. Write the OS, insert the SD card, power on, and SSH into it:
   ```bash
   ssh pi@ai-agent.local
   ```
5. When the first time you logged in, we need to update the os and software by typing:
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   ```
   Then reboot
   ```bash
   sudo reboot
   ```

**2.3 Aggressive Thermal Management**
Lower the fan threshold to prevent heat soaking.
1. Open the boot configuration file:
   ```bash
   sudo nano /boot/firmware/config.txt
   ```
2. Paste the following lines at the very bottom:
   ```ini
   # Aggressive Cooling Curve for AI Workloads
   dtparam=fan_temp0=40000
   dtparam=fan_temp0_hyst=5000
   dtparam=fan_temp0_speed=75

   dtparam=fan_temp1=55000
   dtparam=fan_temp1_hyst=5000
   dtparam=fan_temp1_speed=150

   dtparam=fan_temp2=65000
   dtparam=fan_temp2_hyst=5000
   dtparam=fan_temp2_speed=255
   ```
3. Save (Ctrl+O, Enter) and Exit (Ctrl+X).

________________

## 3. Storage Migration: SD to NVMe
**Objective:** Copy the OS from the slow SD card to the fast NVMe SSD using `rpi-clone`.

**3.1 Install Git and Clone Tool**
```bash
sudo apt install git -y
git clone https://github.com/geerlingguy/rpi-clone.git
cd rpi-clone
sudo cp rpi-clone rpi-clone-setup /usr/local/sbin
```

**3.2 Execute the Clone**
1. Identify your NVMe drive (usually `nvme0n1`): `lsblk`
2. Run the clone command:
   ```bash
   sudo rpi-clone nvme0n1
   ```
   * Prompt: Initialize destination? Type **yes**
   * Prompt: Label partitions? Press **Enter** (leave blank)
   * Prompt: Unmount when finished? Type **yes**

**3.3 Configure Boot Order**
1. Open config: `sudo raspi-config`
2. Navigate to **6 Advanced Options** -> **A4 Boot Order** -> **B2 NVMe/USB Boot**.
3. Select **Yes**, then **Finish**.
4. Shut down the Pi: `sudo shutdown -h now`
5. **Crucial Step:** Physically remove the MicroSD card and turn the power back on. 

**3.4 Enable PCIe Gen 3 (Post-Boot Optimization)**
1. `sudo nano /boot/firmware/config.txt`
2. Add these two lines at the bottom:
   ```ini
   dtparam=pciex1
   dtparam=pciex1_gen=3
   ```
3. Reboot: `sudo reboot`

________________

## 4. System Optimization: Standard Linux 8GB Swap
**Objective:** Create virtual memory to prevent crashes when loading the AI model.

1. Create an 8GB empty file on the NVMe:
   ```bash
   sudo fallocate -l 8G /swapfile
   ```
2. Set secure permissions:
   ```bash
   sudo chmod 600 /swapfile
   ```
3. Format and activate swap:
   ```bash
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```
4. Make it permanent:
   ```bash
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   ```
5. Verify swap is active (should show roughly 8.0Gi): `free -h`

________________

## 5. Inference Engine: Ollama Setup (optional)
**Objective:** Though we will using cloude model of Gemini 2.5 Flash. Install ollama can allow you to run your ai agent with the local models in the future.

**5.1 Install Ollama**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**5.2 Pull AI Models**
```bash
ollama pull qwen2.5:3b
```

________________

## 6. OpenClaw Implementation & Configuration
**Objective:** Install OpenClaw, bypassing native C compilation errors, and connect to Telegram.

**6.1 Install Node.js 22**
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
nvm alias default 22
```

**6.2 Install Build Tools & OpenClaw (CRITICAL COMPILATION FIX)**
Because OpenClaw relies on native audio processing that fails to compile on modern Pi systems, we must explicitly inject C-Compiler flags.

1. Ensure OS-level build dependencies exist:
   ```bash
   sudo apt-get update
   sudo apt-get install -y build-essential python3 make gcc g++ libopus-dev
   ```
2. **Inject Compiler Flags & Install:** Set the flags to force the compiler to ignore the legacy Opus ARM-NEON syntax errors, then install OpenClaw globally.
   ```bash
   export CFLAGS="-Wno-error=implicit-function-declaration -Wno-implicit-function-declaration"
   export CXXFLAGS="-Wno-error=implicit-function-declaration -Wno-implicit-function-declaration"
   
   npm install -g openclaw@latest
   ```
   *(Note: This install may take a few minutes as `node-gyp` will now successfully compile `@discordjs/opus` in the background).*

**6.3 Manual Onboarding Bypass**
Before running openclaw onboarding wizard, we need to manually create the correct folders and configuration file (Agent Config)
```bash
mkdir -p ~/.openclaw/agents/main/agent
```
We directly configure Gemini (2.5 flash), assign you as the administrator, and add your telegram ID.
```bash
cat > ~/.openclaw/agents/main/agent/openclaw.json <<EOF
{
  "id": "main",
  "name": "Clawbot",
  "description": "Gemini Powered Assistant",
  "model": "gemini-2.5-flash",
  "systemPrompt": "You are a helpful assistant running on a Raspberry Pi. Be concise.",
  "llm": {
    "provider": "google",
    "apiKey": "YOUR_API_KEY_HERE",
    "model": "gemini-2.5-flash"
  },
  "embedding": {
    "provider": "google",
    "apiKey": "YOUR_API_KEY_HERE",
    "model": "text-embedding-004"
  },
  "admins": ["YOUR_TELEGRAM_ID"],
  "allowedUsers": ["YOUR_TELEGRAM_ID"]
}
EOF
```
Then we manually set up the Global Configuration File (System config)
```bash
cat > ~/.openclaw/openclaw.json <<EOF
{
  "gateway": {
    "mode": "local",
    "port": 18789
  }
}
EOF
```

Now we can run onboard wizrd using the daemon parameter

```bash
openclaw onboard --install-daemon
```
* Security Warning: Select **Yes**.
* Onboarding Mode: Select **Manual Mode**.
* Gateway Config & Workspace: Press **Enter** to accept defaults.
* Model/Auth Provider & Clients/Channels: Select **Google Gemini and Telegram**.
* Finish: Let it complete. **You can add Skills to your bot or just skip for now**
* Select Node as the gateway runtime and install gateway
* Lastly, hatch your bot in TUI (Terminal User Interface)
If you see gateway connected, your JAVIS is ready now

**6.4 Secure Paring**
By default, OpenClaw will not talk to strangers. You must pair your Telegram messaging account with it.
1. Open your bot inside the Telegram app and click Start.
2. The bot will reply to you with a Pairing Code (For example: A1B2-C3D4).
3. Open a new terminal window, SSH in, and run:
   ```bash
   openclaw pairing approve telegram <Your_Pairing_Code_Here>
   ```
4. Go back to Telegram and type "Hi" to start chatting with your brand new AI agent!
5. **Done!** You now have a fully autonomous, local AI assistant safely run on Raspberry Pi 5.
<br>
<hr>
<br>

<a id="japanese"></a>
## [日本語] Raspberry Pi 5とOpenClawに基づく自律型AIエージェント

このレポートは、Raspberry Pi 5（8GB RAMモデル）にOpenClaw（自律型AIエージェントフレームワーク）を展開するための、本番環境レベルの実装計画を提供します。標準的なチャットボットとは異なり、このシステムはツールを実行し、長期記憶を維持する能力を備えています。
このアーキテクチャには、Raspberry Pi 5のハードウェアの制限を克服するための3つの重要な最適化が含まれています：
1. **I/Oスループット（データ転送速度）：** 従来のSDカードによる速度のボトルネックを解消するため、オペレーティングシステム（OS）をNVMe SSD（より高速なストレージドライブ）に移行します。
2. **熱管理（冷却）：** 積極的なアクティブ冷却カーブ（より低い温度でファンを速く回転させるよう調整）を実装し、AI推論などの重いタスク中もCPUが過熱せずに最高性能を維持できるようにします。
3. **メモリ管理：** 8GB以上のNVMeスワップメモリ（ストレージドライブを余分なRAMとして使用する仮想メモリ）を構築します。これにより、システムが大規模なAIモデルを読み込む際の「メモリ不足（OOM）」によるクラッシュを防ぎます。

ソフトウェアに関して、このセットアップでは高度な推論機能を活用するためにクラウドモデルであるGoogle Gemini 2.5 Flashを使用します。（ただし、代替としてOllamaをインストールし、ローカルモデルを実行することも可能です）。さらに、安全でセキュアな遠隔操作インターフェースを提供するためにTelegram（メッセージングアプリ）を使用します。

以下は、完全なステップバイステップの実装手順です。

***

# エッジAIアーキテクチャレポート：Raspberry Pi 5での自律型エージェントの実装

## 1. 概要
このレポートは、Raspberry Pi 5（8GB）でGemini 2.5 Flashを搭載したOpenclawを展開するための、本番環境レベルのステップバイステップの実装計画を提供します。
当社のAIワークロードは、高速なNVMeドライブ上に配置された巨大な8GBのスワップファイルを明確に必要とするため、レガシーツールをバイパスし、標準のLinuxスワップ作成を使用します。さらに、OpenClawはNode 22/ARM64アーキテクチャでソースからコンパイルする必要があるネイティブのオーディオライブラリに依存しているため、このガイドには、ビルドプロセスを確実に成功させるための特定のコンパイラフラグの最適化が含まれています。

このアーキテクチャは、以下を通じて最大の効率を達成します：
1. **I/Oスループット：** rpi-cloneを使用してNVMeストレージから直接起動。
2. **メモリ管理：** NVMe上に生の8GBの標準Linuxスワップファイルを作成。
3. **熱安定性：** `config.txt`を介した積極的なアクティブ冷却カーブの設定。
4. **ネイティブコンパイルのバイパス：** 依存関係を安全にコンパイルするためのGCCの厳格さの調整。

________________

## 2. ハードウェアの組み立てと熱設定
**目的：** ハードウェアを組み立て、サーマルスロットリングを防ぐためにファームウェアレベルの冷却を設定します。

**2.1 コンポーネントの組み立て**
* **アクティブクーラー：** 公式のRaspberry Pi Active Coolerを取り付けます。ファンを4ピンの「FAN」ヘッダーに接続します。
* **NVMe HATとSSD：** NVMe HATを取り付けます。Hudison 260G NVMe SSDを挿入して固定します。

**2.2 オペレーティングシステムのインストール（一時的なSDブート）**
1. PCでRaspberry Pi Imagerを使用します。**Raspberry Pi OS Lite (64-bit)** を選択します。
2. 32GBのMicroSDカードを選択します。
3. ホスト名を `ai-agent` に設定し、SSHを有効にして、ユーザー/パスワードを設定します。
4. OSを書き込み、SDカードを挿入して電源を入れ、SSHで接続します：
   ```bash
   ssh pi@ai-agent.local
   ```
5. 初回ログイン時に、以下のコマンドを入力してOSとソフトウェアを更新する必要があります：
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   ```
   その後、再起動します：
   ```bash
   sudo reboot
   ```

**2.3 積極的な熱管理**
熱の蓄積を防ぐためにファンのしきい値を下げます。
1. ブート設定ファイルを開きます：
   ```bash
   sudo nano /boot/firmware/config.txt
   ```
2. 一番下に以下の行を貼り付けます：
   ```ini
   # AIワークロードのための積極的な冷却カーブ
   dtparam=fan_temp0=40000
   dtparam=fan_temp0_hyst=5000
   dtparam=fan_temp0_speed=75

   dtparam=fan_temp1=55000
   dtparam=fan_temp1_hyst=5000
   dtparam=fan_temp1_speed=150

   dtparam=fan_temp2=65000
   dtparam=fan_temp2_hyst=5000
   dtparam=fan_temp2_speed=255
   ```
3. 保存（Ctrl+O、Enter）して終了（Ctrl+X）します。

________________

## 3. ストレージの移行：SDからNVMeへ
**目的：** `rpi-clone`を使用して、低速なSDカードから高速なNVMe SSDへOSをコピーします。

**3.1 GitとCloneツールのインストール**
```bash
sudo apt install git -y
git clone https://github.com/geerlingguy/rpi-clone.git
cd rpi-clone
sudo cp rpi-clone rpi-clone-setup /usr/local/sbin
```

**3.2 クローンの実行**
1. NVMeドライブを確認（通常は `nvme0n1`）：`lsblk`
2. クローンコマンドを実行：
   ```bash
   sudo rpi-clone nvme0n1
   ```
   * プロンプト: Initialize destination? **yes** と入力
   * プロンプト: Label partitions? **Enter** を押す（空白のまま）
   * プロンプト: Unmount when finished? **yes** と入力

**3.3 起動順序の設定**
1. 設定を開く：`sudo raspi-config`
2. **6 Advanced Options** -> **A4 Boot Order** -> **B2 NVMe/USB Boot** へ移動。
3. **Yes** を選択し、**Finish** を選択。
4. Piをシャットダウン：`sudo shutdown -h now`
5. **重要な手順：** MicroSDカードを物理的に取り外し、再度電源を入れます。

**3.4 PCIe Gen 3の有効化（起動後の最適化）**
1. `sudo nano /boot/firmware/config.txt`
2. 一番下に以下の2行を追加します：
   ```ini
   dtparam=pciex1
   dtparam=pciex1_gen=3
   ```
3. 再起動：`sudo reboot`

________________

## 4. システムの最適化：標準Linux 8GBスワップ
**目的：** AIモデル読み込み時のクラッシュを防ぐために仮想メモリを作成します。

1. NVMe上に8GBの空ファイルを作成：
   ```bash
   sudo fallocate -l 8G /swapfile
   ```
2. セキュアなパーミッションを設定：
   ```bash
   sudo chmod 600 /swapfile
   ```
3. スワップをフォーマットして有効化：
   ```bash
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```
4. 永続化する：
   ```bash
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   ```
5. スワップがアクティブであることを確認（約8.0Giと表示されるはず）：`free -h`

________________

## 5. 推論エンジン：Ollamaのセットアップ（オプション）
**目的：** Gemini 2.5 Flashのクラウドモデルを使用しますが、Ollamaをインストールしておくと、将来的にローカルモデルでAIエージェントを実行できるようになります。

**5.1 Ollamaのインストール**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**5.2 AIモデルの取得**
```bash
ollama pull qwen2.5:3b
```

________________

## 6. OpenClawの実装と設定
**目的：** ネイティブCコンパイルエラーを回避してOpenClawをインストールし、Telegramに接続します。

**6.1 Node.js 22のインストール**
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
nvm alias default 22
```

**6.2 ビルドツールとOpenClawのインストール（重要なコンパイル修正）**
OpenClawは最新のPiシステムでのコンパイルに失敗するネイティブな音声処理に依存しているため、C-Compilerフラグを明示的に注入する必要があります。

1. OSレベルのビルド依存関係が存在することを確認：
   ```bash
   sudo apt-get update
   sudo apt-get install -y build-essential python3 make gcc g++ libopus-dev
   ```
2. **コンパイラフラグの注入とインストール：** コンパイラにレガシーなOpus ARM-NEONの構文エラーを強制的に無視させるようフラグを設定し、OpenClawをグローバルにインストールします。
   ```bash
   export CFLAGS="-Wno-error=implicit-function-declaration -Wno-implicit-function-declaration"
   export CXXFLAGS="-Wno-error=implicit-function-declaration -Wno-implicit-function-declaration"
   
   npm install -g openclaw@latest
   ```
   *(注: バックグラウンドで `node-gyp` が `@discordjs/opus` を正常にコンパイルするため、このインストールには数分かかる場合があります)。*

**6.3 手動オンボーディングのセットアップ**
openclawのオンボーディングウィザードを実行する前に、適切なフォルダと設定ファイル（エージェント設定）を手動で作成する必要があります。
```bash
mkdir -p ~/.openclaw/agents/main/agent
```
Gemini (2.5 flash) を直接設定し、自分を管理者として割り当て、Telegram IDを追加します。
```bash
cat > ~/.openclaw/agents/main/agent/openclaw.json <<EOF
{
  "id": "main",
  "name": "Clawbot",
  "description": "Gemini Powered Assistant",
  "model": "gemini-2.5-flash",
  "systemPrompt": "You are a helpful assistant running on a Raspberry Pi. Be concise.",
  "llm": {
    "provider": "google",
    "apiKey": "YOUR_API_KEY_HERE",
    "model": "gemini-2.5-flash"
  },
  "embedding": {
    "provider": "google",
    "apiKey": "YOUR_API_KEY_HERE",
    "model": "text-embedding-004"
  },
  "admins": ["YOUR_TELEGRAM_ID"],
  "allowedUsers": ["YOUR_TELEGRAM_ID"]
}
EOF
```
次に、グローバル設定ファイル（システム設定）を手動で設定します。
```bash
cat > ~/.openclaw/openclaw.json <<EOF
{
  "gateway": {
    "mode": "local",
    "port": 18789
  }
}
EOF
```

これで、daemonパラメータを使用してオンボーディングウィザードを実行できます。

```bash
openclaw onboard --install-daemon
```
* Security Warning: **Yes** を選択します。
* Onboarding Mode: **Manual Mode** を選択します。
* Gateway Config & Workspace: **Enter** を押してデフォルトを受け入れます。
* Model/Auth Provider & Clients/Channels: **Google Gemini and Telegram** を選択します。
* Finish: 完了するのを待ちます。**ボットにSkillsを追加することも、今はスキップすることもできます。**
* ゲートウェイのランタイムとしてNodeを選択し、ゲートウェイをインストールします。
* 最後に、TUI（ターミナルユーザーインターフェース）でボットを起動します。
ゲートウェイが接続されたと表示されれば、JAVISの準備は完了です。

**6.4 安全なペアリング**
デフォルトでは、OpenClawは見知らぬ人と会話しません。Telegramのメッセージングアカウントをペアリングする必要があります。
1. Telegramアプリ内でボットを開き、Startをクリックします。
2. ボットがペアリングコード（例：A1B2-C3D4）を返信してきます。
3. 新しいターミナルウィンドウを開き、SSHでログインして以下を実行します：
   ```bash
   openclaw pairing approve telegram <Your_Pairing_Code_Here>
   ```
4. Telegramに戻り、「Hi」と入力して、新しいAIエージェントとのチャットを始めましょう！
5. **完了！** これで、Raspberry Pi 5上で安全に動作する、完全自律型のローカルAIアシスタントの完成です。


<br>
<hr>
<br>

<a id="chinese"></a>
## [繁體中文] 基於 Raspberry Pi 5 和 OpenClaw 的自主 AI 代理

本報告提供了一份生產級別的實施計畫，用於在 Raspberry Pi 5（8GB RAM 版本）上部署 OpenClaw（一個自主 AI 代理框架）。與標準的聊天機器人不同，該系統具備執行工具並保持長期記憶的能力。
此架構包含三項關鍵優化，以克服 Raspberry Pi 5 的硬體限制：
1. **I/O 吞吐量（資料傳輸速度）：** 我們會將作業系統（OS）遷移到 NVMe SSD（一種更快速的儲存硬碟）上，以消除傳統 SD 卡帶來的速度瓶頸。
2. **熱管理（散熱）：** 我們將實施進取的散熱曲線（在較低溫度時讓風扇轉得更快），以確保 CPU（電腦的大腦）在執行繁重的 AI 推理任務時保持最佳性能而不會過熱。
3. **記憶體管理：** 我們將建立 8GB 以上的 NVMe Swap 記憶體（將儲存硬碟作為額外 RAM 使用的虛擬記憶體）。這可防止系統在嘗試載入大型 AI 模型時發生「記憶體不足」（OOM）崩潰。

在軟體方面，此設置使用雲端模型：Google Gemini 2.5 Flash，以利用其先進的推理能力。（不過您也可以安裝 Ollama 執行本地模型作為替代方案）。同時，它也使用 Telegram（通訊軟體）來提供一個安全可靠的遠端操作介面。

以下是完整的逐步實施指引。

***

# 邊緣 AI 架構報告：在 Raspberry Pi 5 上的自主代理實施

## 1. 執行摘要
本報告提供在搭載 Gemini 2.5 Flash 的 Raspberry Pi 5 (8GB) 上部署 Openclaw 的生產級逐步實施計畫。
因為我們的 AI 工作負載明確需要放置在高速 NVMe 硬碟上的巨大 8GB 交換檔 (swap file)，我們將略過傳統工具，並使用標準的 Linux 交換空間建立方式。此外，由於 OpenClaw 依賴需要在 Node 22/ARM64 架構上從原始碼編譯的原生音訊庫，本指南包含了特定的編譯器標誌優化，以確保無縫的建置過程。

此架構透過以下方式實現最高效率：
1. **I/O 吞吐量：** 使用 rpi-clone 直接從 NVMe 儲存空間開機。
2. **記憶體管理：** 在 NVMe 上建立一個原生的 8GB 標準 Linux 交換檔。
3. **熱穩定性：** 透過 `config.txt` 設定進取的散熱風扇曲線。
4. **原生編譯繞過：** 調整 GCC 嚴格度以安全地編譯依賴項目。

________________

## 2. 硬體組裝與散熱設定
**目標：** 組裝硬體並設定韌體層級的散熱以防止過熱降頻。

**2.1 零組件組裝**
* **主動式散熱器：** 安裝官方的 Raspberry Pi Active Cooler。將風扇連接到 4 針 "FAN" 接口。
* **NVMe HAT 與 SSD：** 安裝您的 NVMe HAT。插入 Hudison 260G NVMe SSD 並將其固定。

**2.2 作業系統安裝（暫時使用 SD 卡開機）**
1. 在您的 PC 上使用 Raspberry Pi Imager。選擇 **Raspberry Pi OS Lite (64-bit)**。
2. 選擇您的 32GB MicroSD 卡。
3. 將主機名稱設定為 `ai-agent`，啟用 SSH，並設定您的使用者名稱與密碼。
4. 寫入 OS，插入 SD 卡，接上電源，並透過 SSH 登入：
   ```bash
   ssh pi@ai-agent.local
   ```
5. 首次登入時，我們需要輸入以下指令更新作業系統與軟體：
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   ```
   然後重新開機：
   ```bash
   sudo reboot
   ```

**2.3 進取的熱管理**
降低風扇轉速門檻以防止熱量累積。
1. 開啟開機設定檔：
   ```bash
   sudo nano /boot/firmware/config.txt
   ```
2. 在最底部貼上以下幾行：
   ```ini
   # 針對 AI 工作負載的進取散熱曲線
   dtparam=fan_temp0=40000
   dtparam=fan_temp0_hyst=5000
   dtparam=fan_temp0_speed=75

   dtparam=fan_temp1=55000
   dtparam=fan_temp1_hyst=5000
   dtparam=fan_temp1_speed=150

   dtparam=fan_temp2=65000
   dtparam=fan_temp2_hyst=5000
   dtparam=fan_temp2_speed=255
   ```
3. 儲存 (Ctrl+O, Enter) 並離開 (Ctrl+X)。

________________

## 3. 儲存遷移：從 SD 卡到 NVMe
**目標：** 使用 `rpi-clone` 將作業系統從慢速的 SD 卡複製到快速的 NVMe SSD 上。

**3.1 安裝 Git 與 Clone 工具**
```bash
sudo apt install git -y
git clone https://github.com/geerlingguy/rpi-clone.git
cd rpi-clone
sudo cp rpi-clone rpi-clone-setup /usr/local/sbin
```

**3.2 執行拷貝**
1. 確認您的 NVMe 硬碟（通常為 `nvme0n1`）：`lsblk`
2. 執行 clone 指令：
   ```bash
   sudo rpi-clone nvme0n1
   ```
   * 提示: Initialize destination? 輸入 **yes**
   * 提示: Label partitions? 按 **Enter** (留白)
   * 提示: Unmount when finished? 輸入 **yes**

**3.3 設定開機順序**
1. 開啟設定：`sudo raspi-config`
2. 導覽至 **6 Advanced Options** -> **A4 Boot Order** -> **B2 NVMe/USB Boot**。
3. 選擇 **Yes**，然後選 **Finish**。
4. 將 Pi 關機：`sudo shutdown -h now`
5. **關鍵步驟：** 實體移除 MicroSD 卡並重新開啟電源。

**3.4 啟用 PCIe Gen 3 (開機後最佳化)**
1. `sudo nano /boot/firmware/config.txt`
2. 在底部加入這兩行：
   ```ini
   dtparam=pciex1
   dtparam=pciex1_gen=3
   ```
3. 重新開機：`sudo reboot`

________________

## 4. 系統優化：標準 Linux 8GB Swap
**目標：** 建立虛擬記憶體以防止載入 AI 模型時發生崩潰。

1. 在 NVMe 上建立一個 8GB 的空白檔案：
   ```bash
   sudo fallocate -l 8G /swapfile
   ```
2. 設定安全的權限：
   ```bash
   sudo chmod 600 /swapfile
   ```
3. 格式化並啟用 swap：
   ```bash
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```
4. 使其永久生效：
   ```bash
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   ```
5. 驗證 swap 是否處於啟用狀態（應該會顯示大約 8.0Gi）：`free -h`

________________

## 5. 推理引擎：Ollama 設置 (選擇性)
**目標：** 儘管我們將使用 Google Gemini 2.5 Flash 雲端模型。安裝 Ollama 可以讓您在未來使用本地模型來運行您的 AI 代理。

**5.1 安裝 Ollama**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**5.2 下載 AI 模型**
```bash
ollama pull qwen2.5:3b
```

________________

## 6. OpenClaw 實施與配置
**目標：** 安裝 OpenClaw，繞過原生 C 編譯錯誤，並連接至 Telegram。

**6.1 安裝 Node.js 22**
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
nvm alias default 22
```

**6.2 安裝建置工具與 OpenClaw (關鍵的編譯修正)**
因為 OpenClaw 依賴於在現代 Pi 系統上會編譯失敗的原生音訊處理套件，我們必須明確地注入 C-Compiler 標誌。

1. 確保作業系統層級的建置依賴項目存在：
   ```bash
   sudo apt-get update
   sudo apt-get install -y build-essential python3 make gcc g++ libopus-dev
   ```
2. **注入編譯器標誌並安裝：** 設定標誌以強制編譯器忽略舊版的 Opus ARM-NEON 語法錯誤，然後全域安裝 OpenClaw。
   ```bash
   export CFLAGS="-Wno-error=implicit-function-declaration -Wno-implicit-function-declaration"
   export CXXFLAGS="-Wno-error=implicit-function-declaration -Wno-implicit-function-declaration"
   
   npm install -g openclaw@latest
   ```
   *(註：此安裝可能會花費幾分鐘，因為 `node-gyp` 現在會在背景成功編譯 `@discordjs/opus`)。*

**6.3 手動快速設定繞過**
在執行 openclaw 引導配置精靈之前，我們需要手動建立正確的資料夾與設定檔 (代理設定檔)。
```bash
mkdir -p ~/.openclaw/agents/main/agent
```
我們直接設定 Gemini (2.5 flash)，將您指派為管理員，並加入您的 Telegram ID。
```bash
cat > ~/.openclaw/agents/main/agent/openclaw.json <<EOF
{
  "id": "main",
  "name": "Clawbot",
  "description": "Gemini Powered Assistant",
  "model": "gemini-2.5-flash",
  "systemPrompt": "You are a helpful assistant running on a Raspberry Pi. Be concise.",
  "llm": {
    "provider": "google",
    "apiKey": "YOUR_API_KEY_HERE",
    "model": "gemini-2.5-flash"
  },
  "embedding": {
    "provider": "google",
    "apiKey": "YOUR_API_KEY_HERE",
    "model": "text-embedding-004"
  },
  "admins": ["YOUR_TELEGRAM_ID"],
  "allowedUsers": ["YOUR_TELEGRAM_ID"]
}
EOF
```
然後我們手動設定全域的系統設定檔 (System config)
```bash
cat > ~/.openclaw/openclaw.json <<EOF
{
  "gateway": {
    "mode": "local",
    "port": 18789
  }
}
EOF
```

現在我們可以使用 daemon 參數來執行引導精靈

```bash
openclaw onboard --install-daemon
```
* Security Warning: 選擇 **Yes**。
* Onboarding Mode: 選擇 **Manual Mode**。
* Gateway Config & Workspace: 按 **Enter** 接受預設值。
* Model/Auth Provider & Clients/Channels: 選擇 **Google Gemini and Telegram**。
* Finish: 讓它完成。**您可以為您的機器人添加 Skills，或者現在先略過**
* 選擇 Node 作為 gateway 運行時並安裝 gateway
* 最後，在 TUI (Terminal User Interface) 中孵化您的機器人
如果您看到 gateway connected，您的 JAVIS 現在已經準備就緒

**6.4 安全配對**
預設情況下，OpenClaw 不會與陌生人交談。您必須將您的 Telegram 帳號與其配對。
1. 在 Telegram 應用程式內打開您的機器人並點擊 Start。
2. 機器人會回覆您一組 Pairing Code（例如：A1B2-C3D4）。
3. 打開一個新的終端機視窗，透過 SSH 登入，並執行：
   ```bash
   openclaw pairing approve telegram <Your_Pairing_Code_Here>
   ```
4. 回到 Telegram 並輸入 "Hi" 開始與您全新的 AI 代理聊天！
5. **完成！** 您現在擁有一個安全運行在 Raspberry Pi 5 上的完全自主的本地 AI 助理了。
# NinjaClawAgent

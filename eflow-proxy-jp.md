# EFLOW - Proxy 環境 構築手順

> 最終更新日: 2022/07/15  
> 対象環境: Windows 10 21H2 / EFLOW CR 1.2

## 前提:

この手順では、以下のような環境において EFLOW 環境の構築を行うことを想定しています。  
![EFLOW 環境](./img/eflow-env.png 'EFLOW 環境')

* Windows 10 PC は Proxy 経由のみ インターネットに接続可
* Windows 10 PC は 1つ以上の LAN または Wi-Fi 接続を持つ
* Windows 10 PC と Proxy は HTTP で接続
* 作業用 PC は直接、または Proxy 経由でインターネットに接続可

## 準備:

1. Azure ポータル の Azure IoT Hub で IoT Edge デバイスの作成  
    IoT Edge デバイスの プライマリ接続文字列 をコピーします  
![Azure IoT Hub - IoT Edge](./img/eflow-iotedge-connection-string.png 'Azure IoT Hub - IoT Edge')  
1. Windows 10 November 2021 Update (21H2) のインストール  
    https://www.microsoft.com/ja-jp/software-download/windows10/

    * この段階で Hyper-V を有効にしておきます  
![Windows - Hyper-V](./img/eflow-win10-hyper-v.png 'Windows - Hyper-V')
    * Windows 10 PC 自体にも Proxy 設定を行います  
![Windows - Proxy](./img/eflow-win10-proxy.png 'Windows - Proxy')

    作業PC または Windows 10 PC に動作確認用の Visual Studio Code / Storage Explorer をインストールします  
    https://code.visualstudio.com/  
    https://azure.microsoft.com/ja-jp/features/storage-explorer/  
    Visual Studio Code は 拡張機能: Azure IoT Tools をインストールします  

1. EFLOW のインストール  
    以下のコマンドを実行します  
    ```Powershell
    Set-ExecutionPolicy -ExecutionPolicy AllSigned -Force
    ```
    ```Powershell
    $msiPath = $([io.Path]::Combine($env:TEMP, 'AzureIoTEdge.msi'))
    $ProgressPreference = 'SilentlyContinue'
    Invoke-WebRequest "https://aka.ms/AzEFLOWMSI-CR-X64" -OutFile $msiPath
    ```
    ```Powershell
    Start-Process -Wait msiexec -ArgumentList "/i","$([io.Path]::Combine($env:TEMP, 'AzureIoTEdge.msi'))","/qn"
    ```

    > 詳細は以下ページに記載があります  
    > https://docs.microsoft.com/ja-jp/azure/iot-edge/how-to-provision-single-device-linux-on-windows-symmetric?view=iotedge-2020-11&tabs=azure-portal%2Cpowershell


1. 仮想スイッチの作成  
    Hyper-V マネージャー で **仮想スイッチ - 外部** を作成します  
    > 複数のネットワークアダプターがある場合はネットワークに接続されている方を選択します      
![Windows - Hyper-V - vSwitch](./img/eflow-win10-hyper-v-vswitch.png 'Windows - Hyper-V - vSwitch')  

1. EFLOW VM の作成  
    > 仮想スイッチの名前、IPv4 関連の項目は環境に合わせて変更ください  
    ```Powershell
    Deploy-Eflow -acceptEula Yes -acceptOptionalTelemetry No -cpuCount 2 -memoryInMB 4096 -vswitchName ExtEFLOW -vswitchType External -ip4Address 192.168.8.222 -ip4PrefixLength 24 -ip4GatewayAddress 192.168.8.1
    ```
1. EFLOW VM に接続  
    ```Powershell
    Connect-EflowVm
    ```
    以下のコマンドを実行し、Proxy 経由でインターネット接続が可能なことを確認します  
     > Proxy の設定は環境に合わせて変更ください  
    ```
    curl -L -x http://192.168.8.199:3128 -o eflow-proxy.zip https://github.com/tahirai-microsoft/eflow-ploxy-jp/zipball/master
    ```

## 手順:

1. Docker に Proxy を設定  
    以下のコマンドを実行、記述を行います
    ```
    sudo systemctl edit docker.service
    ```
     > Proxy の設定は環境に合わせて変更ください  
     > HTTP / HTTPS 双方の設定を推奨します  
    ```
    [Service]
    Environment="HTTPS_PROXY=http://192.168.8.199:3128/"
    Environment="HTTP_PROXY=http://192.168.8.199:3128/"
    ```
    ```
    sudo systemctl edit aziot-edged
    ```
     > Proxy の設定は環境に合わせて変更ください  
     > HTTPS のみ設定します  
    ```
    [Service]
    Environment=https_proxy=http://192.168.8.199:3128/
    ```
    ```
    sudo systemctl edit aziot-identityd
    ```
     > Proxy の設定は環境に合わせて変更ください  
     > HTTPS のみ設定します  
    ```
    [Service]
    Environment=https_proxy=http://192.168.8.199:3128/
    ```
    設定を反映します  
    ```
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```
    利用する コンテナーを 事前に Pull します
    ```
    sudo docker pull mcr.microsoft.com/azureiotedge-agent:1.2
    ```
    ```
    sudo docker pull mcr.microsoft.com/azureiotedge-hub:1.2
    ```
    ```  
    sudo docker pull mcr.microsoft.com/azureiotedge-simulated-temperature-sensor:1.0
    ```
    ```  
    sudo docker pull mcr.microsoft.com/azure-blob-storage:latest
    ```
1. IoT Edge を設定 (Proxy 設定を含む)  
    テキストエディタ (以下では nano) で設定ファイルに記述を追加します  
    ```
    sudo nano /etc/aziot/config.toml
    ```
     > provisioning は コメントアウト(#) を削除、agent はファイルの最後に追記します  
     > Proxy の設定は環境に合わせて変更ください (HTTPSのみ)  
     > コンテナーを事前に Pull するため、PullPolicy を Never としています  
    ```Toml
    [provisioning]
    source = "manual"
    connection_string = "(準備: で取得したプライマリ接続文字列)"
    ```
    ```Toml
    [agent]
    name = "edgeAgent"
    type = "docker"
    imagePullPolicy = "never"

    [agent.config]
    image = "mcr.microsoft.com/azureiotedge-agent:1.2"

    [agent.env]
    "UpstreamProtocol" = "AmqpWs"
    "https_proxy" = "http://192.168.8.199:3128"
    ```
    設定を反映します  
    ```
    sudo iotedge config apply
    ```
    ログ、及び モジュールリストを表示し、動作を確認します
    ```
    sudo iotedge system logs -- -f
    ```
    ```
    sudo iotedge list
    ```
1. Azure ポータルで deployment を設定 (simulated temperature sensor のデプロイ)  
    > それぞれ イメージのプルポリシー は なし に設定します (pull 済みのため)  
    > Edge ハブ には Proxy の設定を追加します

    ![Portal IoTEdge 1](./img/portal-iotedge-1.png 'Portal IoTEdge 1')  
    ![Portal IoTEdge 2](./img/portal-iotedge-2.png 'Portal IoTEdge 2')  
    ![Portal IoTEdge 3](./img/portal-iotedge-3.png 'Portal IoTEdge 3')  

    ログ、及び モジュールリストを表示し、動作を確認します
    ```
    sudo iotedge system logs -- -f
    ```
    ```
    sudo iotedge list
    ```
    Azure IoT Hub が IoT Edge デバイスからテレメトリを受け取れていることを確認します  
    ![VSCode Monitor](./img/vscode-monitor.png 'VSCode Monitor')  

1. Azure ポータルで deployment を設定 (azure blob storage のデプロイ)  
    > Proxy の設定を追加します  

    ![Portal Blob 1](./img/portal-iotedge-blob-1.png 'Portal Blob 1') 

    > 詳細は以下ページに記載があります  
    > https://docs.microsoft.com/ja-jp/azure/iot-edge/how-to-deploy-blob?view=iotedge-2020-11

    ログ、及び モジュールリストを表示し、動作を確認します
    ```
    sudo iotedge system logs -- -f
    ```
    ```
    sudo iotedge list
    ```
    Storage Explorer で Azure Blob Storage モジュールにアクセス出来ることを確認します  

    ![Storage Explorer](./img/storage-explorer.png 'Storage Explorer')  

    



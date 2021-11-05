
- 目的：節省安裝(多個) APK 到設備上的時間。

- 事前準備：
	1. 確認能使用 **adb** 指令
	2. 確認已連接 1 台安卓設備
	3. 放打包的目錄下(output/outputHB)
	4. 執行腳本批量安裝，在終端機輸入 `sh install_apks.sh` 或 `./install_apks.sh` 執行，即可以安裝[版本目錄]下所有的 apk

    - e.g. 目錄參考
        - `nativeapp/android/[version]/[apk_id]/[apk_id].apk`
        - 以下區分豎版與橫版

        ```
        output
        ├── install_apks.sh
        └── nativeapp
            └── android
                  └── 6.2.0
                      └── test1001
                          └── test1001.apk

        outputHB
        ├── install_apks.sh
        └── nativeapp
            └── android
                └── 1.1.0
                    ├── test1001hb
                    │   └── test001hb.apk
                    └── test2002hb
                        └── test2002hb.apk
        ```

- install_apks.sh
    ```bash
    #!/bin/bash

    # note:
    # $() 與 `` 作為命令替換使用。命令替換與變量替換差不多，都是用來重組命令行的，先完成引號裡的命令行，然後將其結果替換出來，再重組成新的命令行。

    # 取版本資料夾名稱(6.2.0 或 1.1.0)
    versionDir=`ls ./nativeapp/android`
    apkPath="./nativeapp/android/${versionDir}"
    files=`ls -a ${apkPath}`

    # 調整提示(排除 ".", "..", ".DS_Store")
    text=${files//[ .
    ]/ }
    text=${text/     /}
    text=${text/DS_Store/}

    echo " -> 開始安裝🔨️: $text ..."

    for file in $files; do
      # -a:  且, 多重條件判斷
      # 排除 ".", "..", ".DS_Store"
      if [ x"$file" != x"." -a x"$file" != x".." -a x"$file" != x".DS_Store" ]; then
        # 沒 .apk 則打印訊息，否就安裝 .apk
        if [[ ! -f  "${apkPath}/${file}/${file}.apk" ]]; then
            echo " -> ${file}.apk 不存在"
            # exit 1
        else
          echo " -> 安裝 ${file}.apk ..."

          # 執行 adb devices, 過濾掉提示(List of devices attached)、以及虛擬機(emulator-XXXX device)
          # 取得實際上連接電腦的設備
          phone=`adb devices | awk '{if($1!~/^List|^emulator/) print $1}'`

          adb -s $phone install -r "${apkPath}/${file}/${file}.apk"

          # adb install -r "${apkPath}/${file}/${file}.apk"
        fi
      fi
    done

    echo " -> 🔚 已安裝完成\n"
    exit 0
    ```

- 執行結果

    ![](https://i.imgur.com/CwB1riE.png)

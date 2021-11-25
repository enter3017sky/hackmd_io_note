# React Native Note

## 目錄


- ### 腳本


    - #### [批量下載描述檔](#profile)
    - #### [批量安裝 apk](#install_apks)
    - #### [批量安裝 apk](#install_apks)
    - #### [批量安裝 apk](#install_apks)

<!--
[Inline HTML](#html)  

<a name="html"/>
 -->


---

<a name="profile"/>

### 批量下載描述檔

- 目的：節省下載描述檔 _(*.mobileprovision)_ 的時間。

- 事前準備：
    - 安裝 Ruby
    - 安裝 [Spaceship](https://github.com/fastlane/fastlane/tree/master/spaceship) 套件
    - Apple Developer 帳號


```ruby
#
# spaceship 文件
# https://github.com/fastlane/fastlane/blob/master/spaceship/docs/DeveloperPortal.md#push-certificates
#
# 下載 .p12 的部分參考自這個 issues
# https://github.com/fastlane-old/spaceship/issues/118
#
# profile = profile.repair!
# https://github.com/fastlane/fastlane/issues/2045
#

require "spaceship"

class DevelopPortalHandle
    # 等於 JS 的 constructor
    def initialize(target_app_regex, base_path)
        puts "-> initialize(): 建立處理程序"

        @target_app_regex = target_app_regex
        @base_path = base_path


        if !Dir.exists? @base_path then
            "-> 建立目錄"
            Dir.mkdir(@base_path)
        end

        puts @target_app_regex
        puts @base_path
    end

    def login()
        puts "-> login(): 登入蘋果開發者中心"
        Spaceship::Portal.login("帳號", "密碼")
        Spaceship.client.team_id = "團隊_ID"
    end

    def findApp()
        # 取得所有可用的應用程式
        all_apps = Spaceship::Portal.app.all

        # 篩選出後綴 og 的
        all_og_apps = all_apps.select{ |app| app.bundle_id =~ @target_app_regex }

        result = all_og_apps.map do |app|
            # 下載過就不下載了
            if File.exist?(getProfileName(app.bundle_id.split(".").last)) then
                puts "#{app.bundle_id} Pass!!"
            else
                downloadAdHocProvision(app.bundle_id) # 也能這樣寫 app[:bundle_id]
            end

            # 相當於 return app.bundle_id
            app.bundle_id
        end

        # 建立下載清單
        createResultTxt(result)
    end

    def createAdHocProvision(app_id)
        puts "      -> createAdHocProvision(): 建立描述檔"
        bundle_id = app_id
        name = app_id.split(".").last
        prod_certs = Spaceship::Portal.certificate.production.all

        puts "          => 設定憑證, 所有設備"
        profile = Spaceship::Portal.provisioning_profile.ad_hoc.create!(
            bundle_id: bundle_id,
            certificate: prod_certs,
            name: name
        )

        return profile
    end

    def fixAndUpdateProfile(profile)
        puts "      => 檢查描述檔狀態 #{ profile.status}"

        # Add all available devices to the profile
        profile.devices = Spaceship::Portal.device.all

        # Push the changes back to the Apple Developer Portal
        profile = profile.update!

        # 如果狀態失效或過期，就修復它
        if (profile.status == "Invalid" or profile.status == "Expired")
            puts "      => 修復描述檔狀態"
            profile = profile.repair!
        end

        return profile
    end

    def     addServices(app_id)
        puts "->    addServices(): 設定 App 服務(打開 Push Notifications)"

        app = Spaceship::Portal.app.find(app_id)
        app.update_service(Spaceship::Portal.app_service.push_notification.on)
        puts "->    addServices(): 設定 App 服務 完成。"
        puts "      => app: #{app}"
    end

    def downloadAdHocProvision(app_id)
        puts "-> downloadAdHocProvision(): 下載描述檔 #{app_id}"

        # 查找描述檔(provisioning profile)
        filtered_profiles = Spaceship::Portal.provisioning_profile.ad_hoc.find_by_bundle_id(bundle_id: app_id)

        profile = nil

        # 有找到存在的 Profile
        if  0 < filtered_profiles.length then
            puts "      => 描述檔已存在"

            addServices(app_id)
            profile = fixAndUpdateProfile(filtered_profiles[0])

        # 沒有則建立
        elsif 0 == filtered_profiles.length then
            puts "      => 描述檔不存在"
            puts "      => 建立描述檔"
            profile = createAdHocProvision(app_id)
        end

        # 下載描述檔
        downloadProfile(app_id.split(".").last, profile.download)
        puts "      => 下載完成"
    end

    def getProfileName(name)
        # puts " check: #{@base_path}/#{name}.mobileprovision"
        return "#{@base_path}/#{name}.mobileprovision"
    end

    def downloadProfile(name, profile)
        File.write(getProfileName(name), profile)
    end

    # 建立下載清單
    def createResultTxt(result)
        time = Time.new
        time = time.strftime("%Y-%m-%d_%H:%M")

        File.open("#{@base_path}/all_app-#{time}.txt", "w") { |file| file.write(result) }

        puts time
    end


end


puts ""
puts "🔨️ iOS 自動化程式開始執行"


# 找後綴符合 og 的 APP
target_app_regex = /^com\..+\.og.*/
base_path = "mp3"

handle = DevelopPortalHandle.new(
    target_app_regex,
    base_path
)

handle.login()

begin
    # 批量下載描述檔的過程中，可能會出現 session 過期的噴錯
    # 捕獲錯誤繼續下載(但還沒測試)
    handle.findApp()

    rescue => ex
            puts "ex  #{ex}"
        if ex.to_s.include? "Your session has expired. Please log in."

            # That's the most common failure probably
            puts "      繼續下載"
            handle.login()
            handle.findApp()
        else
            raise ex
    end
end

puts "🔚 執行完成\n"
puts ""
```


<a name="install_apks"/>

### 批量安裝 apk

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


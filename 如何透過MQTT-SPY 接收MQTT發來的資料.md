# 如何透過MQTT-SPY 接收MQTT發來的資料

1. ### [前往官方下載頁](https://www.eclipse.org/paho/index.php?page=downloads.php)，選擇適合的版本下載
    2. 參考[連結](https://github.com/eclipse/paho.mqtt-spy/releases/download/1.0.0/mqtt-spy-1.0.0.jar)
2. 開啟後可看見以下視窗
    * ![](https://i.imgur.com/EegGOGv.png)

3. 點選紅色底線，進入以下視窗
    * ![](https://i.imgur.com/Jqk9cDD.png)
    * Connection name可自由填入
    * Connectivity分頁
        * Sever URI 需填入MQTT server的IP/Port
        * Client ID 可自由填入
    * Security分頁
        * 若server有限制，只讓有帳密的使用者Subscribe，需填入帳號密碼，如下圖
        * ![](https://i.imgur.com/lhTbJ4v.png)
        * 填入User name / Password
            * 勾選 Predefined
    * 點擊Apply
    * 點擊Close and re-open existing connection
    * 若出現綠色分頁，表示訂閱成功
        * ![](https://i.imgur.com/jeb1xew.png)
    * 點擊綠色分頁，進入subscribe頁面
        * ![](https://i.imgur.com/UBNEmVS.png)
        * 點選紅色底線(New)分頁
            * 輸入Topics
            * 指定特定topic: solarplatform/gateway/03760501002003
            * 指定某網段底下的所有topic: solarplatform/gateway/#            * 




4. 觀察是否有正常收到訊息 (參考上圖的Data欄位內容)
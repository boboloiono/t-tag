# [W18] Final Project：第三組 T-tag
###### tags: `單晶片系統設計與應用`
🌟[成果報告書](https://drive.google.com/file/d/1JICwqNZXdJFlhLjRmfHX6R104Dic-sfl/view?usp=sharing)

## 📍產品目標
1. 讓使用我們產品的公司，能清楚掌握場所的人流狀況
2. 提供給政府不同場域的資料，作為未來決策參考。
## 📍材料
1. ESP32 因為內建wifi和藍芽功能，取代Arduino UNO版。T-tag的主機，可用來進出場所時非接觸式感應。
2. MLX90614 感測目前溫度（NT$ 280）
3. SSD1306 OLED 顯示目前溫度（NT$ 95）
4. Button 控制溫度顯示
5. RGB LED 依造體溫狀況亮特定顏色
6. HC-05 連接ESP32的藍芽，用來定位
7. Line Notify 帳戶

## 📍ESP32 主程式
### 安裝及設定ESP32的開發環境
:::success
參考網站：[Windows看這篇](https://makerpro.cc/2020/06/how-to-install-and-configure-esp32-development-environment/)、[Mac看這篇](https://randomnerdtutorials.com/installing-the-esp32-board-in-arduino-ide-mac-and-linux-instructions/)
:::
#### [注意] 除了開發版要設定之外，一定要根據你的板子選擇對應的「Upload Speed」、「Flash Frequency」
![](https://i.imgur.com/cHWH6ff.png)
#### 如何知道你的板子特性？可用此code簡單測試：
```c++=
void setup(){
  Serial.begin(115200);
}
void loop(){
  Serial.println("Hello World");
  delay(1000);
} 
```
板子資訊即會顯示在視窗，而其他測試細節（像是 ESP32 的connect方式都在上述的 [Windows看這篇](https://makerpro.cc/2020/06/how-to-install-and-configure-esp32-development-environment/) 後半部文章
### 設定完成後，開始打你的ESP32主程式
💡注意：為了要連線 Line Notify，記得設定 wifi名稱（SSDI ）和 wifi 密碼。
手機開熱點也可連通！

> #### 如何尋找可用的 WiFi？
> 點選 Arduino `檔案 ＞ 範例 ＞WiFi ＞WiFiScan`
> 螢幕上會出現程式 WiFiScan。點選 Arduino 上傳按鈕，讓 ESP32 直接掃描您所處環境的 WiFi，並顯示可用 wifi 名稱和強度。
### 更改ESP32記憶體空間
更改ESP32記憶體空間（[相關文章](https://www.google.com/search?q=boards.txt&client=ms-android-twm-tw-revc&sourceid=chrome-mobile&ie=UTF-8#fpstate=ive&vld=cid:ea1b6830,vid:sQ5cQ8DORnY,st:0)）
在 Arduino/Hardware/ESP32/1.06/Boards.txt 中間處**新增以下三行**（[參考影片](https://www.youtube.com/watch?v=sQ5cQ8DORnY)）
```
esp32doit-devkit-v1.menu.PartitionScheme.huge_app=Huge APP (3MB No OTA/1MB SPIFFS)
esp32doit-devkit-v1.menu.PartitionScheme.huge_app.build.partitions=huge_app
esp32doit-devkit-v1.menu.PartitionScheme.huge_app.upload.maximum_size=3145728
```
```c=
//首先定義會使用到的程式庫
#include <WiFi.h>
#include <WiFiClient.h>
#include <TridentTD_LineNotify.h>
#include <SPI.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_MLX90614.h>
#include <BluetoothSerial.h>

//藍芽
BluetoothSerial BT;

// 修改成上述寄到登入郵箱的 Token權杖號碼
#define LINE_TOKEN "nivPrNlpL5ydIKySLXxpRUFWme8y2tCPWRClmLHT2eA"

// 設定無線基地台SSID跟密碼
const char* ssid     = "Haha";
const char* password = "111122233";// 設定無線基地台SSID跟密碼

//定義MLX90614
Adafruit_MLX90614 mlx = Adafruit_MLX90614();

//定義OLED
#define SCREEN_WIDTH 128 // OLED display width, in pixels
#define SCREEN_HEIGHT 64 // OLED display height, in pixels
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

//定義紅黃綠led燈腳位
int Red_Led = 26;
int Yellow_Led = 25;
int Green_Led = 33;

//定義按鈕腳位
int button = 13;

//判斷有沒有按按鈕
bool button_press = true;
bool swstate_up = digitalRead(13);


void setup() {
  
  WiFi.mode(WIFI_STA);
  // 連接無線基地台
  WiFi.begin(ssid, password);
  Serial.print(F("\n\r \n\rWorking to connect"));
  // 等待連線
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  //成功連接
  Serial.print("Connected！IP 位址：");
  Serial.print(WiFi.localIP());
  
  //OLED
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);  //I2C address 0x3C (for the 128x32)
  //紅綠黃LED燈
  pinMode(Red_Led, OUTPUT);
  pinMode(Yellow_Led, OUTPUT);
  pinMode(Green_Led, OUTPUT);
  //Button
  pinMode(13, INPUT_PULLUP);  //內建上拉電阻
  //序列埠
  Serial.begin(57600);
  //感溫器
  mlx.begin();
  //藍芽名字
  BT.begin("Ttag");
}

//收到的地點資料
char pos='0';
char past=' ';
//量測的溫度
float temp;
String place=" ";
String tempe=" ";

void loop() {
  
  //判斷有沒有按按鈕
  swstate_up = digitalRead(13);

  //當按下按鈕(低電位)
  if (swstate_up == LOW) {
    //按按鈕量溫度
    temp = mlx.readObjectTempC();
    
    //這裡開始定義OLED上要即時顯示的畫面有什麼
    display.setTextSize(3);
    display.setTextColor(WHITE);
    display.setCursor(0, 0);
    display.setCursor(0, 10);
    display.print(temp);
    display.print("C");
    display.display();
    
    //紅綠黃LED亮暗判斷，顯示三秒
    if (temp >= 35 && temp <= 38) { //正常體溫
      digitalWrite(Green_Led, HIGH);
      delay(3000);
      digitalWrite(Green_Led, LOW);
      Serial.print("green");
    }
    else if (temp > 38) { //發燒
      digitalWrite(Red_Led, HIGH);
      delay(3000);
      digitalWrite(Red_Led, LOW);
      Serial.print("red");
    }
    else  { //體溫異常
      digitalWrite(Yellow_Led, HIGH);
      delay(3000);
      digitalWrite(Yellow_Led, LOW);
      Serial.print("yellow");
    }
    //顯示溫度
    Serial.print(" 體溫 = ");
    Serial.print(mlx.readObjectTempC()); //object temperature
    Serial.println(" *C");
    Serial.println(LINE.getVersion());
  }

  //清除螢幕資料
  else {
    display.clearDisplay();  
    display.display();
  }
  
  //看藍芽有沒有回傳地點
  if (BT.available()) {
    pos=BT.read();
  }

  //判斷地點體溫資料
  if(pos=='1'){
    place="電機系館";
  }
  else if(pos=='2'){
    place="奇美樓";
  }

  //判斷需不需要回傳Line
  delay(1000);
  Serial.println(pos);
  if(pos!='0'){
    Serial.println("line");
    //past=pos;
    tempe = "溫度" + String(temp) + "℃";
    // Line Notify 顯示體溫
    LINE.setToken(LINE_TOKEN); 
    LINE.notify("\n" + tempe+" "+place);
  }
  delay(500);  
  pos='0';
}
```
## 📍場所固定式定位裝置
使用arduino uno板作為此定位裝置的處理器，外接hc-05藍芽模組
當進入到此裝置藍芽接收範圍，傳送場所資訊到個人裝置t-tag上，連動到下一步的line notify顯示

程式碼：
```
#include  <SoftwareSerial.h>
SoftwareSerial BT(10, 11); // 宣告10腳位為Arduino的RX 、11為Arduino的 TX
char val;  //儲存接受到的資料變數
char place = '1';

void setup() {
  Serial.begin(38400);
  Serial.println("hoho");
  BT.begin(38400); //注意，HC-06要設定成9600(bps)
}

void loop() {
// 若收到「序列埠監控視窗」的資料，則送到藍牙模組
  //傳送位置給ESP32
  BT.print(place);
  delay(1000);
  //AT模式需要的
  //if(Serial.available()){
  //  BT.write(Serial.read());
  //}
  //if(BT.available()){
  //  Serial.write(BT.read());
  //}
}
```

## 📍連動到 Line Notify 顯示體溫資訊
:::success
超棒的參考資料：[Esp32 + LINE](https://ithelp.ithome.com.tw/articles/10271219)
:::
STEP1：下載Line Notify函式庫
STEP2：登入Line Notify，建立Server 並取得Token
STEP3：將 Line Notify 的所需連動資訊拷貝 ESP32 主程式，讓 ESP32 有目標且有權限的去訪問。

## 📍利用 MySQL 建立資料庫
[參考文章](https://youyouyou.pixnet.net/blog/post/120275917-%E7%AC%AC%E5%8D%81%E4%B8%80%E7%AF%87-esp32-%E8%B3%87%E6%96%99%E5%BA%AB%E5%AD%98%E5%8F%96mysql%E9%80%A3%E7%B7%9A)

---

## 📣 接下來的進度
1. 目前是一直量、一直跳Notify，可改為「按一下 button 就量一次體溫、跳一次 Line Notify 」
2. 使用藍芽對多個 slaves
3. 將記錄到的體溫放到資料庫，建立「人：時間：場所：體溫」的資料庫
	- 可用 mongoDB 建立資料庫
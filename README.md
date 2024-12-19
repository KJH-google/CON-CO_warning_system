CON-CO_warning_system
=====================
## 배경_일산화탄소(CO) 
### 1. 일산화탄소란?
* 무색, 무취 기체로서, 산소가 부족한 상태에서 석탄이나 석유 등 연료가 탈 때 발생
* 주요 발생원: 연탄, 연소가스, 자동차 배기가스, 산불

### 2. 일산화탄소 중독
* 주요 증상: 두통, 매스꺼움, 구토, 이명, 호흡곤란, 맥박 증가
* 일산화탄소 농도별 인체 영향
> * 3200 ppm - 5분 이내 두통, 어지러움, 구토, 20분 이내 사망
> * 1600 ppm - 10분 이내 두통, 어지러움, 구토, 30분 이내 사망
> * 800 ppm - 20분 이내 두통, 어지러움, 구토, 1시간 이내 사망
> * 400 ppm - 1시간 이내 두통, 어지러움, 구토, 3시간 이내 사망
> * 200 ppm - 2~3시간 이내 두통, 피곤, 어지러움
> * 50 ppm - 영향 거의 없음, 작업장 환경기준 허용농도

### 3. 캠핑 CO 중독사고
> <img src="https://github.com/user-attachments/assets/b12e9047-c89a-4298-837d-5acabe06681d" width="80%" />

> * 해마다 계속해서 증가 추세임.

## 아두이노를 활용한 센서 및 부저 조립과 구현

### 1. MQ-9 가스센서 
#### <개요>
* MQ-9 가스 센서는 다양한 가스의 농도를 감지할 수 있는 센서로, 특히 일산화탄소(CO), 메탄(CH₄), 그리고 액화석유가스(LPG)와 같은 가연성 가스에 민감합니다. 이 센서는 가스 누출 감지, 대기 질 모니터링, 산업 안전 시스템 등에 사용됩니다.
#### <12/2(월)_JETBOT과 연결 및 측정 시도>
> <img src="https://github.com/user-attachments/assets/f15ed0e5-3371-40a2-96b0-ed486a6294dc" alt="Image 1" style="width: 30%;"/>
> <img src="https://github.com/user-attachments/assets/12600b08-ccf8-4090-a4b6-e7ea8cf2b8ab" alt="Image 2" style="width: 30%;"/>

> <img src="https://github.com/user-attachments/assets/d4c0ebcd-934b-4385-bb7f-859cee0ae6cc" alt="Image 2" style="width: 30%;"/>
> <img src="https://github.com/user-attachments/assets/087457fa-1963-446a-8c8b-a9a2f2d3a42c" alt="Image" style="width: 30%;"/>

> * MQ-9을 블로그에 나오는데로 다 해봤는데 농도 변화가 크게 차이가 나지 않는다. (블로그 링크 : [https://cafe.naver.com/mechawiki/3521])
> * MQ-9이랑 MQ-7의 차이는 7이 일산화탄소를 집중적으로 더 잘잡는다고 해서 7으로 교체했다.
>    * MQ-7은, CO 감지에만 초점이 맞춰져 있어서 감지 범위가 20~2000 ppm으로 넓으며, 농도가 더 높은 환경에서도 안정적으로 작동한다.
>    * MQ-9은 다양한 가스와의 반응을 고려해야 하므로, 특정 가스(CO)에 대한 민감도가 상대적으로 낮아질 수 있다.


* MQ-9 사용 코드   
<pre>
<code>
const int MQ9_AOUT_PIN = A0; // MQ-9의 AOUT 핀이 연결된 아날로그 핀
float sensorValue;           // 센서 출력 값

void setup() {
  Serial.begin(9600); // 시리얼 통신 시작
  pinMode(MQ9_AOUT_PIN, INPUT); // MQ-9 핀을 입력으로 설정
}

void loop() {
  // 아날로그 값 읽기
  sensorValue = analogRead(MQ9_AOUT_PIN);

  // 센서 값 변환 (0~1023 → 가스 농도)
  float voltage = sensorValue * (5.0 / 1023.0); // 5V 기준 전압 변환
  float co_concentration = voltage * 100;      // 임의 농도 계산 (보정 필요)

  // 시리얼 모니터에 출력
  Serial.print("Sensor Value: ");
  Serial.print(sensorValue);
  Serial.print(" | Voltage: ");
  Serial.print(voltage);
  Serial.print(" V | CO Concentration: ");
  Serial.print(co_concentration);
  Serial.println(" ppm"); // ppm 단위로 출력 (보정된 농도)

  delay(1000); // 1초 간격으로 업데이트
}
</code>
</pre>


***
### 2. MQ-7 가스센서
#### <개요>
> <img src="https://github.com/user-attachments/assets/bfb6f645-0cb4-4575-8f2f-348e193f5dd4" width="40%" />
* MQ-7은, MQ-7은 일산화탄소(CO) 감지에 특화된 센서로, 높은 CO 감도와 20~2000ppm의 농도를 측정할 수 있습니다. 히터 전압을 주기적으로 변경하여 작동하며, 주로 보일러실이나 자동차 배기가스 감지에 사용됩니다.
* MQ-7 작동 방식
  * <img src="https://github.com/user-attachments/assets/fb709418-5ef7-4d99-9015-0b68cc8758cf" width="40%" />

#### <12/4(수)_JETBOT과 연결 및 측정 시도>
> <img src="https://github.com/user-attachments/assets/d91a8dbe-538e-4ac1-be59-bfde55e4357c" alt="Image 1" style="width: 30%;"/>
> <img src="https://github.com/user-attachments/assets/c4af8b61-ead2-4e2d-98b6-d1453c07f33d" alt="Image 2" style="width: 50%;"/>

> * 디스코드를 활용해 임계치 도달 시 알림이 뜨게 하기 위해서 추가로 코딩이 필요하고, 부저도 활용해볼 계획.
> * 센서의 아날로그 출력 값을 ppm 단위로 변환하기 위해, a,b 값을 찾아야 한다.
>    * 데이터 시트에 있는 CO Curve 식에 가장 일반적으로 사용되는 값인, a=100, b= -1.5을 선택하고 대입하여 사용했다.

***

### 3. 디스코드로 알림 받기, 부저 알림
#### <개요>
* MQ-7 센서로 측정한 CO 농도를 일정 기준에 맞춰 상태(정상, 위험 등)를  특정한 후 Discord로 주의하라는 메시지 알림을 사용자에게 보낸다.
    * <img src="https://github.com/user-attachments/assets/123dbc8f-32cc-46bb-8721-32488e455aec" width="50%" />

* 능동 피에조 부저를 설치해 특정 CO 농도(ppm)에서 소리 알람이 발생하도록 설계한다.
    * 설정한 CO 농도값에 따라 다른 주파수와 소리 지속 시간, 소리 대기 시간이 존재함.
    * <img src="https://github.com/user-attachments/assets/27d8d720-a6f5-44b3-8561-f2664e0d9454" width="50%" />



#### <12/7(토)_실험 진행>
> <img src="https://github.com/user-attachments/assets/cb513ab1-e134-4dc4-8d2e-038b23f017f9" alt="Image 1" style="width: 40%;"/>
> <img src="https://github.com/user-attachments/assets/b747a1c3-58d0-4578-9070-5692ccf27e07" alt="Image 3" style="width: 50%;"/>
> <img src="https://github.com/user-attachments/assets/b470708c-b824-4159-8042-7b94a2f51f14" alt="Image 4" style="width: 40%;"/>
> <img src="https://github.com/user-attachments/assets/fbc4c6bf-ba47-4d46-963b-7fb9b8d64b3e" alt="Image 5" style="width: 40%;"/>

> * 최종적인 실험에서 1.5g 숯 3개를 사용하였다.
> * 처음에는 200, 800, 3200 ppm (WHO 권고 기준)에서 부저 알림 및 디스코드 경고 알림을 설정하였다.
> * 그러나 실험 진행 중에 농도가 충분히 높이 올라가지 않았고, 여건상 높은 CO 농도를 측정하기 어려움이 있었다.
>    * 양초 1개/5개, 작은 크기의 숯으로 시도하였으나 여전히 낮은 농도에 머무름.
>    * 해결 방법 : 설정 기준치를 변경함.
>        * <img src="https://github.com/user-attachments/assets/ab9dee1c-4524-4301-b318-69248710e9f6" width="50%" />


* MQ-7 가스 센서, 부저 알림 아두이노 IDE
<pre>
<code>
const int MQ7_AOUT_PIN = A0;  // MQ-7 센서 아날로그 출력 핀
const int buzzerPin = 8;      // 피에조 부저 연결 핀 (디지털 핀 8)
const float VCC = 5.0;     // Arduino 공급전압  
const float RL = 10.0;   // 로드 저항 값 - 센서 회로에 사용 (보통 10kΩ)
const float R0 = 11.37;    // MQ-7 센서 기준 저항  
const float a = 19.32;    // MQ-7 특정곡선 상수  
const float b = -0.64;    // MQ-7 특정곡선 상수  

void setup() {
  Serial.begin(9600);
  pinMode(buzzerPin, OUTPUT);
  digitalWrite(buzzerPin, LOW); // 초기 상태: 부저 OFF
}

void loop() {
  int sensorValue = analogRead(MQ7_AOUT_PIN);
  float voltage = (sensorValue / 1023.0) * VCC;
  float RS = RL * (VCC - voltage) / voltage;
  float ratio = RS / R0;
  float ppm = a * pow(ratio, b);

  // ppm 값을 시리얼로 전송
  Serial.println(ppm);

  // 상태 판단
  // 정상: ppm < 200
  // 주의: 200 ≤ ppm < 800
  // 위험: 800 ≤ ppm < 3200
  // 매우 위험: ppm ≥ 3200
  if (ppm < 1) {
    // 정상: 부저 꺼짐
    noTone(buzzerPin);
  } else if (ppm < 5) {
    // 주의: 낮은 톤(1000Hz)으로 짧게 울림
    // 1초 루프 내에서 짧게 200ms 울리고 꺼짐
    tone(buzzerPin, 1000, 200);
    delay(200);
    noTone(buzzerPin);
    // 나머지 시간 대기
  } else if (ppm < 20) {
    // 위험: 중간 톤(2000Hz)으로 울림
    // 조금 더 길게 500ms 울리고 500ms 꺼짐 (총 1초 주기)
    tone(buzzerPin, 2000, 500);
    delay(500);
    noTone(buzzerPin);
    delay(500);
  } else {
    // 매우 위험: 높은 톤(3000Hz)으로 계속 울림
    // 여기서는 tone을 계속 주기 어렵기 때문에 다음 루프에서도 같은 상태이면 연속음에 가깝게
    tone(buzzerPin, 3000); 
    // 1초 기다린 후 다시 loop 진입 -> 사실상 계속 울림
    delay(1000);
    return; // 아래 delay(1000)로 안 내려가도록
  }

  delay(1000); // 다음 측정까지 1초 대기
}
</code>
</pre>

* RO 보정 아두이노 IDE
<pre>
<code>
const int MQ7_PIN = A0;   // MQ-7 센서 아날로그 출력 핀
const float VCC = 5.0;    // Arduino 공급 전압
const float RL = 10;    // 로드 저항값(kΩ) - 사용자가 회로 설계 시 정한 값
const float CLEAN_AIR_RATIO = 1; // 깨끗한 공기에서 RS/R0 비율 (예: MQ-7 데이터시트 참조)
float R0 = 28.73; // 초기값, 실제 보정 후에 업데이트

void setup() {
  Serial.begin(9600);
  // 센서 예열 시간 (예: MQ-7은 10~20분 이상 권장, 여기서는 간단히 2분 예)
  // 실제로는 setup 후 일정 시간 대기하거나, 측정 시작 전 기다린 후 보정할 것.
  Serial.println("Sensor preheating...");
  delay(10000); // 1분 예열 (실제 권장시간 확인)

  Serial.println("Calibrating R0. Please ensure the sensor is in clean air.");
  // 보정 측정값 여러 번 읽기 (예: 50회) 평균을 내어 안정적인 값 사용.
  float avgRS = 0;
  int numReadings = 50;
  for (int i = 0; i < numReadings; i++) {
    float sensorValue = analogRead(MQ7_PIN);
    float voltage = (sensorValue / 1023.0) * VCC;
    float RS = RL * (VCC - voltage) / voltage; // RS 계산식
    avgRS += RS;
    delay(200); // 각 측정 사이 약간 대기
  }
  avgRS = avgRS / numReadings; // 평균 RS

  // R0 계산: R0 = RS / (RS/R0(clean air))
  R0 = avgRS / CLEAN_AIR_RATIO;

  Serial.print("Calibrated R0: ");
  Serial.println(R0, 4);
}

void loop() {
  // 보정 완료 후, ppm 계산 시 R0 사용 예제
  // ppm = a * (RS/R0)^b 형태를 사용
  // 여기서는 단순히 R0가 제대로 계산되었는지 확인하는 코드만 둠.
  
  float sensorValue = analogRead(MQ7_PIN);
  float voltage = (sensorValue / 1023.0) * VCC;
  float RS = RL * (VCC - voltage) / voltage;
  float ratio = RS / R0;  // Ratio = RS/R0

  // 예: MQ-7 데이터시트 곡선 상수 a,b 정의 후 ppm 계산 가능
  // float a = 100.0;
  // float b = -1.5;
  // float ppm = a * pow(ratio, b);

  Serial.print("RS: "); Serial.print(RS);
  Serial.print(" R0: "); Serial.print(R0);
  Serial.print(" Ratio: "); Serial.println(ratio);
  
  delay(1000);
}
</code>
</pre>

    
* 디스코드 알림 전송
<pre>
<code>
#include <WiFi.h>
#include <HTTPClient.h>

const char* ssid = "Your_SSID";      // Wi-Fi 이름
const char* password = "Your_PASS";  // Wi-Fi 비밀번호
const char* webhook_url = "https://discord.com/api/webhooks/1313826821787226132/txS4YAXl6tm_5UWQVzSCX0rQRLGOOELs2a_9PIk3vMNALzxxX2r88bDJcZ6f0K5v_3oe"; // 디스코드 웹훅 URL

const int MQ7_AOUT_PIN = A0; // MQ-7의 AOUT 핀
const float VCC = 5.0;       // 아두이노 전원 공급 전압
const float RL = 10.0;       // 로드 저항 (kΩ)
const float R0 = 10.0;       // 깨끗한 공기에서의 센서 저항 (kΩ)
const float a = 100.0;       // 데이터시트 상수 a
const float b = -1.5;        // 데이터시트 상수 b
float ppm_threshold = 200.0; // 경고 임계값 (단위: ppm)

void setup() {
  Serial.begin(9600);
  pinMode(MQ7_AOUT_PIN, INPUT);

  // Wi-Fi 연결
  WiFi.begin(Water430ho_5G, Water430ho_5G);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("\nConnected to WiFi!");
}

void loop() {
  int sensorValue = analogRead(MQ7_AOUT_PIN); // 아날로그 값 읽기
  float voltage = (sensorValue / 1023.0) * VCC; // 전압 계산
  float RS = RL * (VCC - voltage) / voltage;   // 센서 저항 계산
  float ratio = RS / R0;                       // 센서 비율 계산
  float ppm = a * pow(ratio, b);               // PPM 계산

  // 결과 출력
  Serial.print("Sensor Value: ");
  Serial.print(sensorValue);
  Serial.print(" | Voltage: ");
  Serial.print(voltage);
  Serial.print(" V | CO Concentration: ");
  Serial.print(ppm);
  Serial.println(" ppm");

  // 농도가 임계값을 초과하면 디스코드로 경고
  if (ppm > ppm_threshold) {
    sendToDiscord(ppm);
  }

  delay(1000); // 1초 간격 업데이트
}

// 디스코드로 메시지 보내는 함수
void sendToDiscord(float ppm) {
  if (WiFi.status() == WL_CONNECTED) { // Wi-Fi 연결 확인
    HTTPClient http;
    http.begin(webhook_url);
    http.addHeader("Content-Type", "application/json");

    String message = "{\"content\":\"⚠️ Warning: High CO Concentration! PPM: " + String(ppm) + "\"}";

    int httpResponseCode = http.POST(message);

    if (httpResponseCode > 0) {
      Serial.println("Message sent to Discord!");
    } else {
      Serial.print("Error sending message: ");
      Serial.println(httpResponseCode);
    }

    http.end(); // HTTP 연결 종료
  
</code>
</pre>
* 위의 코드는 디스코드 알람을 보내는 방법 중 WIFI 모듈을 이용하는 방법으로 WIFI 모듈이 없는 관계상 밑의 Python의 serial과 time, requests를 이용한 방법으로 수정
* Python 코드
<pre>
<code>
import serial
import time
import requests
from datetime import datetime

# Arduino와 연결할 시리얼 포트 설정
PORT = "COM3"  # Arduino가 연결된 포트 (Windows에서는 COMx, Mac/Linux에서는 /dev/ttyUSBx)
BAUDRATE = 9600  # Arduino와 동일한 보드레이트 설정
WEBHOOK_URL = "YOUR_DISCORD_WEBHOOK_URL"  # 디스코드 웹훅 URL 입력

# CO 경고 임계값
CO_THRESHOLD = 50.0  # PPM 단위

def send_discord_alert(ppm):
    """디스코드로 경고 메시지 전송"""
    current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    color = 0xFF0000 if ppm >= CO_THRESHOLD else 0xFFFF00

    data = {
        "embeds": [{
            "title": "⚠️ CO 농도 경고 ⚠️",
            "description": f"현재 CO 농도는 {ppm:.2f} ppm 입니다.",
            "color": color,
            "fields": [
                {"name": "상태", "value": "위험 수준 도달" if ppm >= CO_THRESHOLD else "정상", "inline": True},
                {"name": "측정 시간", "value": current_time, "inline": False}
            ],
            "footer": {"text": "CO 모니터링 시스템"}
        }]
    }

    try:
        response = requests.post(WEBHOOK_URL, json=data)
        if response.status_code == 204:
            print(f"[{current_time}] 디스코드 알림 전송 성공: {ppm:.2f} ppm")
        else:
            print(f"디스코드 알림 전송 실패: {response.status_code}")
    except Exception as e:
        print(f"디스코드 알림 오류: {str(e)}")

def main():
    try:
        # Arduino 시리얼 포트 열기
        ser = serial.Serial(PORT, BAUDRATE, timeout=1)
        print(f"Arduino 연결됨: {PORT}")
        time.sleep(2)  # Arduino 초기화 대기

        while True:
            # 시리얼 데이터 읽기
            if ser.in_waiting > 0:
                line = ser.readline().decode('utf-8').strip()
                try:
                    ppm = float(line)
                    print(f"CO 농도: {ppm:.2f} ppm")

                    # CO 농도가 임계값 초과 시 디스코드 알림 전송
                    if ppm >= CO_THRESHOLD:
                        send_discord_alert(ppm)

                except ValueError:
                    print(f"잘못된 데이터 수신: {line}")
            time.sleep(1)

    except serial.SerialException as e:
        print(f"시리얼 연결 오류: {str(e)}")
    except KeyboardInterrupt:
        print("프로그램 종료")
    finally:
        if 'ser' in locals() and ser.is_open:
            ser.close()
            print("시리얼 포트 닫힘")

if __name__ == "__main__":
    main()
</code>
</pre>

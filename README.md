# CON-CO_warning_system

const int MQ7_AOUT_PIN = A0; // MQ-7의 AOUT 핀이 연결된 아날로그 핀
const float VCC = 5.0;       // 아두이노 공급 전압
const float RL = 10.0;       // 로드 저항 (단위: kΩ)
const float R0 = 10.0;       // 깨끗한 공기에서의 센서 저항 (단위: kΩ)
const float a = 100.0;       // 데이터시트에서 제공되는 상수 a
const float b = -1.5;        // 데이터시트에서 제공되는 상수 b

void setup() {
  Serial.begin(9600); // 시리얼 통신 시작
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

  delay(1000); // 1초 간격으로 업데이트
}

-------------------------------------------------------------------


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
  } else {
    Serial.println("WiFi not connected!");
  }
}



------------------------------------------------------------

const int MQ7_AOUT_PIN = A0; // MQ-7의 아날로그 출력 핀

const float VCC = 5.0;       // Arduino 공급 전압
const float RL = 10.0;       // 로드 저항 (단위: kΩ)
const float R0 = 10.0;       // 깨끗한 공기에서 센서 저항 (단위: kΩ)
const float a = 100.0;       // 데이터시트에서 제공되는 상수 a
const float b = -1.5;        // 데이터시트에서 제공되는 상수 b

void setup() {
  Serial.begin(9600); // 시리얼 통신 시작
}

void loop() {
  int sensorValue = analogRead(MQ7_AOUT_PIN); // 센서 값 읽기
  float voltage = (sensorValue / 1023.0) * VCC; // 전압 계산
  float RS = RL * (VCC - voltage) / voltage;   // 센서 저항 계산
  float ratio = RS / R0;                       // 센서 비율 계산
  float ppm = a * pow(ratio, b);               // PPM 계산

  // 데이터 시리얼 포트로 전송
  Serial.println(ppm);
  delay(1000); // 1초 간격
}


python part

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

























const int MQ7_AOUT_PIN = A0; // MQ-7의 아날로그 출력 핀

const float VCC = 5.0;       // Arduino 공급 전압 (5V)
const float RL = 10.0;       // 로드 저항 값(kΩ)
const float R0 = 10.0;       // 깨끗한 공기에서 센서 기준 저항(kΩ)
const float a = 100.0;       // MQ-7 센서 곡선 파라미터 (데이터시트 기반)
const float b = -1.5;        // MQ-7 센서 곡선 파라미터 (데이터시트 기반)

void setup() {
  Serial.begin(9600); // 시리얼 통신 시작
}

void loop() {
  int sensorValue = analogRead(MQ7_AOUT_PIN); // 센서 값(0~1023)
  float voltage = (sensorValue / 1023.0) * VCC;
  float RS = RL * (VCC - voltage) / voltage;   
  float ratio = RS / R0;                       
  float ppm = a * pow(ratio, b);               

  Serial.println(ppm); // CO 농도 PPM 출력
  delay(1000); // 1초 간격
}








--------------------------------




import serial
import time
import requests
from datetime import datetime

PORT = "COM3"  # Windows 예시, Linux/Mac: /dev/ttyUSB0 등
BAUDRATE = 9600
WEBHOOK_URL = "DISCORD_WEBHOOK_URL"  # 실제 webhook URL로 변경
CO_THRESHOLD = 50.0  # 임계값 설정

def send_discord_alert(ppm):
    current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    data = {
        "embeds": [{
            "title": "⚠️ CO 농도 경고 ⚠️",
            "description": f"현재 CO 농도는 {ppm:.2f} ppm 입니다.",
            "color": 0xFF0000 if ppm >= CO_THRESHOLD else 0xFFFF00,
            "fields": [
                {"name": "상태", "value": "위험" if ppm >= CO_THRESHOLD else "정상", "inline": True},
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
        ser = serial.Serial(PORT, BAUDRATE, timeout=1)
        print(f"Arduino 연결됨: {PORT}")
        time.sleep(2)  # 초기화 대기

        while True:
            if ser.in_waiting > 0:
                line = ser.readline().decode('utf-8').strip()
                try:
                    ppm = float(line)
                    print(f"CO 농도: {ppm:.2f} ppm")
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


웹후크 = https://discord.com/api/webhooks/1313826821787226132/txS4YAXl6tm_5UWQVzSCX0rQRLGOOELs2a_9PIk3vMNALzxxX2r88bDJcZ6f0K5v_3oe



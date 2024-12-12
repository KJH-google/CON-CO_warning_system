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



















<최종코드>




<아두이노 IDE>
const int MQ7_AOUT_PIN = A0;  // MQ-7 센서 아날로그 출력 핀
const int buzzerPin = 8;      // 피에조 부저 연결 핀 (디지털 핀 8)
const float VCC = 5.0;       
const float RL = 10.0;       
const float R0 = 10.0;       
const float a = 100.0;       
const float b = -1.5;        

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
  if (ppm < 200) {
    // 정상: 부저 꺼짐
    noTone(buzzerPin);
  } else if (ppm < 800) {
    // 주의: 낮은 톤(1000Hz)으로 짧게 울림
    // 1초 루프 내에서 짧게 200ms 울리고 꺼짐
    tone(buzzerPin, 1000, 200);
    delay(200);
    noTone(buzzerPin);
    // 나머지 시간 대기
  } else if (ppm < 3200) {
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










--------------------------------

<Python>


import serial
import time
import requests
from datetime import datetime
import csv
import os

# ===== 사용자 환경에 맞게 수정해야 하는 부분 =====
PORT = "/dev/ttyACM0"  # Jetson Nano에서 Arduino 포트 확인 (예: /dev/ttyACM0)
WEBHOOK_URL = "YOUR_DISCORD_WEBHOOK_URL"  # 실제 Discord Webhook URL
CSV_FILE_PATH = "/home/your_username/data/co_readings.csv"  # CSV 파일 저장 경로
# ==============================================

# 상태 기준
# 정상: ppm < 200
# 주의: 200 ≤ ppm < 800
# 위험: 800 ≤ ppm < 3200
# 매우 위험: ppm ≥ 3200

def init_csv_file(filename):
    dir_path = os.path.dirname(filename)
    if dir_path and not os.path.exists(dir_path):
        os.makedirs(dir_path, exist_ok=True)
    # 파일이 없을 경우 헤더 기록
    if not os.path.exists(filename):
        with open(filename, mode='w', newline='', encoding='utf-8') as f:
            writer = csv.writer(f)
            writer.writerow(["timestamp", "ppm"])
        print(f"CSV 파일 생성 및 헤더 작성 완료: {filename}")

def append_csv_file(filename, timestamp_str, ppm_value):
    with open(filename, mode='a', newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        writer.writerow([timestamp_str, ppm_value])

def get_co_status(ppm):
    if ppm < 200:
        return "정상", 0x00FF00  # Green
    elif ppm < 800:
        return "주의", 0xFFFF00  # Yellow
    elif ppm < 3200:
        return "위험", 0xFFA500  # Orange-ish
    else:
        return "매우 위험", 0xFF0000  # Red

def send_discord_alert(ppm):
    """CO 농도별 단계 알림 전송"""
    current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    status, color = get_co_status(ppm)
    description = f"현재 CO 농도는 {ppm:.2f} ppm 입니다."
    title = f"⚠️ CO {status} 상태 ⚠️"
    
    data = {
        "content": "@here",
        "embeds": [{
            "title": title,
            "description": description,
            "color": color,
            "fields": [
                {"name": "상태", "value": status, "inline": True},
                {"name": "측정 시간", "value": current_time, "inline": False}
            ],
            "footer": {"text": "CO 모니터링 시스템"}
        }]
    }

    try:
        response = requests.post(WEBHOOK_URL, json=data)
        if response.status_code == 204:
            print(f"[{current_time}] 디스코드 알림 전송 성공: {ppm:.2f} ppm ({status})")
        else:
            print(f"디스코드 알림 전송 실패: {response.status_code}")
    except Exception as e:
        print(f"디스코드 알림 오류: {str(e)}")

def main():
    init_csv_file(CSV_FILE_PATH)

    try:
        ser = serial.Serial(PORT, 9600, timeout=1)
        print(f"Arduino 연결됨: {PORT}")
        time.sleep(2)  # 시리얼 초기화 대기 시간

        last_alert_status = "정상"  # 이전에 보낸 알림 상태를 추적하여 불필요한 반복 알림 방지 가능(옵션)
        
        while True:
            if ser.in_waiting > 0:
                line = ser.readline().decode('utf-8').strip()
                try:
                    ppm = float(line)
                    current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                    append_csv_file(CSV_FILE_PATH, current_time, ppm)
                    print(f"CO 농도: {ppm:.2f} ppm")

                    # 상태 판단
                    status, _ = get_co_status(ppm)

                    # 정상일 때는 알림 필요 없다고 가정(원하면 조건 수정)
                    if status in ["주의", "위험", "매우 위험"]:
                        # 상태별로 조건
                        # 이전 상태와 동일한 상태라면 계속 알림 보내지 않도록 할 수도 있음.
                        # 여기서는 매번 임계 이상이면 알림 보내도록 유지.
                        send_discord_alert(ppm)

                except ValueError:
                    print(f"잘못된 데이터 수신: {line}")
            time.sleep(1)

    except serial.SerialException as e:
        print(f"시리얼 연결 오류: {str(e)}")
    except KeyboardInterrupt:
        print("프로그램 종료 요청 받음")
    finally:
        if 'ser' in locals() and ser.is_open:
            ser.close()
            print("시리얼 포트 닫힘")

if __name__ == "__main__":
    main()



PORT = "/dev/ttyUSB0"  # Jetson Nano에서 Arduino 포트 확인 (예: /dev/ttyACM0)
WEBHOOK_URL = "https://discord.com/api/webhooks/1313826821787226132/txS4YAXl6tm_5UWQVzSCX0rQRLGOOELs2a_9PIk3vMNALzxxX2r88bDJcZ6f0K5v_3oe"  # 실제 Discord Webhook URL
CSV_FILE_PATH = "/home/dli/CO_ver2/co_readings_Beta.csv"  # CSV 파일 저장 경로


웹후크 = https://discord.com/api/webhooks/1313826821787226132/txS4YAXl6tm_5UWQVzSCX0rQRLGOOELs2a_9PIk3vMNALzxxX2r88bDJcZ6f0K5v_3oe



<gradio 포함 version>
import serial
import time
import requests
from datetime import datetime
import csv
import os
import numpy as np
import openai
import gradio as gr

# ===== 사용자 환경에 맞게 수정해야 하는 부분 =====
PORT = "/dev/ttyACM0"  # Jetson Nano에서 Arduino 포트 확인 (예: /dev/ttyACM0)
WEBHOOK_URL = "YOUR_DISCORD_WEBHOOK_URL"  # 실제 Discord Webhook URL
CSV_FILE_PATH = "/home/your_username/data/co_readings.csv"  # CSV 파일 저장 경로
OPENAI_API_KEY = "YOUR_OPENAI_API_KEY"
openai.api_key = OPENAI_API_KEY
# ==============================================

# 상태 기준
# 정상: ppm < 200
# 주의: 200 ≤ ppm < 800
# 위험: 800 ≤ ppm < 3200
# 매우 위험: ppm ≥ 3200

def init_csv_file(filename):
    dir_path = os.path.dirname(filename)
    if dir_path and not os.path.exists(dir_path):
        os.makedirs(dir_path, exist_ok=True)
    # 파일이 없을 경우 헤더 기록
    if not os.path.exists(filename):
        with open(filename, mode='w', newline='', encoding='utf-8') as f:
            writer = csv.writer(f)
            writer.writerow(["timestamp", "ppm"])
        print(f"CSV 파일 생성 및 헤더 작성 완료: {filename}")

def append_csv_file(filename, timestamp_str, ppm_value):
    with open(filename, mode='a', newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        writer.writerow([timestamp_str, ppm_value])

def get_co_status(ppm):
    if ppm < 200:
        return "정상", 0x00FF00  # Green
    elif ppm < 800:
        return "주의", 0xFFFF00  # Yellow
    elif ppm < 3200:
        return "위험", 0xFFA500  # Orange
    else:
        return "매우 위험", 0xFF0000  # Red

def send_discord_alert(ppm):
    """CO 농도별 단계 알림 전송"""
    current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    status, color = get_co_status(ppm)
    description = f"현재 CO 농도는 {ppm:.2f} ppm 입니다."
    title = f"⚠️ CO {status} 상태 ⚠️"
    
    data = {
        "content": "@here",
        "embeds": [{
            "title": title,
            "description": description,
            "color": color,
            "fields": [
                {"name": "상태", "value": status, "inline": True},
                {"name": "측정 시간", "value": current_time, "inline": False}
            ],
            "footer": {"text": "CO 모니터링 시스템"}
        }]
    }

    try:
        response = requests.post(WEBHOOK_URL, json=data)
        if response.status_code == 204:
            print(f"[{current_time}] 디스코드 알림 전송 성공: {ppm:.2f} ppm ({status})")
        else:
            print(f"디스코드 알림 전송 실패: {response.status_code}")
    except Exception as e:
        print(f"디스코드 알림 오류: {str(e)}")

# 최근 1분간 데이터 저장용 (3초 간격 -> 20개 데이터)
ppm_buffer = []

def add_ppm_data(ppm):
    ppm_buffer.append((datetime.now(), ppm))
    # 1분보다 오래된 데이터 제거
    cutoff = datetime.now() - timedelta(seconds=60)
    # timedelta import를 위해 위 코드 상단에 from datetime import datetime, timedelta 추가해야 함
    while ppm_buffer and ppm_buffer[0][0] < cutoff:
        ppm_buffer.pop(0)

def predict_time_to_reach_threshold(threshold):
    # 최근 1분간 데이터 사용
    # ppm_buffer: [(time, ppm), ...]
    if len(ppm_buffer) < 2:
        return "데이터가 충분하지 않아 예측 불가합니다."

    # 시간축을 초 단위로 변환
    base_time = ppm_buffer[0][0]
    times = np.array([(t[0]-base_time).total_seconds() for t in ppm_buffer])
    ppms = np.array([t[1] for t in ppm_buffer])

    # 선형 회귀
    # y = m*x + c
    # np.polyfit(x, y, 1) -> (m, c)
    m, c = np.polyfit(times, ppms, 1)

    # 현재 마지막 값 기준으로 앞으로도 m(기울기)로 증가한다고 가정
    # threshold = m*x + c -> x = (threshold - c)/m
    if m <= 0:
        return "CO 농도가 증가하는 추세가 아닙니다. 환기가 급하지 않을 수 있습니다."

    x_threshold = (threshold - c)/m
    current_time_sec = (ppm_buffer[-1][0]-base_time).total_seconds()

    if x_threshold <= current_time_sec:
        return "이미 해당 임계값에 도달한 상태로 보입니다."

    delta = x_threshold - current_time_sec
    return f"{int(delta)}초 후에 CO 농도가 {threshold}ppm에 도달할 것으로 예상됩니다."

# OpenAI Function 정의
functions = [
    {
        "name": "predict_ventilation_time",
        "description": "지정한 임계값(ppm)에 도달하는데 걸리는 예상 시간을 반환",
        "parameters": {
            "type": "object",
            "properties": {
                "threshold": {
                    "type": "number",
                    "description": "CO 임계값(ppm)"
                }
            },
            "required": ["threshold"]
        }
    }
]

def openai_chat(query):
    messages = [
        {"role": "system", "content": "당신은 CO 농도 예측 전문가입니다. 사용자 질문에 대해 필요하면 함수를 호출하여 답해주세요."},
        {"role": "user", "content": query}
    ]
    response = openai.ChatCompletion.create(
        model="gpt-4-0613",
        messages=messages,
        functions=functions,
        function_call="auto"
    )

    if response.choices[0].message.get("function_call"):
        func_name = response.choices[0].message["function_call"]["name"]
        func_args = response.choices[0].message["function_call"]["arguments"]
        threshold = float(func_args["threshold"])
        # 함수 결과
        result = predict_time_to_reach_threshold(threshold)
        # 함수 결과를 사용자에게 반환
        return result
    else:
        # 함수 호출 없이 직접 답변한 경우
        return response.choices[0].message["content"]

def gradio_interface(query):
    answer = openai_chat(query)
    return answer

# Gradio 인터페이스
iface = gr.Interface(fn=gradio_interface,
                     inputs="text",
                     outputs="text",
                     title="CO 모니터링 & 예측 챗봇",
                     description="CO 농도 상태 및 환기가 필요한 시간 예측")

# 메인 로직 (CO 측정)
if __name__ == "__main__":
    from datetime import timedelta
    init_csv_file(CSV_FILE_PATH)

    try:
        ser = serial.Serial(PORT, 9600, timeout=1)
        print(f"Arduino 연결됨: {PORT}")
        time.sleep(2)  # 시리얼 초기화 대기 시간

        # Gradio 인터페이스 별도 스레드에서 실행
        import threading
        server_thread = threading.Thread(target=iface.launch, kwargs={"server_name":"0.0.0.0","server_port":7860}, daemon=True)
        server_thread.start()

        while True:
            if ser.in_waiting > 0:
                line = ser.readline().decode('utf-8').strip()
                try:
                    ppm = float(line)
                    current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                    append_csv_file(CSV_FILE_PATH, current_time, ppm)
                    print(f"CO 농도: {ppm:.2f} ppm")

                    add_ppm_data(ppm)

                    status, _ = get_co_status(ppm)
                    if status in ["주의", "위험", "매우 위험"]:
                        send_discord_alert(ppm)

                except ValueError:
                    print(f"잘못된 데이터 수신: {line}")

            # 측정 주기 3초
            time.sleep(3)

    except serial.SerialException as e:
        print(f"시리얼 연결 오류: {str(e)}")
    except KeyboardInterrupt:
        print("프로그램 종료 요청 받음")
    finally:
        if 'ser' in locals() and ser.is_open:
            ser.close()
            print("시리얼 포트 닫힘")




Traceback (most recent call last):
  File "/home/dli/myenv/lib/python3.8/site-packages/gradio/queueing.py", line 536, in process_events
    response = await route_utils.call_process_api(
  File "/home/dli/myenv/lib/python3.8/site-packages/gradio/route_utils.py", line 322, in call_process_api
    output = await app.get_blocks().process_api(
  File "/home/dli/myenv/lib/python3.8/site-packages/gradio/blocks.py", line 1935, in process_api
    result = await self.call_function(
  File "/home/dli/myenv/lib/python3.8/site-packages/gradio/blocks.py", line 1520, in call_function
    prediction = await anyio.to_thread.run_sync(  # type: ignore
  File "/home/dli/myenv/lib/python3.8/site-packages/anyio/to_thread.py", line 56, in run_sync
    return await get_async_backend().run_sync_in_worker_thread(
  File "/home/dli/myenv/lib/python3.8/site-packages/anyio/_backends/_asyncio.py", line 2364, in run_sync_in_worker_thread
    return await future
  File "/home/dli/myenv/lib/python3.8/site-packages/anyio/_backends/_asyncio.py", line 864, in run
    result = context.run(func, *args)
  File "/home/dli/myenv/lib/python3.8/site-packages/gradio/utils.py", line 826, in wrapper
    response = f(*args, **kwargs)
  File "co_monitor.py", line 165, in gradio_interface
    answer = openai_chat(query)
  File "co_monitor.py", line 145, in openai_chat
    response = openai.ChatCompletion.create(
  File "/home/dli/myenv/lib/python3.8/site-packages/openai/lib/_old_api.py", line 39, in __call__
    raise APIRemovedInV1(symbol=self._symbol)
openai.lib._old_api.APIRemovedInV1: 

You tried to access openai.ChatCompletion, but this is no longer supported in openai>=1.0.0 - see the README at https://github.com/openai/openai-python for the API.

You can run `openai migrate` to automatically upgrade your codebase to use the 1.0.0 interface. 

Alternatively, you can pin your installation to the old version, e.g. `pip install openai==0.28`

A detailed migration guide is available here: https://github.com/openai/openai-python/discussions/742

)



# CON-CO_warning_system


# 아두이노 IDE
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
# 디스코드까지만 포함

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


-------------------------------------------------------------------
# 오류가 있던 graido 포함, 답변이 된 version

import serial
import time
import requests
from datetime import datetime
import csv
import os
import numpy as np
from openai import OpenAI
import gradio as gr
import json

# ===== 사용자 환경에 맞게 수정해야 하는 부분 =====
PORT = "/dev/ttyUSB0"  # Jetson Nano에서 Arduino 포트 확인 (예: /dev/ttyACM0)
WEBHOOK_URL = "https://discord.com/api/webhooks/1313826821787226132/txS4YAXl6tm_5UWQVzSCX0rQRLGOOELs2a_9PIk3vMNALzxxX2r88bDJcZ6f0K5v_3oe"  # 실제 Discord Webhook URL
CSV_FILE_PATH = "/home/dli/CO_ver2/co_readings_gradio.csv"  # CSV 파일 저장 경로

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
    if ppm < 0.6:
        return "정상", 0x00FF00  # Green
    elif ppm < 5:
        return "주의", 0xFFFF00  # Yellow
    elif ppm < 20:
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

# 최근 1분간 데이터 저장용 (1초 간격 -> 60개 데이터)
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
    os.environ['OPENAI_API_KEY'] = ''
    OpenAI.api_key = os.getenv("OPENAI_API_KEY")

    client = OpenAI()	
    response = client.chat.completions.create(
        model="gpt-4",
        messages=messages,
        functions=functions,
        function_call="auto"
    )

    proc_messages = []
    response_message = response.choices[0].message
    tool_calls = response_message.tool_calls

    if tool_calls:
        available_functions = {
            "predict_time_to_reach_threshold": predict_time_to_reach_threshold
        }

    # assistant의 reply를 대화 히스토리에 추가
        messages.append(response_message)

        for tool_call in tool_calls:
            function_name = tool_call.function.name
            function_to_call = available_functions.get(function_name)

            if not function_to_call:
            # 해당 함수가 없는 경우 처리
                proc_messages.append({
                    "role": "assistant",
                    "content": f"요청한 함수({function_name})를 찾을 수 없습니다."
                })
                continue

            function_args = json.loads(tool_call.function.arguments)

        # 함수 호출
        # 예: predict_time_to_reach_threshold(threshold=...)
            function_response = function_to_call(**function_args)

        # 함수 응답을 메시지에 추가
            proc_messages.append(
                {
                    "tool_call_id": tool_call.id,
                    "role": "tool",
                    "name": function_name,
                    "content": function_response,
                }
            )

    # 함수 응답 메시지를 전체 대화에 추가
        messages += proc_messages

    # 함수 호출 후 다시 모델에 요청해 최종 답변 받기
        second_response = client.chat.completions.create(
            model='gpt-4o-mini',
            messages=messages,
        )

        assistant_message = second_response.choices[0].message.content
        return assistant_message

    else:
    # 함수 호출 없이 직접 답변한 경우
        return response_message.content


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
        time.sleep(1)  # 시리얼 초기화 대기 시간

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

            # 측정 주기 1초
            time.sleep(1)

    except serial.SerialException as e:
        print(f"시리얼 연결 오류: {str(e)}")
    except KeyboardInterrupt:
        print("프로그램 종료 요청 받음")
    finally:
        if 'ser' in locals() and ser.is_open:
            ser.close()


# 최종 version

{
 "cells": [
  {
   "cell_type": "code",
   "execution_count": 21,
   "id": "6145ff5e-ac37-41e6-8387-25d415223836",
   "metadata": {},
   "outputs": [],
   "source": [
    "import serial\n",
    "import time\n",
    "import requests\n",
    "from datetime import datetime\n",
    "import csv\n",
    "import os\n",
    "import numpy as np\n",
    "from openai import OpenAI\n",
    "import gradio as gr\n",
    "import json\n",
    "\n",
    "# ===== 사용자 환경에 맞게 수정해야 하는 부분 =====\n",
    "PORT = \"/dev/ttyUSB0\"  # Jetson Nano에서 Arduino 포트 확인 (예: /dev/ttyACM0)\n",
    "WEBHOOK_URL = \"https://discord.com/api/webhooks/1313826821787226132/txS4YAXl6tm_5UWQVzSCX0rQRLGOOELs2a_9PIk3vMNALzxxX2r88bDJcZ6f0K5v_3oe\"  # 실제 Discord Webhook URL\n",
    "CSV_FILE_PATH = \"/home/dli/CO_ver2/co_readings_gradio.csv\"  # CSV 파일 저장 경로\n",
    "os.environ['OPENAI_API_KEY'] = '',
    "OpenAI.api_key = os.getenv(\"OPENAI_API_KEY\")\n",
    "# =============================================="
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 22,
   "id": "e079cac9-9d40-4a19-ad85-f613d6c5710a",
   "metadata": {},
   "outputs": [],
   "source": [
    "# 상태 기준\n",
    "# 정상: ppm < 200\n",
    "# 주의: 200 ≤ ppm < 800\n",
    "# 위험: 800 ≤ ppm < 3200\n",
    "# 매우 위험: ppm ≥ 3200\n",
    "\n",
    "def init_csv_file(filename):\n",
    "    dir_path = os.path.dirname(filename)\n",
    "    if dir_path and not os.path.exists(dir_path):\n",
    "        os.makedirs(dir_path, exist_ok=True)\n",
    "    # 파일이 없을 경우 헤더 기록\n",
    "    if not os.path.exists(filename):\n",
    "        with open(filename, mode='w', newline='', encoding='utf-8') as f:\n",
    "            writer = csv.writer(f)\n",
    "            writer.writerow([\"timestamp\", \"ppm\"])\n",
    "        print(f\"CSV 파일 생성 및 헤더 작성 완료: {filename}\")\n",
    "\n",
    "def append_csv_file(filename, timestamp_str, ppm_value):\n",
    "    with open(filename, mode='a', newline='', encoding='utf-8') as f:\n",
    "        writer = csv.writer(f)\n",
    "        writer.writerow([timestamp_str, ppm_value])\n",
    "\n",
    "def get_co_status(ppm):\n",
    "    if ppm < 0.6:\n",
    "        return \"정상\", 0x00FF00  # Green\n",
    "    elif ppm < 5:\n",
    "        return \"주의\", 0xFFFF00  # Yellow\n",
    "    elif ppm < 20:\n",
    "        return \"위험\", 0xFFA500  # Orange\n",
    "    else:\n",
    "        return \"매우 위험\", 0xFF0000  # Red"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 23,
   "id": "9d821488-1bf6-48f3-bebf-27bd396a296a",
   "metadata": {},
   "outputs": [],
   "source": [
    "def send_discord_alert(ppm):\n",
    "    \"\"\"CO 농도별 단계 알림 전송\"\"\"\n",
    "    current_time = datetime.now().strftime(\"%Y-%m-%d %H:%M:%S\")\n",
    "    status, color = get_co_status(ppm)\n",
    "    description = f\"현재 CO 농도는 {ppm:.2f} ppm 입니다.\"\n",
    "    title = f\"⚠️ CO {status} 상태 ⚠️\"\n",
    "    \n",
    "    data = {\n",
    "        \"content\": \"@here\",\n",
    "        \"embeds\": [{\n",
    "            \"title\": title,\n",
    "            \"description\": description,\n",
    "            \"color\": color,\n",
    "            \"fields\": [\n",
    "                {\"name\": \"상태\", \"value\": status, \"inline\": True},\n",
    "                {\"name\": \"측정 시간\", \"value\": current_time, \"inline\": False}\n",
    "            ],\n",
    "            \"footer\": {\"text\": \"CO 모니터링 시스템\"}\n",
    "        }]\n",
    "    }\n",
    "\n",
    "    try:\n",
    "        response = requests.post(WEBHOOK_URL, json=data)\n",
    "        if response.status_code == 204:\n",
    "            print(f\"[{current_time}] 디스코드 알림 전송 성공: {ppm:.2f} ppm ({status})\")\n",
    "        else:\n",
    "            print(f\"디스코드 알림 전송 실패: {response.status_code}\")\n",
    "    except Exception as e:\n",
    "        print(f\"디스코드 알림 오류: {str(e)}\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 24,
   "id": "f1f6018c-4389-40fc-ac44-770b6b7bc238",
   "metadata": {},
   "outputs": [],
   "source": [
    "# 최근 1분간 데이터 저장용 (1초 간격 -> 60개 데이터)\n",
    "ppm_buffer = []\n",
    "\n",
    "def add_ppm_data(ppm):\n",
    "    ppm_buffer.append((datetime.now(), ppm))\n",
    "    # 1분보다 오래된 데이터 제거\n",
    "    cutoff = datetime.now() - timedelta(seconds=60)\n",
    "    # timedelta import를 위해 위 코드 상단에 from datetime import datetime, timedelta 추가해야 함\n",
    "    while ppm_buffer and ppm_buffer[0][0] < cutoff:\n",
    "        ppm_buffer.pop(0)\n",
    "\n",
    "def predict_time_to_reach_threshold(threshold):\n",
    "    # 최근 1분간 데이터 사용\n",
    "    # ppm_buffer: [(time, ppm), ...]\n",
    "    if len(ppm_buffer) < 2:\n",
    "        return \"데이터가 충분하지 않아 예측 불가합니다.\"\n",
    "\n",
    "    # 시간축을 초 단위로 변환\n",
    "    base_time = ppm_buffer[0][0]\n",
    "    times = np.array([(t[0]-base_time).total_seconds() for t in ppm_buffer])\n",
    "    ppms = np.array([t[1] for t in ppm_buffer])\n",
    "\n",
    "    # 선형 회귀\n",
    "    # y = m*x + c\n",
    "    # np.polyfit(x, y, 1) -> (m, c)\n",
    "    m, c = np.polyfit(times, ppms, 1)\n",
    "\n",
    "    # 현재 마지막 값 기준으로 앞으로도 m(기울기)로 증가한다고 가정\n",
    "    # threshold = m*x + c -> x = (threshold - c)/m\n",
    "    if m <= 0:\n",
    "        return \"CO 농도가 증가하는 추세가 아닙니다. 환기가 급하지 않을 수 있습니다.\"\n",
    "\n",
    "    x_threshold = (threshold - c)/m\n",
    "    current_time_sec = (ppm_buffer[-1][0]-base_time).total_seconds()\n",
    "\n",
    "    if x_threshold <= current_time_sec:\n",
    "        return \"이미 해당 임계값에 도달한 상태로 보입니다.\"\n",
    "\n",
    "    delta = x_threshold - current_time_sec\n",
    "    hours = delta // 3600  \n",
    "    minutes = (delta % 3600) // 60  \n",
    "    seconds = delta % 60  \n",
    "    return f\"{int(hours)}시간 {int(minutes)}분 {int(seconds)}초 후에, 즉 CO 농도가 {threshold}ppm에 도달할 것으로 예상됩니다.\""
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 25,
   "id": "58c49219-4df7-4c8c-b8bc-6bf1a63f5884",
   "metadata": {},
   "outputs": [],
   "source": [
    "use_functions = [\n",
    "    {\n",
    "        \"type\": \"function\",\n",
    "        \"function\": {\n",
    "        \"name\": \"predict_time_to_reach_threshold\",\n",
    "        \"description\": \"지정한 임계값(ppm)에 도달하는데 걸리는 예상 시간을 반환\",\n",
    "        \"parameters\": {\n",
    "            \"type\": \"object\",\n",
    "            \"properties\": {\n",
    "                \"threshold\": {\n",
    "                    \"type\": \"number\",\n",
    "                    \"description\": \"CO 임계값(ppm)\"\n",
    "                }\n",
    "            },\n",
    "            \"required\": [\"threshold\"]\n",
    "            }\n",
    "        }\n",
    "    },\n",
    "    {\n",
    "        \"type\": \"function\",\n",
    "        \"function\": {\n",
    "        \"name\": \"add_ppm_data\",\n",
    "        \"description\": \"1분간 CO 데이터를 모아주는 함\",\n",
    "        \"parameters\": {\n",
    "            \"type\": \"object\",\n",
    "            \"properties\": {\n",
    "                \"ppm\": {\n",
    "                    \"type\": \"number\",\n",
    "                    \"description\": \"CO 측정\"\n",
    "                }\n",
    "            },\n",
    "            \"required\": [\"ppm\"]\n",
    "            }\n",
    "        }\n",
    "    }\n",
    "]"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 26,
   "id": "135c9e3e-3640-446e-8890-c6307900aa72",
   "metadata": {},
   "outputs": [],
   "source": [
    "def ask_openai(llm_model, messages, user_message, functions = ''):\n",
    "    client = OpenAI()\n",
    "    proc_messages = messages\n",
    "\n",
    "    if user_message != '':\n",
    "        proc_messages.append({\"role\": \"user\", \"content\": user_message})\n",
    "\n",
    "    if functions == '':\n",
    "        response = client.chat.completions.create(model=llm_model, messages=proc_messages, temperature = 1.0)\n",
    "    else:\n",
    "        response = client.chat.completions.create(model=llm_model, messages=proc_messages, tools=use_functions, tool_choice=\"auto\") # 이전 코드와 바뀐 부분\n",
    "\n",
    "    response_message = response.choices[0].message\n",
    "    tool_calls = response_message.tool_calls\n",
    "\n",
    "    if tool_calls:\n",
    "        # Step 3: call the function\n",
    "        # Note: the JSON response may not always be valid; be sure to handle errors\n",
    "\n",
    "        available_functions = {\n",
    "            # \"add_ppm_data\": add_ppm_data,\t\t\n",
    "            \"predict_time_to_reach_threshold\": predict_time_to_reach_threshold\n",
    "        }\n",
    "\n",
    "        messages.append(response_message)  # extend conversation with assistant's reply\n",
    "        print(response_message)\n",
    "        # Step 4: send the info for each function call and function response to GPT\n",
    "        for tool_call in tool_calls:\n",
    "            function_name = tool_call.function.name\n",
    "            function_to_call = available_functions[function_name]\n",
    "            function_args = json.loads(tool_call.function.arguments)\n",
    "            print(function_args)\n",
    "\n",
    "            if 'user_prompt' in function_args:\n",
    "                function_response = function_to_call(function_args.get('user_prompt'))\n",
    "            else:\n",
    "                function_response = function_to_call(**function_args)\n",
    "\n",
    "            proc_messages.append(\n",
    "                {\n",
    "                    \"tool_call_id\": tool_call.id,\n",
    "                    \"role\": \"tool\",\n",
    "                    \"name\": function_name,\n",
    "                    \"content\": function_response,\n",
    "                }\n",
    "            )  # extend conversation with function response\n",
    "        second_response = client.chat.completions.create(\n",
    "            model=llm_model,\n",
    "            messages=messages,\n",
    "        )  # get a new response from GPT where it can see the function response\n",
    "\n",
    "        assistant_message = second_response.choices[0].message.content\n",
    "    else:\n",
    "        assistant_message = response_message.content\n",
    "\n",
    "    text = assistant_message.replace('\\n', ' ').replace(' .', '.').strip()\n",
    "\n",
    "\n",
    "    proc_messages.append({\"role\": \"assistant\", \"content\": assistant_message})\n",
    "\n",
    "    return text # proc_messages, "
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 27,
   "id": "5b355555-091e-428e-bb46-cede58dd917e",
   "metadata": {},
   "outputs": [],
   "source": [
    "import gradio as gr\n",
    "\n",
    "def gradio_interface(user_message):\n",
    "    messages = [\n",
    "    {\"role\": \"system\", \"content\": \"당신은 CO 농도 예측 전문가입니다. 사용자 질문에 대해 필요하면 함수를 호출하여 답해주세요.\"},\n",
    "    {\"role\": \"user\", \"content\": \"임계값은 10입니다. 환기가 언제 필요할까요?\"}\n",
    "    ]\n",
    "    answer = ask_openai(\"gpt-4o-mini\", messages, user_message, functions= use_functions)\n",
    "    return answer\n",
    "\n",
    "\n",
    "\n",
    "# Gradio 인터페이스\n",
    "iface = gr.Interface(fn=gradio_interface,\n",
    "                     inputs=\"text\",\n",
    "                     outputs=\"text\",\n",
    "                     title=\"CO 모니터링 & 예측 챗봇\",\n",
    "                     description=\"CO 농도 상태 및 환기가 필요한 시간 예측\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 28,
   "id": "c51e64aa-a2a3-4e28-aef7-4f57e11a336b",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Arduino 연결됨: /dev/ttyUSB0\n",
      "잘못된 데이터 수신: \u0000\n",
      "Running on local URL:  http://127.0.0.1:7862\n",
      "\n",
      "To create a public link, set `share=True` in `launch()`.\n"
     ]
    },
    {
     "data": {
      "text/html": [
       "<div><iframe src=\"http://127.0.0.1:7862/\" width=\"100%\" height=\"500\" allow=\"autoplay; camera; microphone; clipboard-read; clipboard-write;\" frameborder=\"0\" allowfullscreen></iframe></div>"
      ],
      "text/plain": [
       "<IPython.core.display.HTML object>"
      ]
     },
     "metadata": {},
     "output_type": "display_data"
    },
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "ChatCompletionMessage(content=None, refusal=None, role='assistant', audio=None, function_call=None, tool_calls=[ChatCompletionMessageToolCall(id='call_Mj0sp7ALXtfWrtdDJUTphxkS', function=Function(arguments='{\"threshold\":10}', name='predict_time_to_reach_threshold'), type='function')])\n",
      "{'threshold': 10}\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 1.40 ppm\n",
      "[2024-12-13 01:07:19] 디스코드 알림 전송 성공: 1.40 ppm (주의)\n",
      "CO 농도: 3.25 ppm\n",
      "[2024-12-13 01:07:21] 디스코드 알림 전송 성공: 3.25 ppm (주의)\n",
      "CO 농도: 4.72 ppm\n",
      "[2024-12-13 01:07:22] 디스코드 알림 전송 성공: 4.72 ppm (주의)\n",
      "CO 농도: 2.46 ppm\n",
      "[2024-12-13 01:07:24] 디스코드 알림 전송 성공: 2.46 ppm (주의)\n",
      "CO 농도: 1.40 ppm\n",
      "[2024-12-13 01:07:25] 디스코드 알림 전송 성공: 1.40 ppm (주의)\n",
      "CO 농도: 0.87 ppm\n",
      "[2024-12-13 01:07:27] 디스코드 알림 전송 성공: 0.87 ppm (주의)\n",
      "CO 농도: 0.61 ppm\n",
      "[2024-12-13 01:07:28] 디스코드 알림 전송 성공: 0.61 ppm (주의)\n",
      "CO 농도: 0.46 ppm\n",
      "CO 농도: 0.39 ppm\n",
      "CO 농도: 0.39 ppm\n",
      "CO 농도: 0.33 ppm\n",
      "CO 농도: 0.33 ppm\n",
      "CO 농도: 0.33 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "ChatCompletionMessage(content=None, refusal=None, role='assistant', audio=None, function_call=None, tool_calls=[ChatCompletionMessageToolCall(id='call_QrXpL1VbKpqnrn7pbMdEWrjz', function=Function(arguments='{\"threshold\":100}', name='predict_time_to_reach_threshold'), type='function')])\n",
      "{'threshold': 100}\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.27 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "ChatCompletionMessage(content=None, refusal=None, role='assistant', audio=None, function_call=None, tool_calls=[ChatCompletionMessageToolCall(id='call_yGHdlTN6qB8n2GUToftEhCHt', function=Function(arguments='{\"threshold\": 10}', name='predict_time_to_reach_threshold'), type='function'), ChatCompletionMessageToolCall(id='call_20pLrJjGZKnfoV1smVHPnLET', function=Function(arguments='{\"threshold\": 50}', name='predict_time_to_reach_threshold'), type='function')])\n",
      "{'threshold': 10}\n",
      "{'threshold': 50}\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.21 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "CO 농도: 0.16 ppm\n",
      "프로그램 종료 요청 받음\n"
     ]
    }
   ],
   "source": [
    "from datetime import timedelta\n",
    "init_csv_file(CSV_FILE_PATH)\n",
    "\n",
    "try:\n",
    "    ser = serial.Serial(PORT, 9600, timeout=1)\n",
    "    print(f\"Arduino 연결됨: {PORT}\")\n",
    "    time.sleep(1)  # 시리얼 초기화 대기 시간\n",
    "\n",
    "    # Gradio 인터페이스 별도 스레드에서 실행\n",
    "    import threading\n",
    "    server_thread = threading.Thread(target=iface.launch, kwargs={\"debug\":True}, daemon=True)\n",
    "    server_thread.start()\n",
    "\n",
    "    while True:\n",
    "        if ser.in_waiting > 0:\n",
    "            line = ser.readline().decode('utf-8').strip()\n",
    "            try:\n",
    "                ppm = float(line)\n",
    "                current_time = datetime.now().strftime(\"%Y-%m-%d %H:%M:%S\")\n",
    "                append_csv_file(CSV_FILE_PATH, current_time, ppm)\n",
    "                print(f\"CO 농도: {ppm:.2f} ppm\")\n",
    "\n",
    "                add_ppm_data(ppm)\n",
    "\n",
    "                status, _ = get_co_status(ppm)\n",
    "                if status in [\"주의\", \"위험\", \"매우 위험\"]:\n",
    "                    send_discord_alert(ppm)\n",
    "\n",
    "            except ValueError:\n",
    "                print(f\"잘못된 데이터 수신: {line}\")\n",
    "\n",
    "        # 측정 주기 1초\n",
    "        time.sleep(1)\n",
    "\n",
    "except serial.SerialException as e:\n",
    "    print(f\"시리얼 연결 오류: {str(e)}\")\n",
    "except KeyboardInterrupt:\n",
    "    print(\"프로그램 종료 요청 받음\")\n",
    "finally:\n",
    "    if 'ser' in locals() and ser.is_open:\n",
    "        ser.close()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "6539a0d8-76af-4b39-be96-9276deb13ff5",
   "metadata": {},
   "outputs": [],
   "source": []
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3 (ipykernel)",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.8.12"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}


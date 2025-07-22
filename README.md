# Access_Control_System
얼굴 인식 출입 관리 시스템

### 개요
얼굴 인식 기술을 사용하여 등록된 사용자만 출입할 수 있도록 하는 시스템입니다. Raspberry Pi와 웹캠을 활용하여 얼굴을 인식하고, 등록된 사용자가 얼굴을 인식하면 문을 열 수 있도록 합니다. MQTT, MySQL DB를 사용하여 사용자 정보를 저장하고 관리하며, 윈도우 애플리케이션에서 실시간으로 얼굴 인식 및 출입 상태를 확인할 수 있는 UI를 구현할 계획입니다.

### 목표 및 구현 계획
1. 얼굴 인식 시스템 구현
    - 얼굴 인식 라이브러리인 face_recognition을 사용하여 등록된 얼굴을 인식하고 비교합니다.
    - 웹캠에서 실시간으로 얼굴을 인식하고, 등록된 사용자일 경우 문을 열고, 그렇지 않으면 경고 메시지를 출력합니다.
    - 모터를 사용하여 문을 열고, 일정 시간이 지나면 자동으로 닫히도록 설정합니다.

2. 윈도우 애플리케이션 개발
    - 얼굴 인식 웹캠 화면을 C# 윈도우 애플리케이션에서 띄우고 제어할 수 있도록 합니다.

3. 사용자 정보 저장 및 관리
    - MQTT를 사용하여 출입 기록, 도어락 상태, 경고 메시지등의 데이터를 실시간으로 전달하고 수신할 수 있습니다.
    - 사용자 정보(이름, 등록된 얼굴 이미지 등)를 DB에 저장하여 윈앱에서 이를 조회하거나 수정할 수 있도록 합니다.

4. 추가 기능
    - 얼굴 인식 정확도를 높이기 위한 학습 모델 개선 및 데이터 증강
    - 알림 기능 추가 (사용자 출입 시 푸시 알림)
    - 보안 강화를 위한 2차 인증 (얼굴 인식 + RFID 출입증)

### 개발 환경
```bash
# 가상환경 생성
python3 -m venv venv
source venv/bin/activate

# OpenCV 설치
pip install opencv-python

# dlib 설치
sudo apt install -y cmake libopenblas-dev liblapack-dev libx11-dev libgtk-3-dev libboost-python-dev
pip install dlib

# 얼굴인식 라이브러리
pip install face_recognition

# 카메라 사용에 필요한 패키지
sudo apt install -y libcamera-dev v4l-utils
pip install imutils
```

>**dlib?**
>C++ 기반의 머신러닝 알고리즘 및 도구 모음 라이브러리. 파이썬에서 사용할 수 있도록 Python 바인딩도 제공됨.
>- 얼굴 탐지 (face detection)
>- 얼굴 랜드마크 추출 (68 or 5 points)
>- 얼굴 인식 (face recognition)
>- 객체 추적 (object tracking)
>- SVM/ML 알고리즘
>- 이미지 전처리/변환


### 코드 설명
**1. 라이브러리**
```python
import cv2
import face_recognition
import os
import time
from gpiozero import DigitalOutputDevice
from PIL import ImageFont, ImageDraw, Image
import numpy as np
```
- OpenCV: 실시간 영상 처리 및 카메라 제어를 위해 사용
- face_recognition: 얼굴 인식을 위한 라이브러리
- gpiozero: Raspberry Pi의 GPIO 핀을 제어하여 도어락을 제어
- PIL: 텍스트 출력 및 이미지 처리를 위해 사용
- numpy: 얼굴 인식의 유사도를 계산하는 데 사용

**2. 기준 이미지 인코딩/로드 함수**
```python
def load_target_encodings(folder):
    encodings = []
    for filename in os.listdir(folder):
        if filename.lower().endswith((".jpg", ".png")):
            path = os.path.join(folder, filename)
            img = face_recognition.load_image_file(path)
            face_enc = face_recognition.face_encodings(img)
            if face_enc:
                encodings.append(face_enc[0])
            else:
                print(f"얼굴 없음: {filename}")
    if not encodings:
        raise Exception("기준 이미지에서 얼굴을 찾지 못했습니다.")
    return encodings
```
- `face_recognition.face_encodings(img)`: 주어진 이미지에서 얼굴을 인식하고, 128D 임베딩 벡터를 반환합니다.
- 여러 이미지를 로드하여 등록된 사용자 얼굴 정보를 인코딩하고 리스트에 저장합니다.

**128D 임베딩?**


**3. 웹캠 설정**
```python
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
```
- `cv2.VideoCapture(0)`: 웹캠을 열고, 첫 번째 카메라 장치를 사용
- 카메라의 해상도를 640x480으로 설정하여 웹캠에서 프레임을 캡처

- face_recognition.face_encodings(img): 주어진 이미지에서 얼굴을 인식하고, 128D 임베딩 벡터를 반환
- 여러 이미지를 로드하여 등록된 사용자 얼굴 정보를 인코딩하고 리스트에 저장

**3. 얼굴 인식 및 등록 여부 확인**
```python
face_locations = face_recognition.face_locations(rgb_frame)
face_encodings = face_recognition.face_encodings(rgb_frame, face_locations)
```
- 얼굴 위치 및 인코딩: `face_recognition.face_locations`로 얼굴 위치를 찾고, `face_recognition.face_encodings`로 얼굴 특징을 추출하여 등록된 얼굴들과 비교

**4. 도어락 제어**

**5. 메시지 출력**
```python
if is_registered and in_center:
    status_text = "등록자 확인 - 문이 열립니다"
    last_grant_time = current_time
elif not is_registered:
    status_text = "등록되지 않은 사용자입니다"
    status_color = (0, 0, 255)
elif not in_center:
    status_text = "얼굴을 가운데로 이동하세요"
    status_color = (255, 140, 0)
```
- 알림 출력: 사용자가 등록된 경우와 등록되지 않은 경우 각각에 대한 알림을 출력하고, 화면에 상태 메시지를 표시

**6. 시스템 종료**
```python
cap.release()
cv2.destroyAllWindows()
```
- 시스템 종료: 프로그램이 종료될 때 웹캠을 해제하고, OpenCV 창을 닫음
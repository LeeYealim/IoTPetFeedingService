# IoT 원격 사료 배식 서비스

**IoT 원격 사료 배식 서비스** 는 IoT사료배식기 디바이스와 AWS를 연동하여 웹에서 원격으로 사료 배식을 제어하고, 배식기에서 측정된 사료 잔량을 모니터링하는 서비스입니다. 또한 주변환경에 민감한 반려동물을 위하여 주변 온습도 값을 모니터링할 수 있습니다.


## 1. Web 프로젝트

### IoTFeedingServiceWebProject
- **login.html**
	- 관리자 로그인
	- 로그인 시, homepage.html Page로 이동
- **homepage.html**
	- 사용자정보 조회 버튼 클릭 시, 사용자 정보 확인 모달 열림
	- 기록 조회 버튼 클릭 시, menu.html Page로 이동
	- 상태확인 및 사료주기 버튼 클릭 시, 모달에서 현재 온도, 습도, 잔량을 확인할 수 있으며 사료배식 버튼을 클릭하여 원격으로 배식할 수 있음
- **menu.html**
	- 배식기록 조회하기 버튼 클릭 시, log_feed.html Page로 이동
	- 이슈기록 조회하기 버튼 클릭 시, log_issue.html Page로 이동
	- 온습도기록 조회하기 버튼 클릭 시, log_environment.html Page로 이동
	- 남은 사료잔량 확인하기 버튼 클릭 시, log_quantity.html Page로 이동
- **log_feed.html**
	- 조회 시작일과 조회 종료일 선택 후, 배식 기록 조회
- **log_issue.html**
	- 조회 시작일과 조회 종료일 선택 후, 이슈 기록 조회
- **log_environment.html**
	- 조회 시작일과 조회 종료일 선택 후, 온습도 기록 조회
- **log_quantity.html**
	- 조회 시작일과 조회 종료일 선택 후, 잔량 기록 조회



## 2. Arduino 프로젝트

### IoTFeedingServiceArduinoProject
- 속성
	- MOTOR : 사료 배식 입구 제어를 위한 서보모터 센서 데이터
	- LED : 잔량에 따른 빛 색깔 표시를 위한 네오픽셀 센서 데이터
	- temperature : 온도 수집 데이터
	- humidity : 습도 수집 데이터
	- cm : 배식기 천장에 위치한 초음파 센서를 이용하여 측정한 거리(cm) 데이터 값을 이용하여 잔량 파악
- 제어
	- MOTOR에 "ON" 요청이 들어올 경우 사료 배식기 입구가 열리고 "OFF" 요청이 들어오면 사료 배식기 입구가 닫힘
	- LED에 "RED", "YELLOW", "GREEN" 요청이 들어올 경우 네오픽셀의 불빛을 요청한 색으로 변경



## 3. 기능 구현

### 1. SNS 이메일 알림
- IoT 규칙에 의해 온도, 습도, 잔량에 이슈가 발생하면 알림 이메일 전송
- PetTopit 주제를 생성하고, 해당 주제에 이메일 프로토콜을 사용하는 구독을 생성하여 설정한 이메일 주소로 잔량거리 cm이상일 때 "잔량부족", 온도가 30도 초과일 경우 "온도이상", 습도가 60% 초과일 경우 "습도이상" 알림 이메일을 전송함
	- IoT규칙 : petSNSRule
	- 규칙 쿼리 설명문 : SELECT * FROM '$aws/things/MKR2/shadow/update/accepted'
	- lambda함수 : SNSPetFunction

### 2. 수집 데이터 Amazon DynamoDB 저장
- IoT규칙에 의해 실시간 디바이스 상태 정보를 Amazon DynamoDB에 저장
- 효율적인 DB 저장공간 사용을 위해 이전 값과 다를 경우 저장되도록 설정
- Pet_Environment 테이블에는 시간, 온도, 습도 값을 저장
- Pet_Quantity 테이블에는 시간, 잔량(cm), LED상태(잔량에 따라 LED 색이 다름) 저장
- Pet_Issue 테이블에는 온도 30초과일 경우 “온도 이상”, 습도 60초과일 경우 “습도 이상”, 잔량 10cm 이상일 경우 “잔량 부족”, 잔량 10cm미만 5cm이상일 경우 "잔량 경고"
	- IoT규칙 : petDataRule
	- 규칙 쿼리 설명문 : SELECT *, 'MKR2' as device FROM '$aws/things/MKR2/shadow/update/documents'
	- lambda함수 : RecordingPetDataFunction

### 3. IoT 배식기 입구 원격 제어
- 디바이스 상태변경 API를 이용하여 웹에서 원격으로 배식기 입구 제어
- JavaScript에서 API 호출을 통해 디바이스 입구 "MOTOR"의 상태를 “ON”으로 변경한 후, 3초 뒤에 "OFF"로 변경하여 제어하는 방식으로 사료 배식
	- API : PUT /devices/{device}
	- 매핑템플릿 : 
  	  ```
      #set($inputRoot = $input.path('$'))
      {
          "device": "$input.params('device')",
          "tags" : [
          ##TODO: Update this foreach loop to reference array from input json
              #foreach($elem in $inputRoot.tags)
              {
                  "tagName" : "$elem.tagName",
                  "tagValue" : "$elem.tagValue"
              } 
              #if($foreach.hasNext),#end
              #end
          ]
      }
      ```
	- lambda함수 : UpdateDeviceFunction

### 4. DynamoDB 로그 조회
- DB에 저장된 로그 데이터를 조회하는 API를 호출하여 웹에서 로그 기록을 보여줌
- 쿼리 문자열의 what은 조회하고 싶은 Table 이름, from은 조회 시작일, to는 조회 종료일을 의미
	- API : GET /devices/{device}/log
	- 쿼리 문자열 : what=Pet_Issue&from=2021-11-29%2000:00:00&to=2021-12-10%2018:09:36
	- 매핑템플릿 : 
  	  ```
      {
        "what": "$input.params('what')",
        "from": "$input.params('from')",
        "to":  "$input.params('to')"
      }
      ```
	- lambda함수 : PetLogFunction

### 5. 로그인 기능
- 로그인 API를 이용하여 로그인 기능 구현
- DB에 admin(관리자)계정 하나만 저장되어 있도록 가정하여 구현
- 로그인 API 호출할 때, 쿼리 문자열에 id와 password를 전달
- 람다함수에서 Pet_User 테이블에서 파티션키가 id인 항목의 password 값이 쿼리 문자열로 넘어온 password 값과 일치하면 true를 응답으로 리턴하고 다르면 false를 리턴
	- API : GET devices/{device}/login
	- 쿼리 문자열 : id=admin&password=0000
	- 매핑템플릿 : 
  	  ```
      {
        "id": "$input.params('id')",
        "password": "$input.params('fpassword)"
      }
      ```
	- lambda함수 : LoginPetFunction

### 6. 디바이스 현재 상태 조회
- 디바이스 상태를 가져오는 API를 이용하여 웹에서 실시간 디바이스 상태를 조회할 수 있음
	- API : GET /devices/{deviceId}
	- 쿼리 문자열 : id=admin&password=0000
	- 매핑템플릿 : 
  	  ```
      {
        "device": "$input.params('device')"
      }
      ```
	- lambda함수 : GetDeviceFunction

### 7. 잔량에 따른 LED 불빛 자동 제어
- IoT룰에 의해 lambda함수로 넘어오는 잔량 거리값(cm)에 따라 cm가 10cm 이상이면 “LED”를 “RED”로, 5cm이상 10cm미만이면 "YELLOW"로, 그 이외(5cm 미만)인 경우는 "LED"를 "GREEN"으로 디바이스 상태를 변경함
	- IoT규칙 : petLEDRule
	- 규칙 쿼리 설명문 : SELECT *, 'MKR2' as device FROM '$aws/things/MKR2/shadow/update/accepted'
	- lambda함수 : UpdatePetLEDFunction

### 8. 사용자 정보 조회
- Amazon DynamoDB의 Pet_User Table에 저장된 id가 admin인 항목의 값을 가져오는 API를 호출하여 사용자 정보를 조회할 수 있음
	- API : GET /devices/{deviceId}/user
	- lambda함수 : getPetUserFunction

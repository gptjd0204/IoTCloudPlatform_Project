# IoTCloudPlatform_Project

## AWS를 이용한 IoT 클라우드 플랫폼 프로젝트

### 주제 : AWS를 이용한 지붕 붕괴 예방 서비스

1. Arduino(MKR WIFI 1010)에 연결된 weight sensor을 이용해 지붕의 무게를 감지해 DynamoDB에 값을 Upload

2. Upload된 값과 기상청 API, 건축물API를 통해 위험도를 예측(추후, 데이터가 쌓여서 위험도 예측 모델을 만들 수 있음.) 

3. 붕괴 피해가 예상될 경우, APP을 통해 LED와 BUZZER를 ON으로 변화시켜 주민들의 대피를 유도함.(상황 종료시, OFF)

![c](https://user-images.githubusercontent.com/70942492/101444752-815f8b00-3963-11eb-910a-29919af5f08c.png)

## 1. Arduino MKR WIFI 1010 관련 Library 설치

* WIFININA
* ArduinoBearSSL
* ArduinoECCX08
* ArduinoMqttClient
* Arduino Cloud Provider Examples

## 2. ECCX08SCR예제를 통해 인증서 만들기

1. Arduino 파일 -> 예제 -> ArduinoECCX08 -> Tools -> ECCX08CSR Click!

2. Serial Monitor를 연 후, Common Name: 부분에 SnowProject 입력(나머지 질문들은 입력 없이 전송 누르기) Would you like to generate? 에는 Y 입력!

3. 생성된 CSR을 csr.txt 파일로 만들어 Save!

## 3. AWS IoT Core에서 사물 등록하기

1. 관리 -> 사물 -> 단일 사물 생성 -> 사물 이름은 SnowProject 입력 -> CSR을 통한 생성을 Click -> 2번에서 저장한 csr.txt를 Upload -> 사물 등록

* region은 아시아 태평양(서울) ap-northeast-2로 해줌./ 사물의 정책 AllowEverything(작업 필드 : iot.* 관련) 생성 후 연결해줌.

2. 보안 -> 관리에서 생성된 인증서도 정책(AllowEverything)을 연결 해줌.

3. 생성된 인증서의 …를 Click한 후, 다운로드 선택

4. 다운로드 된 인증서 확인

## 4. Arduino_SnowProject/arduino_secrets.h 

1. #define SECRET_SSID ""에 자신의 Wifi 이름을 적고, #define SECRET_PASS ""에 Wifi의 비밀번호를 적는다.

2. #define SECRET_BROKER "xxxxxxxxxxxxxx.iot.xx-xxxx-x.amazonaws.com"에는 설정에서 확인한 자신의 엔드포인트를 붙여넣기 한다.

3. const char SECRET_CERTIFICATE[] 부분에는, 3에서 다운 받은 인증서 긴 영어들을 복사 붙여넣기 해준다.

* 올바르게 작성 후, 업로드를 하면 Serial Monitor에는 network와 MQTT broker에 connect된 문구가 뜰것이다.

## 5. AWS DynamoDB 테이블 만들기 / Lambda함수 정의 / 규칙 정의

 1. 테이블 만들기 -> 테이블 이름 : SnowData / 파티션 키: deviceId(데이터 유형 : 문자열) 
 
 2. 정렬 키 추카 선택 -> time 입력(데이터 유형 : 번호 선택)
 
 3. Lambda함수 Eclipse용 AWS Toolkit 이용해 생성 & Upload 

> Project name : RecordingDeviceDataJavaProject2

> Group ID: com.example.lambda

> Artifact ID: recording

> Class Name: RecordingDeviceInfoHandler2

> Input Type : Custom

>> RecordingDeviceInfoHandler2.java Code

>> 
```javascript
mport java.text.SimpleDateFormat;
import java.util.TimeZone;

import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
import com.amazonaws.services.dynamodbv2.document.DynamoDB;
import com.amazonaws.services.dynamodbv2.document.Item;
import com.amazonaws.services.dynamodbv2.document.spec.PutItemSpec;
import com.amazonaws.services.dynamodbv2.model.ConditionalCheckFailedException;
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;

public class RecordingDeviceInfoHandler2 implements RequestHandler<Document, String> {
    private DynamoDB dynamoDb;
    private String DYNAMODB_TABLE_NAME = "SnowData";

    @Override
    public String handleRequest(Document input, Context context) {
        this.initDynamoDbClient();
        context.getLogger().log("Input: " + input);

        //return null;
        return persistData(input);
    }

    private String persistData(Document document) throws ConditionalCheckFailedException {

        // Epoch Conversion Code: https://www.epochconverter.com/
        SimpleDateFormat sdf = new SimpleDateFormat ( "yyyy-MM-dd HH:mm:ss");
        sdf.setTimeZone(TimeZone.getTimeZone("Asia/Seoul"));
        String timeString = sdf.format(new java.util.Date (document.timestamp*1000));

        return this.dynamoDb.getTable(DYNAMODB_TABLE_NAME)
                .putItem(new PutItemSpec().withItem(new Item().withPrimaryKey("deviceId", document.device)
                        .withLong("time", document.timestamp)
                        .withString("weight1", document.current.state.reported.weight1)
                        .withString("weight2", document.current.state.reported.weight2)
                        .withString("LED", document.current.state.reported.LED)
                        .withString("BUZZER", document.current.state.reported.BUZZER)
                        .withString("timestamp",timeString)))
                .toString();
    }

    private void initDynamoDbClient() {
        AmazonDynamoDB client = AmazonDynamoDBClientBuilder.standard().withRegion("ap-northeast-2").build();

        this.dynamoDb = new DynamoDB(client);
    }

}

class Document {
    public Thing previous;       
    public Thing current;
    public long timestamp;
    public String device;       // AWS IoT에 등록된 사물 이름 
}

class Thing {
    public State state = new State();
    public long timestamp;
    public String clientToken;

    public class State {
        public Tag reported = new Tag();
        public Tag desired = new Tag();

        public class Tag {
            public String weight1;
            public String weight2;
            public String LED;
            public String BUZZER;
        }
    }
}
```

>> 다음과 같이 수정 후, [Upload function to AWS Lambda] Click! -> 함수 이름 : SnowDataFunction -> dynamoDB정책에 연결되어있는 IAM 역할 선택 -> Upload!

 
 4. AWS IoT Core -> 동작 -> 규칙 -> 이름 : SnowRule인 규칙 생성 
 
 > 규칙 쿼리 설명문 : SELECT *, 'SnowProject' as device FROM '$aws/things/SnowProject/shadow/update/documents' 
 
 -> 작업 추가-> 메시지 데이터를 전달하는 Lambda 함수 호출 선택
 
 > 5-3에서 upload한 SnowDataFunction Lambda함수 선택 
 
 -> 작업 추가 -> 규칙 생성 Click!
 
## 6. API Gateway를 이용한 RestAPI 생성

### 0. CORS 활성화 및 API Gateway 콘솔에서 RESTAPI 배포 (공통)

> 0-1. 리소스 /devices 선택

> 0-2. 작업 드롭다운 메뉴 CORS 활성화(Enable CORS) 선택

> 0-3. CORS 활성화 및 기존의 CORS 헤더 대체 선택

> 0-4. 메서드 변경사항 확인 창에서 예, 기존 값을 대체하겠습니다. 선택

> 0-5. 작업 드롭다운 메뉴 Deploy API(API 배포) 선택

> 0-6. 배포 스테이지 드롭다운 메뉴 prod 선택

> 0-7. 배포 Click!

### 1. 디바이스 목록 조회 REST API 구축

> 1-1. Lambda 함수 생성 

>> Project name : ListingDeviceLambdaJavaProject

>> Class Name: ListingDeviceHandler

>> Input Type : Custom

>>> pom.xml파일에 다음과 같이 추가

>>>
```javascript
 <dependencies>
    ...    
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-java-sdk-iot</artifactId>
    </dependency>

  </dependencies>
```

>>> ListingDeviceHandler.java Code

>>>
```javascript
import java.util.List;
import com.amazonaws.services.iot.AWSIot;
import com.amazonaws.services.iot.AWSIotClientBuilder;
import com.amazonaws.services.iot.model.ListThingsRequest;
import com.amazonaws.services.iot.model.ListThingsResult;
import com.amazonaws.services.iot.model.ThingAttribute;
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;

public class ListingDeviceHandler implements RequestHandler<Object, String> {

    @Override
    public String handleRequest(Object input, Context context) {

        // AWSIot 객체를 얻는다. 
        AWSIot iot = AWSIotClientBuilder.standard().build();

        // ListThingsRequest 객체 설정. 
        ListThingsRequest listThingsRequest = new ListThingsRequest();

        // listThings 메소드 호출하여 결과 얻음. 
        ListThingsResult result = iot.listThings(listThingsRequest);

        // result 객체로부터 API 응답모델 문자열 생성하여 반
        return getResponse(result);
    }

    /**
     * ListThingsResult 객체인 result로 부터 ThingName과 ThingArn을 얻어서 Json문자 형식의
     * 응답모델을 만들어 반환한다.
     * {
     *  "things": [ 
     *       { 
     *          "thingName": "string",
     *          "thingArn": "string"
     *       },
     *       ...
     *     ]
     * }
     */
    private String getResponse(ListThingsResult result) {
        List<ThingAttribute> things = result.getThings();

        String response = "{ \"things\": [";
        for (int i =0; i<things.size(); i++) {
            if (i!=0) 
                response +=",";
            response += String.format("{\"thingName\":\"%s\", \"thingArn\":\"%s\"}", 
                                                things.get(i).getThingName(),
                                                things.get(i).getThingArn());

        }
        response += "]}";
        return response;
    }

}
```

>>> 다음과 같이 작성 후, [Upload function to AWS Lambda] Click! -> 함수 이름 : ListThingsFunction -> AWSIoTFullAccess정책에 연결되어있는 IAM 역할 선택 -> Upload!

> 1-2. API Gateway 콘솔에서 REST API 생성

>> 1. API 생성 

>>> API 유형 : REST API / API 이름 : snow-api 

>> 2. 리소스 아래 /를 선택 -> 작업 드롭다운 메뉴 리소스 생성을 선택 -> 리소스 이름 :  devices 입력 

>> 3. 작업 드롭다운 메뉴 메서드 생성(Create Method) 선택

>> 5. 리소스 이름 (/devices) 아래에 드롭다운 메뉴 -> GET을 선택 후 확인 표시 아이콘(체크) 선택

>> 6. /devices – GET – 설정 -> 통합 유형에서 Lambda 함수를 선택 -> 저장

>>> - Lambda 프록시 통합 사용 상자를 선택하지 않은 상태 / Lambda 리전 : ap-northeast-2 / Lambda 함수 : ListThingsFunction

>> 7. Lambda 함수에 대한 권한 추가 팝업(Lambda 함수를 호출하기 위해 API Gateway에 권한을 부여하려고 합니다....”) 확인 Click!

>> 8. 앞서 적은, 0. CORS 활성화 및 API Gateway 콘솔에서 RESTAPI 배포 실행!

>>> - 여기서는 작업 드롭다운 메뉴 -> API 배포 Click! -> 배포 스테이지 드롭다운 메뉴 [새 스테이지]를 선택 -> 스테이지 이름 :  prod -> 배포 Click!


### 2. 디바이스 상태 조회 REST API 구축 

> 2-1. Lambda 함수 생성 

>> Project name : GetDeviceLambdaJavaProject

>> Class Name: GetDeviceHandler

>> Input Type : Custom

>>> pom.xml파일에 다음과 같이 추가

>>>
```javascript
 <dependencies>
    ...    
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-java-sdk-iot</artifactId>
    </dependency>

  </dependencies>
```

>>> GetDeviceHandler.java Code

>>>
```javascript
import com.amazonaws.services.iotdata.AWSIotData;
import com.amazonaws.services.iotdata.AWSIotDataClientBuilder;
import com.amazonaws.services.iotdata.model.GetThingShadowRequest;
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;

public class GetDeviceHandler implements RequestHandler<Event, String> {

    @Override
    public String handleRequest(Event event, Context context) {
        AWSIotData iotData = AWSIotDataClientBuilder.standard().build();

        GetThingShadowRequest getThingShadowRequest  = 
        new GetThingShadowRequest()
            .withThingName(event.device);

        iotData.getThingShadow(getThingShadowRequest);

        return new String(iotData.getThingShadow(getThingShadowRequest).getPayload().array());
    }
}

class Event {
    public String device;
}
``` 

>>> 다음과 같이 수정 후, [Upload function to AWS Lambda] Click! -> 함수 이름 : GetDeviceFunction -> AWSIoTFullAccess정책에 연결되어있는 IAM 역할 선택 -> Upload!

> 2-2. API Gateway 콘솔에서 REST API 생성

>> 1. 생성한 snow-api Click! -> 리소스 이름(/devices)을 선택

>> 2. 작업 드롭다운 메뉴에서 리소스 생성을 선택 -> 리소스 이름 :  device 입력 -> 리소스 경로(Resource Path)를 {device}로 바꾸기

>> 3. API Gateway Cors 활성화 옵션을 선택 -> 리소스 생성을 Click!

>> 4. /{device} 리소스가 강조 표시되면 작업에서 메서드 생성(Create Method) 선택

>> 5. 리소스 이름 (/{devices}) 아래에 드롭다운 메뉴 -> GET을 선택 후 확인 표시 아이콘(체크) 선택

>> 6. /devices/{device} – GET – 설정 -> 통합 유형에서 Lambda 함수를 선택 -> 저장

>>> - Lambda 프록시 통합 사용 상자를 선택하지 않은 상태 / Lambda 리전 : ap-northeast-2 / Lambda 함수 : GetDeviceFunction

>> 7. Lambda 함수에 대한 권한 추가 팝업(Lambda 함수를 호출하기 위해 API Gateway에 권한을 부여하려고 합니다....”) 확인 Click!

>> 8. /{device}의 GET 메서드의 통합 요청(Integration Request) 선택 -> 매핑 템플릿 Click -> 매핑 템플릿 추가 Click!

>>> - 요청 본문 패스스루 : 정의된 템플릿이 없는 경우(권장) / Content-Type : application/json 

>> 9. 추가 팝업 예, 이 통합 보호(Yes, secure this integration) Click!

>>> - 템플릿 생성 밑에 다음 code 작성 -> 저장

```javascript
{
  "device": "$input.params('device')"
}
```

>> 10. 앞서 적은, 0. CORS 활성화 및 API Gateway 콘솔에서 RESTAPI 배포 실행!

### 3. 디바이스 상태 변경 REST API 구축 

> 3-1. Lambda 함수 생성 

>> Project name : UpdateDeviceLambdaJavaProject

>> Class Name: UpdateDeviceHandler

>> Input Type : Custom

>>> pom.xml파일에 다음과 같이 추가

>>>
```javascript
 <dependencies>
    ...    
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-java-sdk-iot</artifactId>
    </dependency>

  </dependencies>
```

>>> UpdateDeviceHandler.java Code

>>>
```javascript
import java.nio.ByteBuffer;
import java.util.ArrayList;

import com.amazonaws.services.iotdata.AWSIotData;
import com.amazonaws.services.iotdata.AWSIotDataClientBuilder;
import com.amazonaws.services.iotdata.model.UpdateThingShadowRequest;
import com.amazonaws.services.iotdata.model.UpdateThingShadowResult;
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.fasterxml.jackson.annotation.JsonCreator;

public class UpdateDeviceHandler implements RequestHandler<Event, String> {

    @Override
    public String handleRequest(Event event, Context context) {
        context.getLogger().log("Input: " + event);

        AWSIotData iotData = AWSIotDataClientBuilder.standard().build();

        String payload = getPayload(event.tags);

        UpdateThingShadowRequest updateThingShadowRequest  = 
                new UpdateThingShadowRequest()
                    .withThingName(event.device)
                    .withPayload(ByteBuffer.wrap(payload.getBytes()));

        UpdateThingShadowResult result = iotData.updateThingShadow(updateThingShadowRequest);
        byte[] bytes = new byte[result.getPayload().remaining()];
        result.getPayload().get(bytes);
        String resultString = new String(bytes);
        return resultString;
    }

    private String getPayload(ArrayList<Tag> tags) {
        String tagstr = "";
        for (int i=0; i < tags.size(); i++) {
            if (i !=  0) tagstr += ", ";
            tagstr += String.format("\"%s\" : \"%s\"", tags.get(i).tagName, tags.get(i).tagValue);
        }
        return String.format("{ \"state\": { \"desired\": { %s } } }", tagstr);
    }

}

class Event {
    public String device;
    public ArrayList<Tag> tags;

    public Event() {
         tags = new ArrayList<Tag>();
    }
}

class Tag {
    public String tagName;
    public String tagValue;

    @JsonCreator 
    public Tag() {
    }

    public Tag(String n, String v) {
        tagName = n;
        tagValue = v;
    }
}
```

>>> 다음과 같이 작성 후, [Upload function to AWS Lambda] Click! -> 함수 이름 : UpdateDeviceFunction -> AWSIoTFullAccess정책에 연결되어있는 IAM 역할 선택 -> Upload!

> 3-2. API Gateway 콘솔에서 REST API 생성

>> 1. 생성한 snow-api Click! -> 리소스 이름(/{device})을 선택

>> 2. 리소스 이름 (/{devices}) 아래에 드롭다운 메뉴 -> PUT을 선택 후 확인 표시 아이콘(체크) 선택

>> 3. /devices/{device} – PUT – 설정 -> 통합 유형에서 Lambda 함수를 선택 -> 저장

>>> - Lambda 프록시 통합 사용 상자를 선택하지 않은 상태 / Lambda 리전 : ap-northeast-2 / Lambda 함수 : UpdateDeviceFunction

>> 4. Lambda 함수에 대한 권한 추가 팝업(Lambda 함수를 호출하기 위해 API Gateway에 권한을 부여하려고 합니다....”) 확인 Click!

>> 5. 모델 Click(리소스, 스테이지 등 메뉴가 있는 곳) -> 생성 Click! -> 작성 후, 모델 생성 Click!

>>> - 모델 이름 : UpdateDeviceInput / 콘텐츠 유형 : application/json 

>>> - 모델 스키마 다음 code 작성!
```javascript
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "title": "UpdateDeviceInput",
  "type" : "object",
  "properties" : {
      "tags" : {
          "type": "array",
          "items": {
              "type": "object",
              "properties" : {
                "tagName" : { "type" : "string"},
                "tagValue" : { "type" : "string"}
              }
          }
      }
  }
}
```

>> 6. /{device}의 PUT 메서드의 통합 요청(Integration Request) 선택 -> 매핑 템플릿 Click -> 매핑 템플릿 추가 Click!

>>> - 요청 본문 패스스루 : 정의된 템플릿이 없는 경우(권장) / Content-Type : application/json 

>> 7. 추가 팝업 예, 이 통합 보호(Yes, secure this integration) Click!

>>> - 템플릿 생성 UpdateDeviceInput 선택 -> 매핑 탬플릿 편집기에 다음과 같은 code 작성 -> 저장

```javascript
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

>> 8. 앞서 적은, 0. CORS 활성화 및 API Gateway 콘솔에서 RESTAPI 배포 실행!


### 4. 디바이스 로그 조회 REST API 구축

> 4-1. Lambda 함수 생성 

>> Project name : LogDeviceLambdaJavaProject

>> Class Name: LogDeviceHandler

>> Input Type : Custom

>>>  LogDeviceHandler.java Code

>>>
```javascript
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Iterator;
import java.util.TimeZone;

import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
import com.amazonaws.services.dynamodbv2.document.DynamoDB;
import com.amazonaws.services.dynamodbv2.document.Item;
import com.amazonaws.services.dynamodbv2.document.ItemCollection;
import com.amazonaws.services.dynamodbv2.document.QueryOutcome;
import com.amazonaws.services.dynamodbv2.document.Table;
import com.amazonaws.services.dynamodbv2.document.spec.QuerySpec;
import com.amazonaws.services.dynamodbv2.document.utils.NameMap;
import com.amazonaws.services.dynamodbv2.document.utils.ValueMap;
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;

public class LogDeviceHandler implements RequestHandler<Event, String> {

    private DynamoDB dynamoDb;
    private String DYNAMODB_TABLE_NAME = "Logging";

    @Override
    public String handleRequest(Event input, Context context) {
        this.initDynamoDbClient();

        Table table = dynamoDb.getTable(DYNAMODB_TABLE_NAME);

        long from=0;
        long to=0;
        try {
            SimpleDateFormat sdf = new SimpleDateFormat ( "yyyy-MM-dd HH:mm:ss");
            sdf.setTimeZone(TimeZone.getTimeZone("Asia/Seoul"));

            from = sdf.parse(input.from).getTime() / 1000;
            to = sdf.parse(input.to).getTime() / 1000;
        } catch (ParseException e1) {
            e1.printStackTrace();
        }

        QuerySpec querySpec = new QuerySpec()
                .withKeyConditionExpression("deviceId = :v_id and #t between :from and :to")
                .withNameMap(new NameMap().with("#t", "time"))
                .withValueMap(new ValueMap().withString(":v_id",input.device).withNumber(":from", from).withNumber(":to", to)); 

        ItemCollection<QueryOutcome> items=null;
        try {           
            items = table.query(querySpec);
        }
        catch (Exception e) {
            System.err.println("Unable to scan the table:");
            System.err.println(e.getMessage());
        }

        return getResponse(items);
    }

    private String getResponse(ItemCollection<QueryOutcome> items) {

        Iterator<Item> iter = items.iterator();
        String response = "{ \"data\": [";
        for (int i =0; iter.hasNext(); i++) {
            if (i!=0) 
                response +=",";
            response += iter.next().toJSON();
        }
        response += "]}";
        return response;
    }

    private void initDynamoDbClient() {
        AmazonDynamoDB client = AmazonDynamoDBClientBuilder.standard().build();

        this.dynamoDb = new DynamoDB(client);
    }
}

class Event {
    public String device;
    public String from;
    public String to;
}
```

>>> 다음과 같이 작성 후, [Upload function to AWS Lambda] Click! -> 함수 이름 : LogDeviceFunction -> AmazonDynamoDBFullAccess정책에 연결되어있는 IAM 역할 선택 -> Upload!


> 4-2. API Gateway 콘솔에서 REST API 생성

>> 1. 생성한 snow-api Click! -> 리소스 이름(/{device})을 선택

>> 2. 작업 드롭다운 메뉴에서 리소스 생성을 선택 -> 리소스 이름 :  log 입력 ->  /log – GET – 설정

>> 3. /devices/{device}/log – GET – 설정 -> 통합 유형에서 Lambda 함수를 선택 -> 저장

>>> - Lambda 프록시 통합 사용 상자를 선택하지 않은 상태 / Lambda 리전 : ap-northeast-2 / Lambda 함수 : LogDeviceFunction

>> 4. Lambda 함수에 대한 권한 추가 팝업(Lambda 함수를 호출하기 위해 API Gateway에 권한을 부여하려고 합니다....”) 확인 Click!

>> 5. 메서드 요청 Click -> URL 쿼리 문자열 파라미터(URL Query String Parameters) Click!

>>> 쿼리 문자열 추가 : from (캐싱 uncheck) / 쿼리 문자열 추가(Add query string) : to (캐싱 uncheck)

>> 6. /log GET 메서드 메서드의 통합 요청(Integration Request) 선택 -> 매핑 템플릿 Click -> 매핑 템플릿 추가 Click!

>>> - 요청 본문 패스스루 : 정의된 템플릿이 없는 경우(권장) / Content-Type : application/json 

>> 7. 추가 팝업 예, 이 통합 보호(Yes, secure this integration) Click!

>>> - 템플릿 생성 UpdateDeviceInput 선택 -> 매핑 탬플릿 편집기에 다음과 같은 code 작성 -> 저장

```javascript
{
  "device": "$input.params('device')",
  "from": "$input.params('from')",
  "to":  "$input.params('to')"
}
```

>> 8. 앞서 적은, 0. CORS 활성화 및 API Gateway 콘솔에서 RESTAPI 배포 실행!

## File 설명

### 1. DFDFRobot_HX711-master / HX711_library

* Weight Sensor관련 library

> Weight Sensor 1 관련 library : DFDFRobot_HX711-master

> Weight Sensor 2 관련 library : HX711_library

### 2. Arduino_SnowProject(Arduino Code)

* arduino_secrets.h는 위에서 설명했기때문에 생략

#### 2-1. Arduino_SnowProject.ino

> 1. Weight sensor 관련 library, Led, Buzzer, mkr, wifi 관련 파일들 정의

> 2. 

- Arduino에 연결된 Weight Sensor, Led, buzzer의 현재 상태를 topic에 update 

```javascript
    if(MyScale.readWeight() < 0.0 ){
    float t1 = MyScale.readWeight();
    float t2 = scale.get_units(); 
    t1 *= -1.0;
    
    if(scale.get_units()<0.0){
      t2 *= -1.0;
    }
    // make payload for the device update topic ($aws/things/SnowProject/shadow/update)
    sprintf(payload,"{\"state\":{\"reported\":{\"weight1\":\"%0.2f\",\"weight2\":\"%0.2f\",\"LED\":\"%s\",\"BUZZER\":\"%s\"}}}",t1,t2,led,buzzer);
  
  else{
    float t1 = MyScale.readWeight();
    float t2 = scale.get_units();
    if(scale.get_units()<0.0){
       t2 *= -1.0;
    }
    // make payload for the device update topic ($aws/things/SnowProject/shadow/update)
     sprintf(payload,"{\"state\":{\"reported\":{\"weight1\":\"%0.2f\",\"weight2\":\"%0.2f\",\"LED\":\"%s\",\"BUZZER\":\"%s\"}}}",t1,t2,led,buzzer);
  }
```

- Led가 ON값으로 변경되면, Led와 Buzzer ON <-> Led가 OFF값으로 변경되면, Led와 Buzzer OFF

```javascript
/*
   * LED와 BUZEER를 하나의 제어로 하게 했음.
   * LED가 ON -> BUZZER도 ON
   * LED가 OFF -> BUZZER도 OFF
   */
 if (strcmp(led,"ON")==0 /*&& strcmp(buzzer,"ON")==0*/) {
    led1.on();
    buzzer1.on();
    sprintf(payload,"{\"state\":{\"reported\":{\"LED\":\"%s\",\"BUZZER\":\"%s\"}}}","ON","ON");
    sendMessage(payload);
    
  } else if (strcmp(led,"OFF")==0 /*&& strcmp(buzzer,"OFF")==0*/) {
    led1.off();
    buzzer1.off();
    sprintf(payload,"{\"state\":{\"reported\":{\"LED\":\"%s\",\"BUZZER\":\"%s\"}}}","OFF","OFF");
    sendMessage(payload);
  }
```

#### 2-2. Buzzer.h, Buzzer.cpp / Led.h, Led.cpp

> 1. Buzzer.h, Buzzer.cpp : Buzzer관련 입출력 PIN 설정, ON/OFF기능 구현, Buzzer의 ON/OFF 상태 관련

>> (BUZZER가 ON이면 사이렌 소리가 울리도록 코드 작성)

> 2. Led.h, Led.cpp : Led관련 입출력 PIN 설정, ON/OFF기능 구현, Led의 ON/OFF 상태 관련

>> (LED가 ON이면 led가 켜지도록 코드 작성)

### 3. Android_SnowProject/SnowProject(Android App Code)

- RealActivity, AllActivity에 있는 urlStr1, urlStr2는 각각 6. API Gateway를 이용한 RestAPI 생성에서 만든 API 서버 URL

> urlStr1 : 디바이스 상태 조회/변경 API URL(https://xxxxxxxx.execute-api.ap-northeast-2.amazonaws.com/prod/devices/{devices_name})

> urlStr2 : 디바이스 로그 조회 API URL(https://xxxxxxxx.execute-api.ap-northeast-2.amazonaws.com/prod/devices/{devices_name}/log)


#### 3-1. ButtonActivity.java

1. Button을 클릭해서 intent로 API URI를 넘겨준다.

- It will start RealActivity when you click the "nowBtn(현재상태/제어)" Button.

- Likewise, Click the "allBtn(로그 조회/제어)" Button to start AllActivity.

<img src = "https://user-images.githubusercontent.com/70942492/101483897-2bf3a000-399c-11eb-860f-6b8fe85679a1.png" width="30%" height="30%">

#### 3-2. RealActivity.java

1. 현재 센서들의 값을 보여주며, LED/BUZZER를 제어할 수 있도록 하였다. (Weight 2 Sensor가 고장나서 제대로 측정이 안됨.)

- 기상청 API, 건축물대장 API를 이용해서 정보가 뜨도록 하였다.

<img src = "https://user-images.githubusercontent.com/70942492/101484015-52194000-399c-11eb-9dad-f21527d6d003.png" width="30%" height="30%">

### 3-3. AllActivity.java

1. 원하는 날짜, 시간을 선택해서 센서들의 로그값을 확인 하며, LED/BUZZER를 제어할 수 있다.

<img src = "https://user-images.githubusercontent.com/70942492/101485555-aae9d800-399e-11eb-9fd4-70f1489738ea.png" width="30%" height="30%">



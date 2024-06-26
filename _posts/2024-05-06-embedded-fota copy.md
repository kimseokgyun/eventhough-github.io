---
layout: post
title: FOTA (Firmware Over The Air)
date: 2024-05-06 20:32 +0900
last_modified_at: 2024-05-06 20:32 +0900
tags: [Embedded system]
toc:  true
---
## Conclusion
Embedded System 에서 사용되는 Live Firmware Update System 을 설명한다

## Meaning
- MCU에 물리적으로 접근없이 Server를 연동시켜 자동으로 Fimware Update 이 가능한 시스템.
- **XCP Protocl** 를 이용하여 Version을 체크하여 형상유지에 용이한 Firmware Update를 진행한다.
- Link File을 이용하여 MCU Flash Memory Section을 만들어 Version, 보안체크용 데이터를 지정시켜 보안성에 강화

<!-- ![placeholder](http://placehold.it/800x400 "Large example image") -->

## Concept
![placeholder](/upload_image/fota/fota_1.png "Large example image")

IOT이든, 차량이든 어떤 제품에 완제품형태로 결착된 MCU에 직접 물리적으로 접촉하여 펌웨어 업데이트하는건 여간 불편한 일이아니다.

따라서 물리적 연결없이 쉽게 업데이트할수있되, 누구든 쉽게 업데이트할수 없게끔 보안 장치를 걸어둔 부트로더 프로그램을 OTA 프로그램이라고 한다.

차량에서 사용하는 가장 간단하고 기본적인 MCU OTA 흐름도는 다음과 같다.




### XCP Protocol
![placeholder](/upload_image/fota/fota_2.png "Medium example image")

XCP on CAN , XCP on Ethernet 다양한 물리적 통신에 패킷에 실리는 명령 명세라고 이해하면 편할듯하다

예를들어 Host 에서 보내는 CAN 메세지중 XCP Message Frame 의 PID 에 0x55 , DAQ 에 8000A000 이라는 메세지를 보냈다고 가정하자

Slave측에서는 0x55는 Flash Memory Read 라는 명령이고 , 읽어야하는 Flash Memory주소는 8000A000 이구나 라고 해석을 하게된다

이러한 미리 정의된 XCP 명령 프로토콜을 이용하여 Host는 Slave 의 Flash 또는 RAM 에 저장되어있는 값을 받아서 볼수있다


다음은 실제 XCP PID "C9" Flash Write 명령을통해 Flash Memory에 값을 저장한 모습

![placeholder](/upload_image/fota/fota_2_1.png "Medium example image")

이 Flash , RAM 의 저장되어있는 값은 ECU 에서 돌고있는 **칼만필터의 파라미터**일수도, **ECU의 ID**가 될수도, ECU가 가지고있는 프로그램의 **펌웨어 버전**이 될수도 있는것

<ins>XCP Protocol을 이용하여 Flash Write,Erase,Read 명령을 수행하며, ECU와의 Version 태그를 비교하며 Live Firmware Update를 수행할수있다</ins>


### MCU (Slave) 준비과정

위 과정에서 사용되는 ECU ID , Version 정보는 모두 MCU측의 FLASH Memory의 특정 Section에 저장되어야한다.

그 이유는 Host 측에서 정확히 ECU 의 어느 주소값에 ECU의 ID, Version 정보가 있는지 알아야 하기때문

{% highlight js %}
// STM32H Example Flash.id file

MEMORY
{
  RAM   (xrw)   : ORIGIN = 0X20000000,  LENGTH = 128K
  FLASH   (rx)   : ORIGIN = 0X08000000,  LENGTH = 12K-4
  ECUID   (rx)   : ORIGIN = 0X08002FFFC,  LENGTH = 1
  VERSION_MAJOR   (rx)   : ORIGIN = 0X0802FFFD,  LENGTH = 128K
  VERSION_MINOR   (rx)   : ORIGIN = 0X0802FFFE,  LENGTH = 128K
  VERSION_PATCH   (rx)   : ORIGIN = 0X0802FFFF,  LENGTH = 128K
}

{% endhighlight %}


다음 예시와같이 Link File을 통해 저장할 공간 (Section)을 만들어준다.

그후 Jenkins와 같은 CI/CD 툴로 Version Tag를 변수로 넣어 자동 컴파일을 걸어두면 쉽게 유지보수가 가능하다.


![placeholder](/upload_image/fota/fota_4.png "Medium example imagee")


OpenBlt는 다음과같이 Bootloader단과 Application단 Section을 나눈다.

우리가 지우고, 다시 덮어야하는 Update부분은 User Program , 즉 Application 단이다.

이렇게 나누지 않는경우, Application 단에서 Firmware Update가 진행되면 Flash Write/Read시 위험할 상황이 생긴다.

또한, 다음과같이 Bootloader단에서 Firmware Update를 진행하면 Update 실패시 재요청하면 되므로 안전하다.





### Host -> Slave WorkFlow

보통 MCU는 Application 단에서 Firmware Update 요청이 들어올 것이다.

안전성을위해 오직 재부팅시에만 Frimware Update를 할 예정이라면 아래의 Application Flow 를 구현하지않아도 된다.

#### Application Flow

![placeholder](/upload_image/fota/fota_5.png "Medium example imagee")

1. ECU1 는 Application 수행중에 XCP Call back을 수행하고 있을것이다.<ins>(대부분 CAN통신을 이용해 Application을 수행함으로)</ins>
이때 HOST Req에서 어떤 암호입력 예를들어 string 변수 12345678을 함께 보내고, MCU측의 Hash 함수로 "password"를 변환하는 형식의 보안 알고리즘을 추가 가능

2. ECU1 Res를 통해 ECU1이 확실하다면, ECU1의 Version 정보를 체크한다. 
해당 프로세스또한 XCP Protocol PID를 이용하여 진행한다

3. Host측이 ECU1 Version을 비교하여 Update Flag를 보낸다

4. ECU1측에서 Update Flag를 받는다면 재부팅을 진행함

#### Bootloader Flow

![placeholder](/upload_image/fota/fota_6.png "Medium example imagee")

1. 재부팅이 시작되면 Application에서 진행했던, ECU ID, Version Check를 다시 진행한다.

2. Version Check이후 ECU1 Res를 통해 Firmware Update가 확실해진다면 XCP PID를 이용한 MEMORY Section Clear 가 수행된다.

3. CRC Check와 함게 Flash Write 수행


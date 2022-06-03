# raspberypi4
라즈베리파이4 수업 기말대체과제

rp2040내부의 센서는 

x축 =>  땅과 수평으로 걸어가는 방향의 변화

y축 =>  진행방향의 옆방향에서의 변화

z축 =>  수직으로 점프하는 방향으로의 변화를 나타낸다.

사람이 걸을때 평균적으로 x축 변화랑이 가장크고, 양옆으로 조금씩 왔다갔다 하는 변화가 곧 y축의 변화이고,z축이 위아래의 변화인데 
알고리즘으로 구현하려 했더니 약간의 문제가 있다. 
가속도 센서가 걸을때 정확하게 이동하는 방향으로 x축을 측정하는 센서가 향해 있어야 이게 성립한다.
센서를 주머니에 넣고 걸었는데 나는 x축으로 이동하지만 센서 입장에서는 y축으로 이동하는 결과가 될수 있다는것.

3축 가속도 센서중 가장큰 변화량을 가지는 값을 x축으로 삼고, 상대적으로 변화율이 적은 값을 y축, 나머지를 z축으로 삼으면 이를 해결할수있을것 같지만
사람에 따라 보폭도 다르고 위아래로 진동하는것과 양옆으로 이동하는것도 다 달라서 좀더 정확한 알고리즘의 필요성을 느끼고
한영환. (2015). 가속도 센서를 이용한 걸음수 검출 알고리즘.재활복지공학회 논문지 제9권 제3호 . 245-250.https://www.koreascience.or.kr/article/JAKO201515139872162.pdf
논문을 참고해서 걸음수 알고리즘을 작성해 보았다

![3축 가속도 센서](https://user-images.githubusercontent.com/103232943/171842388-c15a1357-d271-4917-84c1-f5800d6bff2a.PNG)

위의 논문에 따르면 3축 가속도에서 나온값으로 SVM이라는 신호벡터 크기를 구하고

![신호 벡터 크기값](https://user-images.githubusercontent.com/103232943/171842590-84b360c8-582c-4f5c-910d-df0959be35f3.PNG)

그래프로 나타내면 x,y,z 축의 가속도 정보가 아래 SVM신호 벡터 크기 정보처럼 나온다고 한다.


이 SVM 신호를 가지고 2개의 임계값을 고정으로 설정하는게 아닌 걸음에 따라 다르게 임계값을 설정하여
걸음수의 정확성을 높였다고 하는데 이건 다음에 기회가 되면 만들어보고 우선 고정으로 임계값을 2개 설정하여 걸음수를 구하는 알고리즘을
작성해 보려고 한다.

![임계값](https://user-images.githubusercontent.com/103232943/171843793-6c8155eb-aa41-4a14-9d81-6b012197d51c.PNG)

임계값을 C1,C2이렇게 2개를 잡고 C1을 넘고 한번 C2아래로 내려갔다가 오면 1걸음을 걸었다고 판단하고 10초동안 재었을때 걸음수가 많으면 뛰는 상태라고 생각하고 코드를 짜보았다.

1.  I2c  통신 프로토콜을 이용해 외부 rp2040장치 => 라즈베리파이 내부로 text형식의 데이터 전송
 
import network,time
from umqtt.simple import MQTTClient #导入MQTT板块
from machine import I2C,Pin,Timer
from lsm6dsox import LSM6DSOX

step1 = 0
from machine import Pin, I2C
lsm = LSM6DSOX(I2C(0, scl=Pin(13), sda=Pin(12)))

def WIFI_Connect():
    wlan = network.WLAN(network.STA_IF) #STA模式
    wlan.active(True)                   #激活接口
    start_time=time.time()              #记录时间做超时判断

    if not wlan.isconnected():
        print('connecting to network...')
        wlan.connect('iptime105', '@rjsghks12') #输入WIFI账号密码
        
    if wlan.isconnected():
        print('network information:', wlan.ifconfig())
        return True    

def MQTT_Send(tim):
    client.publish(TOPIC, 'Accelerometer: x:{:>8.3f} y:{:>8.3f} z:{:>8.3f}'.format(*lsm.read_accel()))
    print('Accelerometer: x:{:>8.3f} y:{:>8.3f} z:{:>8.3f}'.format(*lsm.read_accel()))

if WIFI_Connect():
    SERVER = '192.168.0.6'   # my rapa ip address , mqtt broker가 실행되고 있음
    PORT = 1883
    CLIENT_ID = '' # clinet id 이름
    TOPIC = 'rp2040' # TOPIC 이름
    client = MQTTClient(CLIENT_ID, SERVER, PORT,keepalive=30)
    client.connect()

    #开启RTOS定时器，编号为-1,周期1000ms，执行socket通信接收任务
    tim = Timer(-1)
    tim.init(period=1000, mode=Timer.PERIODIC,callback=MQTT_Send)

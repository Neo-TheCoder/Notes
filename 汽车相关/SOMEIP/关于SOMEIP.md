# �������ͨ���м��SOME/IP����
## 1 SOME/IP���м����
׼ȷ��˵��ͨ��Э�飬����˵Ҳ��������Ϊͨ���м��
ȫ����**Scalable service-Oriented MiddlewarE over IP**
��������IPЭ����������Ŀ���չ��ͨ���м��Э��
���Կ���������Ҫ�أ�
1. �������SOA
2. ����IPЭ��֮�ϵ�ͨ��Э��
3. �м��

## 2 SOME/IP������
����ͨ����أ�
1. �����֣�Service Discovery��
2. Զ�̷�����ã�RPC��Remote Producer Call��
3. ��д������Ϣ��Getter & Setter��

## 3 SOME/IP��CAN������
CAN�Ǵ�ͳ���������ͨ��Э�飬CAN FD������չ
���ߵ���Ҫ�������ڣ�
1. ͨ�Ÿ��ɵĲ��죺�����ص��ֽ�����8Byte -- 1500Byte
2. ͨ�����ʵĲ���   1Mb/s -- 1000Mb/s
3. CAN���������ģ�SOME/IP����������
4. CAN�ǻ����ź���˫�����д����ź�  --  SOME/IP���ڳ�����̫���д����ź�
SOME/IPʵ������λ�ڳ�����̫������Э���е�Ӧ�ò㣬�ǻ���TCP/UDPЭ��֮�ϵ�Ӧ��
GENIVI��֯���SOME/IP��׼ʵ���˿�Դvsomeip����
SOME/IP Э��һ��ָ������ SOME/IP��SOME/IP-SD��SOME/IP-TP 3�֡�

5. ��Ϣ��ʽ
һ�������� SOME/IP ��Ϣ�������������ݣ�
MessageID��Event ID / Method ID�� -- 32bit
Length -- 32bit -- ��Ϣ����
Request ID�� Client ID & Session ID��-- 32bit
Protocal Version -- Э��汾��
Interface Version -- �ӿڰ汾��
Message Type -- ��Ϣ����
Return Code -- ���ر���
Payload -- ���ݸ��أ��ɱ䣩��SOME/IP�ײ���Ի���TCP����UDP��ʹ��Payload��������һ���������UDP����payload������1400B�������������TCP�����ݷֶδ��䣬��ôSOME/IP����ʵ�ָ��������Ĵ���
PS������SOME/IP Header���ݲ��ô�˴��䣬��Payload�е����ݴ��˳��ͨ����������

6. SOME/IP֧�ֵ����ݽṹ����
������uint��int...�����ڽṹ�����ݻ����ڴ��а���˳�����δ��

7. SOME/IP��Ϣͨ������
1. R & R��Request & Response��
2. F & F (Fire & Forget)��ֻ����Ӧ��
3. Notification Event-- �������Ļ���
�� Notification ��3�������
Cyclic Update �����Եķ������ value �ı仯
Update On Change ��� value �����仯������������
Epsilon Change ��� value ��ֵ������Ӧ�� epsilonֵ����ô����������Ϣ
5. Fields
��Ե���һ��status��������һ���Ϸ�ֵ
һ��Fields����Ӧ������Setter��Getter��Notification�Ľ�ϡ�
ִ��Setter���ͻ��˷������󣬲���������Ϣ�е�payload����ȷ��ֵ�����ڷ����Ӧ����Ϣ�е�payloadҪͬ���������ֵ
ִ��Getter���ͻ��˷�������������Ϣ�а���һ���յ�payload��������յ�������Ϣ��field��ֵ��䵽Ӧ����Ϣ�е�payload

8. SOME/IP-SD
��������Ҫ���ܣ�
̽����������ṩ����
SD����Ϣ�ṹ����SOME/IP��Ϣ��֮�ϸ�����һЩ����������Entries Array�����ڱ��Service����Event Group

�����ֵ�����
̸�� Service Discovery ֮ǰ��Ҫ�����������������
**Service**��ָ�� functional entities���������һ���.
**find**����һ����������ڱ�ʶ available ״̬�� service �������ǵ�λ��
**event**��Service Discovery ���͵� Message ����Ϊ event
**Eventgroups**��event �ļ���

ervice�������ϸ�ڡ�
Service��2�ࣺServer Service��Client Service�� Service��2��״̬: Down��Available��
һ��ECU��Ҫ��������Service:ServerService��ClientService��
Ҳ����˵��һ�� ECU ���Զ����ṩ����,���ʱ�������� Server Service ģ�鸺��Ҳ���Զ�������������ʱ���� Client Service �ṩ��
ʵ�ʵ����̷ǳ����ӣ�һ�� Service �� Down ״̬�� Available ״̬֮����л���һϵ�е�״̬�׶�ת����

# SOME/IP��DDS�Ĳ���
DDS���������ݵģ�ͨ�������������������ޣ����һ�ʹ���˹����ڴ棬QOS�ḻ��




# ʵ������
������**service**Ϊ��
```json
 "services" : [
    {
      "name" : "SomeipStartApplicationCmService1_ServiceInterface",
      "service_id" : 1667,
      "major_version" : 1,
      "minor_version" : 0,
      "methods" : [
        {
          "name" : "StartApplicationMethod1",
          "id" : 5,
          "proto" : "tcp"
        },
        {
          "name" : "StartApplicationField1Getter",
          "id" : 2,
          "proto" : "tcp"
        },
        {
          "name" : "StartApplicationField1Setter",
          "id" : 4,
          "proto" : "tcp"
        }
      ],
      "events" : [
        {
          "name" : "StartApplicationEvent1",
          "id" : 32769,
          "field" : false,
          "proto" : "tcp"
        },
        {
          "name" : "StartApplicationField1Notifier",
          "id" : 32770,
          "field" : true,
          "proto" : "tcp"
        }
      ],
      "eventgroups" : [
        {
          "id" : 1,
          "events" : [
            32769,
            32770
          ]
        }
      ]
    }
 ]
```
## ������
**�㲥**
### OfferService
`Entries Array`����`Offer Service`������Service ID��Instance ID

### FindService
`Entries Array`����`Find Service`������Service ID��Instance ID

һ��**�㲥����**��������`Entries Array`�У�ARRAY1��ʾ`FindService`��ARRAY2��ʾ`OfferService`
Options Array�У�����Ipv4 Endpoint Option��`IP��ַ + �˿ں� + Э��UDP`


**����**
### Subscribe
`Entries Array`����`Subscribe Eventgroup Entry`
ָ��Ҫ���ĵ�`service id - instance id - Eventgroup - version`

### SubscribeAck
`Entries Array`����`Subscribe Eventgroup Ack Entry`

һ��**�㲥����**��������`Entries Array`�У�ARRAY1��ʾ`Subscribe`��ARRAY2��ʾ`SubscribeAck`

## ͨ��
### Event
server����`Message Type`Ϊ`Notification`���͵ı���

### Method
������Ҫ���ģ���**����-Ӧ��ģʽ**��
client����Type`Message Type`Ϊ`Request`���͵ı���
server����Type`Message Type`Ϊ`Response`���͵ı���







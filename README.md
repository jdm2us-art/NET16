Лабораторная работа "Классификация и маркировка трафика" \
Можете использовать симулятор для построения топологии и поиска/проверки ответов на задания.

# Задание

На картинке изображена схема сети с тремя маршрутизаторами.

<img width="1503" height="619" alt="169475092-354046d6-f69c-4572-a2aa-4e86f0df9fa6" src="https://github.com/user-attachments/assets/73684657-2779-434f-9888-dcb0142b5f65" />

Необходимо классифицировать и маркировать трафик следующим образом:

TCP трафик с destination порт 80/443 установить DSCP AF43 между всеми клиентами. \
UDP трафик с destination порт 5060-5061 установить CS5 между всеми клиентами. \
Для ICMP трафика значение COS установить 3 между всеми клиентами. \
Трафик от клиента Client2, Client3 до Client1 должен маркироваться EF. \
Отправьте полный список конфигураций: class-map, policy-map service-policy.

# ОТВЕТ:

1. Списки доступа (ACL) для классификации \
Сначала создадим расширенные ACL для точного захвата трафика по портам и адресам.

! --- ACL для TCP портов 80 и 443 --- \
ip access-list extended HTTP_HTTPS_TRAFFIC \
 permit tcp any any eq 80 \
 permit tcp any any eq 443 \
! --- ACL для UDP портов 5060-5061 (SIP) --- \
ip access-list extended SIP_TRAFFIC \
 permit udp any any range 5060 5061 \
! --- ACL для ICMP --- \
ip access-list extended ICMP_TRAFFIC \
 permit icmp any any \
! --- ACL для трафика от Client2 и Client3 (VPC) к Client1 --- \
! Client1 — 192.168.5.10, Client2 — 172.16.1.20, VPC — 10.10.10.30 
ip access-list extended CLIENT2_3_TO_CLIENT1 \
 permit ip host 172.16.1.20 host 192.168.5.10 \
 permit ip host 10.10.10.30 host 192.168.5.10

 2. Class-map (Классификация) \
Создаем классы для сопоставления трафика с ACL.

class-map match-any CM_HTTP_HTTPS \
 match access-group name HTTP_HTTPS_TRAFFIC \
class-map match-any CM_SIP  \
 match access-group name SIP_TRAFFIC \
class-map match-any CM_ICMP \
 match access-group name ICMP_TRAFFIC \
class-map match-any CM_EF_TRAFFIC \
 match access-group name CLIENT2_3_TO_CLIENT1 \

3. Policy-map (Маркировка) \
Создаем политику, которая будет присваивать необходимые значения DSCP и CoS.

policy-map PM_MARKING \
 ! TCP 80/443 -> DSCP AF43 \
 class CM_HTTP_HTTPS \
  set dscp af43 \
 ! UDP 5060-5061 -> CS5 \
 class CM_SIP \
  set dscp cs5 \
 ! ICMP -> CoS 3 \
 class CM_ICMP \
  set cos 3 \
 ! Client2/VPC -> Client1 -> EF \
 class CM_EF_TRAFFIC \
  set dscp ef \
 ! Весь остальной трафик не маркируется (или маркируется по умолчанию) \
 class class-default \
  set dscp default \

  4. Применение на интерфейсах (Service-policy) \
Применяем политику на входящих интерфейсах, где подключены клиенты.

На маршрутизаторе R1 (интерфейс к Client1):

interface GigabitEthernet0/1 \
 service-policy input PM_MARKING

 На маршрутизаторе R2 (интерфейс к Client2):

interface GigabitEthernet0/1 \
 service-policy input PM_MARKING \

На маршрутизаторе vIOS (интерфейс к VPC):

interface GigabitEthernet0/1 \
 service-policy input PM_MARKING

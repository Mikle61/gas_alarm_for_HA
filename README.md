# gas_alarm_for_HA

## Описание

Данное примитивное устройство служит адаптером между Home Assistant и системами автоматического контроля загазованности САКЗ-МК-1(2)-1Аi. В задачу адаптера входит:
- передача сигнала «Обнаружен газ» от САКЗ к HA
- передача сигнала «Неисправность» от САКЗ к НА
- передача сигнала «Обнаружен газ» от НА к САКЗ

Т.о. на стороне НА не только появляется возможность мониторить состояние сигнализации САКЗ, но и давать команду на отключения газа, руководствуясь показаниями других датчиков загазованности и задымленности, подключенных непосредственно к HA

Адаптер реализован на плате Wemos D1 mini (ESP8266) на платформе ESPHome. Подключение адаптера к САКЗ не требует никакой переделки этой системы.

Система САКЗ представляет собой набор независимых датчиков, клапана газа и (опционально) пультов и GPS извещателей, которые объединяются общей шиной на базе телефонных 6-ти пиновых разъемов RJ12. Чтобы подключить данный адаптер к САКЗ необходим __Прямой__ кабель 6Р6С с соединением пинов 1-1, 2-2, 3-3, 4-4, 5-5, 6-6. __ВНИМАНИЕ__ поставляемый в комплекте с САКЗ соединительный кабель шины для этой цели не подходит!
Адаптер может подключаться только к сигнализаторам СЗ-2-2Аi, СЗ-1-1Аi и СЗ-3-1Аi. Если в системе используются пульты ПК-Аi или аналогичные, адаптер следует подключать к сигнализаторам, а не к пульпитам.
Адаптер может быть подключен двумя способами:
- при подключении непосредственно к сигнализаторам он будет получать питание от них.
- можно подать питание от штатного источника питания САКЗ на адаптер и от него записать всю систему. (__внимание__ разъемы адаптера не взаимозаменяемы. Разъем «ПИТ» предназначен только для подачи питания)
 

-----------------------

## Инфрмация для изготовления:

[Схема и плата](Schematic/Schematic1.md)

[Корпус для 3д печати](3d/3d.md)

--------------
Файл для ESPhome

``` yaml
esphome:
  name: esp-gas-alarm
  friendly_name: esp-gas-alarm


esp8266:
  board: d1
#  restore_from_flash: true
#
#  Используемые пины:
#
# GPIO4 - входной сигнал превышения порога газа
# GPIO5 - входной сигнал неисправности  
#
# GPIO3 - выходной сигнал тревоги
# GPIO2 - просто мигалка
#
# Enable logging
logger:
  level: DEBUG
#  level: ERROR
  logs:
    switch: ERROR


#
# При отключении не виден в HA и не может получать дамп по эфиру
api:
  encryption:
    key: "n9lCNL7BLBAR1gyOy6R56tTk8Wl4K14OvJ5xaXek4iw="

# при отключении пропадает загрузка по эфиру
ota:
  - platform: esphome
    password: "c513c7ef9371d436397ad98b11693225"

wifi:
#  use_address: esp-test-roz.local
# При отклычении паспорта и идентификатора  на точке доступа поднимается веб сервер на адресе 192.168.4.1
# Для этого необходимо также выключить captive_portal  
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  reboot_timeout: 0s
  ap:
    ssid: "AP-gas-alarm"
    password: "12345678"
    ap_timeout: 10s

# При отключении на точке доступа поднимается веб сервер на адресе 192.168.4.1 
# При включении и не поключении к WIFI подимается диалог для выбора сети
captive_portal:

web_server:
  port: 80
  version: 2
  local: true
  auth:
    username: "admin"
    password: "12345678"

text_sensor:
# Строка времени работы
# служебное
  - platform: template
    name: "uptime_text"
    id: uptime_human
    icon: mdi:clock-start

          
sensor:
# служебное
# время работы в секундах
  - platform: uptime
    name: "uptime"
    id: uptime_sensor
    update_interval: 60s
    on_raw_value:
      then:
        - text_sensor.template.publish:
            id: uptime_human
            state: !lambda |-
              int seconds = round(id(uptime_sensor).raw_state);
              int days = seconds / (24 * 3600);
              seconds = seconds % (24 * 3600);
              int hours = seconds / 3600;
              seconds = seconds % 3600;
              int minutes = seconds /  60;
              seconds = seconds % 60;
              return (
                (days ? to_string(days) + "d " : "") +
                (hours ? to_string(hours) + "h " : "") +
                (minutes ? to_string(minutes) + "m " : "") +
                (to_string(seconds) + "s")
              ).c_str();

binary_sensor:
# Покказыввает доступность в Home Assistant
  - platform: status 
    name: "status"
#сигнал превышения порога газа  
  - platform: gpio
    pin:
      number: GPIO4
      inverted: true
    name: "Alarm gas"
    id: Porog
#сигнал неисправности  
  - platform: gpio
    pin: GPIO5
    name: "Failure"
    id: Failure

switch:
#Включение тревоги
  - platform: gpio
    pin:
      number: GPIO3
      inverted: true
    name: "Alarm switch"
    id: Trevoga
    restore_mode: ALWAYS_OFF
    on_turn_on:
    - delay: 10000ms
    - switch.turn_off: Trevoga
#перезапуск esp
  - platform: restart
    name: "Restart"
#просто мигалка
  - platform: gpio
    pin: GPIO2
    id: Migalka
    internal: true
    restore_mode: ALWAYS_OFF
    on_turn_on:
    - delay: 1000ms
    - switch.turn_off: Migalka
    on_turn_off:
    - delay: 10000ms
    - switch.turn_on: Migalka

```

# control_air
Официальная документация по проекту "Многофункциональный датчик на базе ESP8266"

Введение

Этот проект представляет собой многофункциональный датчик, разработанный на базе платы ESP8266. Датчик способен измерять уровни CO2, влажности, температуры и органических соединений в окружающей среде. 

Основные характеристики

- Датчик CO2: Измеряет уровень углекислого газа в воздухе.
- Датчик влажности: Определяет относительную влажность воздуха.
- Датчик температуры: Измеряет температуру окружающей среды.
- Датчик органических соединений: Обнаруживает наличие органических соединений в воздухе.

Описание компонентов
-
- Датчик углекислого газа (CO2): Датчик CJMCU-811 используется для измерения уровня углекислого газа в воздухе. Он обеспечивает точные и надежные показания, что делает его идеальным выбором для этого проекта.

- Датчик влажности и температуры: Датчик AHT2x используется для измерения относительной влажности и температуры окружающей среды. Он обладает высокой точностью и стабильностью.

- ESP8266: Это мощная плата для разработки, которая обеспечивает Wi-Fi соединение и умеренное количество GPIO-пинов для подключения датчиков.

- Варианты питания: Проект может быть питаем от различных источников питания. Вы можете использовать USB, батареи или даже солнечные панели.

- Программатор: используется переделанный программатор для загрузки кода на ESP8266. Это обеспечивает гибкость и удобство при разработке проекта.

Фотографии

фотография самого датчика. Он компактен и легко подключается к ESP8266.

фотография платы ESP8266. Она маленькая, но мощная, и обеспечивает все необходимые функции для этого проекта.

фотография различных вариантов питания, которые можно использовать в этом проекте.

фотография переделанного программатора, который используется для загрузки кода на ESP8266.

Код проекта
```
#include <Wire.h>
#include "SparkFunCCS811.h"
#include <Adafruit_Sensor.h>
#include <Adafruit_AHTX0.h>
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <WiFiUdp.h>
#include <NTPClient.h>


#define CCS811_ADDR 0x5A
CCS811 mySensor(CCS811_ADDR);
Adafruit_AHTX0 aht;
ESP8266WebServer server(80);
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP);


// Добавим два новых массива для хранения последних 30 значений каждого датчика
float last30CO2[30];
float last30TVOC[30];
float last30Temp[30];
float last30Hum[30];
int idx = 0; // Изменили имя переменной с 'index' на 'idx'
void handleSensor() {
  sensors_event_t humidity, temp;
  aht.getEvent(&humidity, &temp);
  mySensor.setEnvironmentalData(humidity.relative_humidity, temp.temperature);
  mySensor.readAlgorithmResults();
  // Добавим новые значения в массивы
  last30CO2[idx] = mySensor.getCO2();
  last30TVOC[idx] = mySensor.getTVOC();
  last30Temp[idx] = temp.temperature;
  last30Hum[idx] = humidity.relative_humidity;
  idx = (idx + 1) % 30; // Обновим индекс, чтобы он всегда был в диапазоне 0-29
  // Вычислим средние значения
  float avgCO2 = 0, avgTVOC = 0, avgTemp = 0, avgHum = 0;
  for (int i = 0; i < 30; i++) {
    avgCO2 += last30CO2[i];
    avgTVOC += last30TVOC[i];
    avgTemp += last30Temp[i];
    avgHum += last30Hum[i];
  }
  avgCO2 /= 30;
  avgTVOC /= 30;
  avgTemp /= 30;
  avgHum /= 30;
  String json = "{";
  json += "\"co2\": " + String(mySensor.getCO2()) + ",";
  json += "\"tvoc\": " + String(mySensor.getTVOC()) + ",";
  json += "\"temperature\": " + String(temp.temperature) + ",";
  json += "\"humidity\": " + String(humidity.relative_humidity) + ",";
  json += "\"avgCO2\": " + String(avgCO2) + ",";
  json += "\"avgTVOC\": " + String(avgTVOC) + ",";
  json += "\"avgTemp\": " + String(avgTemp) + ",";
  json += "\"avgHum\": " + String(avgHum);
  json += "}";
  server.send(200, "application/json", json);
}


void handleRoot() {
  sensors_event_t h, t; aht.getEvent(&h, &t); mySensor.setEnvironmentalData(h.relative_humidity, t.temperature);
  String html = "<!DOCTYPE html><html><head><meta charset=\"UTF-8\"><script src=\"https://cdn.jsdelivr.net/npm/chart.js\"></script><style>body { font-family: Arial, Helvetica, sans-serif; background-color: #f0f0f0; } h1 { color: #333; } p { color: #666; } .chart-container { display: inline-block; width: 50%; }</style></head><body><h1>Данные с датчиков</h1><div class=\"chart-container\"><canvas id=\"chart1\"></canvas><p id=\"avgCO2\"></p><p id=\"lastCO2\"></p></div><div class=\"chart-container\"><canvas id=\"chart2\"></canvas><p id=\"avgTVOC\"></p><p id=\"lastTVOC\"></p></div><div class=\"chart-container\"><canvas id=\"chart3\"></canvas><p id=\"avgTemp\"></p><p id=\"lastTemp\"></p></div><div class=\"chart-container\"><canvas id=\"chart4\"></canvas><p id=\"avgHum\"></p><p id=\"lastHum\"></p></div><script>function createChart(ctx, label) { return new Chart(ctx, { type: 'line', data: { labels: [], datasets: [{ label: label, data: [], borderColor: 'rgba(75, 192, 192, 1)', borderWidth: 1 }] }, options: { scales: { y: { beginAtZero: true } } } }); } var chart1 = createChart(document.getElementById('chart1').getContext('2d'), 'CO2'); var chart2 = createChart(document.getElementById('chart2').getContext('2d'), 'TVOC'); var chart3 = createChart(document.getElementById('chart3').getContext('2d'), 'Temperature'); var chart4 = createChart(document.getElementById('chart4').getContext('2d'), 'Humidity'); setInterval(function() { fetch('/sensor').then(response => response.json()).then(data => { var time = new Date().toLocaleTimeString(); chart1.data.labels.push(time); chart1.data.datasets[0].data.push(data.co2); chart1.update(); document.getElementById('avgCO2').innerText = 'Среднее CO2 за последние 30 минут: ' + data.avgCO2; document.getElementById('lastCO2').innerText = 'Последнее значение CO2: ' + data.co2; chart2.data.labels.push(time); chart2.data.datasets[0].data.push(data.tvoc); chart2.update(); document.getElementById('avgTVOC').innerText = 'Среднее TVOC за последние 30 минут: ' + data.avgTVOC; document.getElementById('lastTVOC').innerText = 'Последнее значение TVOC: ' + data.tvoc; chart3.data.labels.push(time); chart3.data.datasets[0].data.push(data.temperature); chart3.update(); document.getElementById('avgTemp').innerText = 'Средняя температура за последние 30 минут: ' + data.avgTemp; document.getElementById('lastTemp').innerText = 'Последнее значение температуры: ' + data.temperature; chart4.data.labels.push(time); chart4.data.datasets[0].data.push(data.humidity); chart4.update(); document.getElementById('avgHum').innerText = 'Средняя влажность за последние 30 минут: ' + data.avgHum; document.getElementById('lastHum').innerText = 'Последнее значение влажности: ' + data.humidity; }); }, 1000);</script></body></html>";
  server.send(200, "text/html", html);
}






void setup() {
  Serial.begin(57600);
  Wire.begin(2, 0);
  WiFi.config(IPAddress(192, 168, 1, 119), IPAddress(192, 168, 137, 1), IPAddress(255, 255, 255, 0));
  WiFi.begin("Redmi 10", "btd56c2fma49mt3");
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);


  }
  timeClient.begin();
  timeClient.setTimeOffset(10800);
  if (!aht.begin() || !mySensor.begin()) {
    Serial.println("Ошибка инициализации датчика");
  }
  Serial.println(WiFi.localIP());


  server.on("/", handleRoot);
  server.on("/sensor", handleSensor);
  server.begin();
}


void loop() {
  timeClient.update();
  server.handleClient();
}
```



Описание кода:

Этот код отвечает за функционирование многофункционального датчика. Вот некоторые ключевые аспекты:

- Обновление данных датчика: Код обновляет данные с датчиков и отображает их на веб-странице. Это включает в себя последние и средние значения для каждого из датчиков.

- Графики: Код также обновляет графики на веб-странице в реальном времени, позволяя наглядно отслеживать изменения показателей.

- Настройка WiFi: Код включает настройку WiFi для подключения к интернету и отправки данных на веб-страницу.

- Обработка запросов: Код обрабатывает HTTP-запросы, позволяя обновлять веб-страницу с новыми данными от датчиков.

- Цикл обработки: В основном цикле кода (loop) обновляются данные времени и обрабатываются клиентские запросы.


Веб-интерфейс

Датчик создает веб-страницу, на которой отображаются графики и показания датчиков в реальном времени. Это обеспечивает удобный и наглядный способ мониторинга состояния окружающей среды.

Применение

Этот проект может быть использован в различных областях, включая, но не ограничиваясь:

- Мониторинг качества воздуха в помещениях.
- Исследования в области экологии.
- Образовательные цели.

Планы

- Заменить модуль углекислого газа на Senseair S8
- Внедрить датчик в систему умный дом.
- Добавить домен для датчика.Или статический ip при любом способе подключения. Сделать удобный ввод пароля/логина wifi.

Заключение

Этот проект представляет собой мощный инструмент для мониторинга и анализа состояния окружающей среды. С его помощью можно получить точные и актуальные данные о состоянии воздуха, что позволяет принимать обоснованные решения и проводить эффективные исследования. Благодаря встроенному веб-интерфейсу, данные легко интерпретировать и анализировать. 

Проект является отличным примером применения технологий интернета вещей для решения актуальных задач. Он демонстрирует, как можно использовать современные технологии для создания полезных и эффективных систем мониторинга.

TELEGRAM: @MX_IDEA // ПО ВСЕМ ВОПРОСАМ

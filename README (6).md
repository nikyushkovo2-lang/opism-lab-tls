# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ | Юшков Микита Олександрович |
| Група | 21 |
| Ідентифікатор | se-21-08 |
| Дата виконання | 24.11.2026 |
| Середовище | GitHub Codespaces |
| Версія OpenSSL | Не зафіксовано; уточнити командою `openssl version` у контейнері |

## Параметри варіанта

| Параметр | Значення |
|---|---|
| Віддалений сервер | 10.53.0.80 (gate.s08.lab) |
| Імена для сертифіката | www.s08.lab, portal.s08.lab |
| Ваш сервер (IP) | 10.53.1.1 |

Перезапуск стенда під час експериментів не зафіксовано.

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1 (фрагмент дампа):


```text
19:04:10.932236 IP 10.53.1.1.43506 > 10.53.0.80.80: Flags [P.], seq 1:76, ack 1, win 502, options [nop,nop,TS val 2772931495 ecr 502637482], length 75: HTTP: GET / HTTP/1.1
.G......GET / HTTP/1.1
Host: gate.s08.lab
server: web
Host: gate.s08.lab
```


Вивід кроку 1.2: пошук GET /, Host: та server: web не повернув рядків. Кількість входжень імені — 1. Відповідь curl за фото:


```text
server: web
Host: gate.s08.lab
SNI: gate.s08.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
```


| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| GET / | Так | Ні |
| Host | Так | Ні |
| server: web | Так | Ні |
| Кількість входжень імені | 2 | 1 |

1. Ім’я передається в ClientHello, у розширенні server_name (SNI).
2. HTTP не використовує TLS, тому SNI відсутній.
3. Протокол HTTP/2 узгоджено через ALPN під час TLS-рукостискання.

## Завдання 2. Рукостискання по записах

Вивід 2.1 (TLS 1.3):


```text
Sent Record
  Content Type = Handshake (22)
  Length = 327
    ClientHello, Length=323
Received Record
  Content Type = Handshake (22)
  Length = 122
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
  Length = 1
Received Record
  Content Type = ApplicationData (23)
  Length = 36
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Length = 995
  Inner Content Type = Handshake (22)
    Certificate, Length=974
Received Record
  Content Type = ApplicationData (23)
  Length = 97
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=76
Received Record
  Content Type = ApplicationData (23)
  Length = 69
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
  Length = 1
Sent Record
  Content Type = ApplicationData (23)
  Length = 69
  Inner Content Type = Handshake (22)
    Finished, Length=48
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)
Sent Record
  Content Type = ApplicationData (23)
  Length = 19
  Inner Content Type = Alert (21)
```


Вивід 2.2 (TLS 1.2):


```text
Sent Record
  Content Type = Handshake (22)
  Length = 222
    ClientHello, Length=218
Received Record
  Content Type = Handshake (22)
  Length = 78
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
  Length = 973
    Certificate, Length=969
Received Record
  Content Type = Handshake (22)
  Length = 115
    ServerKeyExchange, Length=111
Received Record
  Content Type = Handshake (22)
  Length = 4
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
  Length = 37
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
  Length = 1
Sent Record
  Content Type = Handshake (22)
  Length = 40
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
  Length = 218
    NewSessionTicket, Length=214
Received Record
  Content Type = ChangeCipherSpec (20)
  Length = 1
Received Record
  Content Type = Handshake (22)
  Length = 40
    Finished, Length=12
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES256-GCM-SHA384
ALPN protocol: h2
    Verify return code: 0 (ok)
Received Record
  Content Type = ApplicationData (23)
  Length = 64
Sent Record
  Content Type = Alert (21)
  Length = 26
```


| № | Напрям | Content Type | Inner Content Type | Повідомлення | Length (запису) |
|---|---|---|---|---|---:|
| 1 | К→С | Handshake | — | ClientHello | 327 |
| 2 | С→К | Handshake | — | ServerHello | 122 |
| 3 | С→К | ChangeCipherSpec | — | ChangeCipherSpec | 1 |
| 4 | С→К | ApplicationData | Handshake | EncryptedExtensions | 36 |
| 5 | С→К | ApplicationData | Handshake | Certificate | 995 |
| 6 | С→К | ApplicationData | Handshake | CertificateVerify | 97 |
| 7 | С→К | ApplicationData | Handshake | Finished | 69 |
| 8 | К→С | ChangeCipherSpec | — | ChangeCipherSpec | 1 |
| 9 | К→С | ApplicationData | Handshake | Finished | 69 |
| 10 | К→С | ApplicationData | Alert | Alert | 19 |

Вивід 2.3 і 2.4:


```text
extension_type=server_name(0), length=17
          0000 - 00 0f 00 00 0c 67 61 74-65 2e 73 30 38 2e 6c   .....gate.s08.l
          000f - 61 62                                          ab
TLS 1.3: 0
TLS 1.2: 2
```


| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello (повідомлення), байт | 323 | 218 |
| Перший запис із прихованим внутрішнім типом | 4 | Внутрішній тип не приховується таким способом |
| RTT до першого HTTP-запиту | 1 | 2 |
| Сертифікат видно в дампі | Ні | Так |
| ALPN | h2 | h2 |

1. Починаючи із запису 4, зовнішній тип ApplicationData, внутрішній Handshake. Спостерігач бачить лише зовнішній тип.
2. TLS 1.3 містить додаткові розширення, зокрема key_share, що дозволяє одразу передати параметри обміну ключами.
3. 2 байти довжини списку + 1 байт типу імені + 2 байти довжини імені + 12 байтів gate.s08.lab = 17 байтів.
4. ChangeCipherSpec потрібний для сумісності зі старими мережевими пристроями.

## Завдання 3. Сертифікат для власного сервера

Наведені нижче фрагменти відновлено з фотографій термінала; повні журнали завдання 3 в архіві відсутні.

Вивід 3.1 і 3.2:


```text
subject=CN = Pebble Root CA 44c0e5
notBefore=Nov 24 18:59:35 2026 GMT
notAfter=Nov 24 18:59:35 2056 GMT
curl: (60) SSL certificate problem: self-signed certificate
```


Вивід 3.3 (фрагменти):


```text
[www.s08.lab] acme: use http-01 solver
[portal.s08.lab] acme: use http-01 solver
[www.s08.lab] The server validated our request
[portal.s08.lab] The server validated our request
[www.s08.lab, portal.s08.lab] acme: Validations succeeded; requesting certificates
[www.s08.lab] Server responded with a certificate.
```


У журналі nginx видно GET /.well-known/acme-challenge/… HTTP/1.1 зі статусом 200 для обох перевірок (токени скорочено).

Вивід 3.4:


```text
subject=CN = www.s08.lab
issuer=CN = Pebble Intermediate CA 3275d9
notBefore=Nov 24 19:17:08 2026 GMT
notAfter=Nov 24 19:17:07 2031 GMT
X509v3 Subject Alternative Name:
    DNS:www.s08.lab, DNS:portal.s08.lab
subject=CN = Pebble Intermediate CA 3275d9
issuer=CN = Pebble Root CA 44c0e5
```


Вивід 3.5:


```text
Certificate chain
 0 s:CN = www.s08.lab
   i:CN = Pebble Intermediate CA 3275d9
Verify return code: 21 (unable to verify the first certificate)
```


Вивід 3.6:


```text
server: s08 (https)
Host: www.s08.lab
SNI: www.s08.lab
protocol: TLSv1.3
server: s08 (https)
Host: portal.s08.lab
SNI: portal.s08.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.s08.lab
   i:CN = Pebble Intermediate CA 3275d9
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
 1 s:CN = Pebble Intermediate CA 3275d9
   i:CN = Pebble Root CA 44c0e5
Verify return code: 0 (ok)
```


| Параметр | Значення |
|---|---|
| Видавець | Pebble Intermediate CA 3275d9 |
| Видавець видавця | Pebble Root CA 44c0e5 |
| SAN | www.s08.lab, portal.s08.lab |
| Строк дії | 24.11.2026 19:17:08 — 24.11.2031 19:17:07 GMT |
| Ключ і підпис | EC 256 біт; RSA-SHA256 |
| Кількість сертифікатів 3.5 / 3.6 | 1 / 2 |

1. ACME-сервер Pebble перевірив обидва імені через HTTP-01; підтвердження — The server validated our request і HTTP 200 у журналі nginx.
2. Сервер не передавав проміжний сертифікат. Після додавання його до ланцюжка перевірка пройшла.
3. id-ecPublicKey — тип відкритого ключа сервера; RSA-SHA256 — алгоритм підпису сертифіката видавцем. Це різні ключі.
4. Якщо після перезапуску Pebble створить новий корінь, старий ланцюжок не перевірятиметься новим коренем. Потрібно оновити довіру та перевипустити сертифікат; аналог — заміна CA у реальній PKI.

## Завдання 4. Діагностика несправностей

Вивід 4.1:


```text
=== f1.s08.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f1.s08.lab
   i:CN = f1.s08.lab
   v:NotBefore: Nov 24 18:59:34 2026 GMT; NotAfter: Feb 22 18:59:34 2027 GMT
---
Verify return code: 18 (self-signed certificate)
=== f2.s08.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f2.s08.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: May  8 18:59:34 2026 GMT; NotAfter: Nov  4 18:59:34 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 24 18:59:34 2026 GMT; NotAfter: Nov 23 18:59:34 2031 GMT
---
Verify return code: 10 (certificate has expired)
=== f3.s08.lab
server: web, f3
Certificate chain
 0 s:CN = f3.s08.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 24 18:59:34 2026 GMT; NotAfter: Feb 22 18:59:34 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 24 18:59:34 2026 GMT; NotAfter: Nov 23 18:59:34 2031 GMT
---
Verify return code: 0 (ok)
=== f4.s08.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f4.s08.lab
   i:O = Foreign Org, CN = Foreign Intermediate CA
   v:NotBefore: Nov 24 18:59:34 2026 GMT; NotAfter: Feb 22 18:59:34 2027 GMT
 1 s:O = Foreign Org, CN = Foreign Intermediate CA
   i:O = Foreign Org, CN = Foreign Root CA
   v:NotBefore: Nov 24 18:59:34 2026 GMT; NotAfter: Nov 23 18:59:34 2031 GMT
---
Verify return code: 20 (unable to get local issuer certificate)
=== f5.s08.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f5.s08.lab'
Certificate chain
 0 s:CN = legacy.s08.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 24 18:59:34 2026 GMT; NotAfter: Feb 22 18:59:34 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 24 18:59:34 2026 GMT; NotAfter: Nov 23 18:59:34 2031 GMT
---
Verify return code: 0 (ok)
```


| Сервер | Помилка curl | Verify return code | Перевірка | Доказ | Виправлення |
|---|---|---|---|---|---|
| f1 | self-signed certificate | 18 | Довіра | s:CN = i:CN = f1.s08.lab | Сертифікат довіреного CA |
| f2 | certificate has expired | 10 | Строк дії | NotAfter: Nov 4 2026 | Оновити сертифікат |
| f3 | Немає | 0 | Усі пройшли | server: web, f3 | Не потрібно |
| f4 | unable to get local issuer certificate | 20 | Довіра до CA | Foreign Root CA | Сертифікат довіреного CA |
| f5 | no alternative certificate subject name matches | 0 | Ім’я | CN = legacy.s08.lab; помилка curl | Додати f5.s08.lab до SAN нового сертифіката |

1. f4 надсилає проміжний сертифікат Foreign Intermediate CA, але його корінь не є довіреним. При відсутньому проміжному сертифікаті в ланцюжку бракувало б відповідного запису.
2. openssl s_client без -verify_hostname перевіряє ланцюжок, але не відповідність імені; curl перевіряє обидва.
3. Очікується відповідь усіх п’яти серверів із curl -k, але це не перевірено окремо. Вимикається перевірка автентичності сервера, тому захист від підміни втрачається.

## Завдання 5. Ціна нового з’єднання

Прогноз (теоретичний, відновлений після вимірювань; без DNS):

| Етап | TLS 1.3 | TLS 1.2 | Повторне використання |
|---|---|---|---|
| time_connect | 1 RTT | 1 RTT | 0 |
| time_appconnect | 2 RTT | 3 RTT | 0 |
| time_starttransfer | 3 RTT | 4 RTT | 1 RTT |

Вивід 5.2 і 5.3:

t5-13.txt


```text
0.070903 0.106106 0.144952 0.180300 0.180361 2 1
0.036589 0.071696 0.109858 0.145164 0.145215 2 1
0.035408 0.070516 0.109641 0.144885 0.144939 2 1
```


t5-12.txt


```text
0.035485 0.070613 0.145548 0.181015 0.181069 2 1
0.035422 0.070539 0.144911 0.180208 0.180271 2 1
0.037499 0.072748 0.146729 0.182032 0.182084 2 1
```


t5-reuse.txt


```text
0.035533 0.070671 0.109595 0.144973 0.145029 2 1
0.000011 0.000000 0.000000 0.035268 0.035324 2 0
0.000013 0.000000 0.000000 0.035283 0.035350 2 0
```


| Запит | namelookup | connect | appconnect | starttransfer | total | num_connects |
|---|---:|---:|---:|---:|---:|---:|

| TLS 1.3, нове | 36.589 | 71.696 | 109.858 | 145.164 | 145.215 | 1 |

| TLS 1.2, нове | 35.485 | 70.613 | 145.548 | 181.015 | 181.069 | 1 |

| 5.3, перший | 35.533 | 70.671 | 109.595 | 144.973 | 145.029 | 1 |

| 5.3, другий | 0.011 | 0.000 | 0.000 | 35.268 | 35.324 | 0 |

| 5.3, третій | 0.013 | 0.000 | 0.000 | 35.283 | 35.350 | 0 |


RTT ≈ 35,3 мс за time_starttransfer повторних запитів (35,268 та 35,283 мс).

1. Значення у RTT (namelookup, connect, appconnect, starttransfer, total):


- TLS 1.3: 1.04, 2.03, 3.11, 4.11, 4.11.

- TLS 1.2: 1.01, 2.00, 4.12, 5.13, 5.13.

- Перший: 1.01, 2.00, 3.10, 4.11, 4.11.

- Другий: 0.00, 0.00, 0.00, 1.00, 1.00.

- Третій: 0.00, 0.00, 0.00, 1.00, 1.00.


Час curl накопичувальний, включає DNS; відхилення спричинені обробкою, плануванням та коливаннями затримки. Перший DNS-запит TLS 1.3 тривав близько 71 мс.

2. Навіть кешований запис потребує мережевого обміну з резолвером. За додаткового послідовного DNS-обміну час може подвоїтися.
3. Без DNS: TLS 1.3 — 200 × 3 × 35,3 = 21 180 мс; TLS 1.2 — 200 × 4 × 35,3 = 28 240 мс; одне TLS 1.3 з’єднання — (3 + 199) × 35,3 = 7130,6 мс. Якщо кожне нове з’єднання також потребує одного DNS RTT, оцінки становлять 28,24 с та 35,30 с, а для одного з’єднання — 7,1659 с.

## Висновки

**Що виявилося несподіваним.** Можливий висновок для підтвердження студентом: f5 має Verify return code: 0 (ok), але curl відхиляє його через невідповідність імені.

**Яке питання залишилося без відповіді.** Чому перший DNS-запит TLS 1.3 тривав 70,903 мс, а наступні близько 35 мс? Вимірювання це підтверджують, але окремий DNS-дамп не збирався, тому точна причина не встановлена. Студенту слід підтвердити, що це відповідає його спостереженням.

## Декларація використання ШІ

Використовувалися інструменти ШІ: так.

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | ChatGPT | Що зробити треба? | Допомога з послідовністю виконання; команди виконано студентом |
| 2 | ChatGPT | Надіслано бланк звіту та архів results.tar.gz без окремого текстового запиту | Чернетка звіту, таблиці, пояснення та розрахунки за результатами стенда |

Декларацію необхідно доповнити всіма іншими фактичними зверненнями, які вплинули на звіт, із дослівними промптами, та зазначити власні зміни після перевірки.

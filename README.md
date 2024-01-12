﻿# Описание API панели механика

В документе будут отсылки [на дизайн механика](https://www.figma.com/file/1TV9mhpqnHOcwronXVoMGV/%D0%A2%D0%B8%D0%BA%D0%B0%D0%BC%D0%B8%D1%81-%D0%BF%D1%80%D0%BE%D1%82%D0%BE%D1%82%D0%B8%D0%BF?type=design&node-id=124-2&mode=design).

Общение с API сервера осуществляется с помощью JSON. И запросы к серверу и ответы от него будут в этом формате.

Если нужен swagger для динамического отображения формата запросов-ответов и ожидаемых статус кодов, то он находится по этому пути:
`:9443/swagger/index.html`

## Аутентификация
Конечная цель аутентификации - получить access и refresh токены авторизации.  
  
Шаги следующие:  
1. Директор на панели механика вводит свои логин и пароль. (У механиков нет своих паролей).  
2. На следующем экране вводится номер поста, за которым физически располагается панель.  
3. По нажатию на кнопку "войти", на сервер отправляется **POST** запрос по пути `/mechanic-api/authenticate-panel`.  
   Тело запроса должно содержать логин, пароль и номер поста.  
   Пример:  
   ```json  
   {
       "login": "surgut_aero",
       "password": "password1",
       "postNumber": 4
   }
   ```  
4. Если данные введены верно, сервер возвращает ответ со статусом **200** со следующим телом ответа:
   ```json
   {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjEiLCJqdGkiOiJTZVoxVGh4V3BPRTciLCJ0eXBlIjoiYWNjZXNzIiwiZXhwIjoxNzAzNjg5MjM4fQ.yru2tyj1JLs-h69XSk-JuERVjcOC5oAmpbmtVTrITlc",
      "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjEiLCJqdGkiOiJTZVoxVGh4V3BPRTciLCJ0eXBlIjoicmVmcmVzaCIsImV4cCI6NDg1OTM0ODQzOH0.cePj3rH8-EO25oXtlXx9kucY2XLN8WX0G-huFt_arhw"
   }  
   ```  
   Механика перекидывает на экран "экран открытия поста" (экраны есть в фигме, по ссылке в начале).  
   Если данные введены неверно, возвращается ответ со статусом **401**.  
5. Далее в каждом запросе в заголовке `Authorization` необходимо передавать access токен с префиксом *Bearer* в таком виде:  
`Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjEiLCJqdGkiOiJTZVoxVGh4V3BPRTciLCJ0eXBlIjoiYWNjZXNzIiwiZXhwIjoxNzAzNjg5MjM4fQ.yru2tyj1JLs-h69XSk-JuERVjcOC5oAmpbmtVTrITlc`
6. Механика перекидывает на экран 

## Обновление access токена
Так как у access токена ограниченное время действия, когда оно истечет, все запросы к серверу будут возвращаться с ответом **401**.  

Чтобы обновить токен, нужно отправить **POST** запрос по пути `mechanic-api/refresh`.  
В заголовок `Authorization` необходимо добавить Refresh токен с тем же префиксом *Bearer*.  

Если обновление токена произошло успешно, возвращается ответ **200**.  
Если refresh токен не подошёл, возвращается ответ **401**, тогда необходимо, чтобы сайт выкинул механика из учетной 
записи и снова показал страницу с логином и паролем.  

> [!CAUTION]  
> На сайте будут места, где отправляется одновременно несколько запросов. И если в этот момент access токен истечет, и 
> оба запроса вызовут метод `mechanic-api/refresh`, то один обновит токены, а другой словит ответ 401 от сервера и деавторизует механика.  
> Это *race condition* и его можно исправить буквально в две строчки. Например, [по инструкции](https://fuse8.ru/articles/how-to-avoid-race-condition).

## Выбор механика, `mechanic-api/list` и `mechanic-api/mechanic-login`
Предполагается, что механик никогда не будет выходить из своей учетной записи, но в рамках этой учетной записи
механики могут сменять друг друга.  
В идеальных условиях, на этой панели больше никогда не нужно будет вводить логин, пароль, номер поста на экранах "Вход" и "Выбор номера поста".  
В конце рабочей смены механик нажимает кнопку "ВЫХОД" (кнопка показана на экране "добавление услуги"), 
что означает НЕ выход из учетной записи, а конец работы этого механика.  
Тогда его перекидывает на экран "Экран открытия поста", где другой механик в следующую смену сможет выбрать себя.  

Чтобы запросить у сервера список механиков, которые могут сегодня работать: 
1. Высылается **GET** запрос по пути `mechanic-api/list` без тела запроса.  
2. Формат ответа сервера:
   ```json
   {
     "mechanics": [
       {
         "id": "ПБ-98329872",
         "name": "Гирасимов Михаил Павлович"
       },
       {
         "id": "ПБ-10203472",
         "name": "Гирасимов Михаил Павлович"
       }
     ]
   }
   ```
3. После выбора механика и нажатия кнопки "подтвердить и продолжить", необходимо вызвать **POST** метод `mechanic-api/mechanic-login` со следующим телом запроса:  
   ```json
   {
     "mechanicId": "ПБ-98329872"
   }
   ```
   Может вернуться ответ **200**, или **403**, если этот механик уже занят на другом посте или переданные данные устарели.  
4. Далее механика перекидывает на "Экран приема заказа", где, чтобы наполнить его данными, необходимо вызывать метод `mechanic-api/order/get-next`

## Выбор номера поста из POST метода `mechanic-api/get-posts`
В теле запроса передаются логин и пароль директора, и по ним в ответе присылается список доступных для логина постов в виде:
```json
{
  "posts": [
    1,
    2,
    3,
    4
  ]
}
```
Тело запроса должно выглядеть так:
```json
{
  "login": "string",
  "password": "string"
}
```

Может прийти ответ **401**, если логин админа введён неверно.
## Ответ от сервера 409 Conflict
Если случилась какая-то непредсказуемая проблема, о которой нужно уведомить пользователя (в данном случае механика), то сообщение
об этой проблеме присылается сервером со статус кодом **409**.

Ответ такого типа может прийти от ЛЮБОГО запроса, поэтому на фронтенде любой ответ от сервера с кодом **409** нужно как-то
обрабатывать и показывать сообщение пользователю в теле ответа.  
В отличие от всех остальных ответов сервера, этот приходит не в виде json, а в виде строки.  
Можно обрабатывать сообщения со статус кодом 409, например таким образом:
```js
let response = await fetch(URL+'/mobile-api/customer/qr-code');
if (response.status == 409)
   alert(await response.text());
// Появляется сообщение для пользователя
```

### POST `mechanic-api/order/get-next`
На "Экране приема заказа" необходимо примерно каждую минуту вызывать этот запрос, чтобы отображались всегда были актуальные данные.  
Данные могут поменяться, если, например, пользователь отменит заказ, или появится заказ ближе, чем отображаемый ранее.  
Тела запроса нет.  
Могут быть следующие варианты статуса кода ответа:  

| Код ответа     | Описание                                                                                                                                                                                                                                          |
|----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 200 OK         | Заказ существует, в теле ответа данные                                                                                                                                                                                                            |
| 204 No content | Заказов на сегодня пока нет                                                                                                                                                                                                                       |
| 409 Conflict   | Произошла какая-то ошибка, сообщение о которой нужно <br/> отобразить механику. Подробнее в "Ответ от сервера 409 Conflict". Во всех следующих таблицах <br/> не будет дублироваться этот вариант ответа, но он может приходить от любого запроса |

Если статус код **200**, то тело ответа будет таким:  
   ```json
   {
     "orderId": 20000382,
     "plateNumber": "AA000A",
     "startTime": "13:20",
     "startDate": "26 октября, четверг",
     "completionTimeHours": 1.3,
     "works": [
       {
         "id": "БП-874923",
         "name": "Проверка тормозных шлангов"
       },
       {
          "id": "БП-98091",
          "name": "Замена масла"
       }
     ]
   }
   ```
В поле `completionTimeHours` содержится дробное значение в часах, за сколько по нормативу заказ должен быть завершён.

### DELETE `mechanic-api/order/work`
Перед началом заказа и во время его выполнения, механик может удалить некоторые работы из заказа (иконка корзины на "Экране приема заказа").

Тело запроса должно выглядеть так:
```json
{
  "orderId": 20000382,
  "workId": "БП-874923"
}
```

Если сервер вернул ответ со статусом **200**, то тело ответа будет пустым.  

### POST `mechanic-api/order/work`
Добавление работы в заказ. Механик может добавить услугу по нажатию на кнопку "Добавить услугу".  
Тело запроса такого формата:
```json
{
  "orderId": 20000382,
  "workIds": [
    "БП-874923",
    "БП-98091"
  ]
}
```

Тело ответа будет пустым при успешном добавлении работ.

### GET `mechanic-api/work/list`
Получить список актуальных работ, их названий и цен можно получить с помощью этого метода. Тело ответа будет таким:
```json
{
  "works": [
    {
      "workId": "БП-874923",
      "name": "Проверка стоек и амортизаторов",
      "price": 1500
    }
  ]
}
```

### POST `mechanic-api/order/cancel`
Пока механик ещё не начал заказ, он может отменить заказ и перейти к следующему.  
Формат тела запроса:
```json
{
  "orderId": 20000382
}
```
В случае успешной отмены заказа, тело ответа будет пустым.  
Механика перебрасывает на "Экран приема заказа", и для поиска, где снова необходимо запрашивать данные о следующем заказе через метод `mechanic-api/order/get-next`.

### POST `mechanic-api/order/start`
По нажатию на кнопку "Начать выполнение" вызывается этот метод, что сервер запомнил, что механик работает над этим заказом.  
Тело запроса:
```json
{
  "orderId": 20000382
}
```  
В случае успеха тело ответа будет пустым.

### POST `mechanic-api/order/complete`
Для завершения заказа механик обязательно должен указать пробег автомобиля клиента. Без этого кнопка завершения заказа должна быть неактивна.  

Тело запроса должно содержать только значение пробега. Сервер сам найдет привязанный к механику заказ и завершит его. Пример:  
```json
{
  "odometer": 12502
}
```

В ответе от сервера будет QR код, о котором будет написано подробно ниже. Формат ответа от сервера:
```json
{
   "result": "eyJJRCI6MjAwMDAwMDEsIkNBUiI6IkFBMDAxQSIsIlJFRyI6IjE4NiIsIk9ET01FVEVSIjoiMTI0NTQiLCJXT1JLUyI6W3si0KbQkS0wMDAwMjg4OSI6Miwi0KbQkS0wMDAwMjkwOSI6MSwi0KbQkS0wMDAwMjkyNyI6MX1dLCJIQVNIIjoiYzUwMWJkNDdiYzk4YTk5MGI0ZjkwMDljOGQ2NjBlNDllZTQ3MDhiYzNjNjE2ZjcxMzdhZGUyZDVkNmMxYzViMCJ9"
}
```

Данные нужно отрисовать в QR-код.  
После закрытия окна с кодом, механик должен возвращаться на "Экран приема заказа".

Логика работы: после завершения заказа, механика перекидывает на страницу "QR код".  
У клиента есть свой QR код в приложении, он должен будет предъявить его на кассе.  
Но если у него неполадки с телефоном, то у механика будет такой же QR код на экране после завершения работы.  

### POST `mechanic-api/mechanic-logout`
В конце рабочего дня механик завершает работу под своим именем. По нажатию на кнопку выход вызывается
этот метод, чтобы отвязать механика от этого поста.  
Тело запроса и ответа пустые.
Важно, что токены авторизации не удаляются, и следующему механику не нужно будет вводить логин и пароль заново.  
После выхода механика происходит только переход на "Экран открытия поста". 

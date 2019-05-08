---
layout: post
title: Получение результатов региональной выдачи у Google
tags: [Google, SEO, SEO инструменты, поиск по регионам, региональная выдача]
---

Не секрет, что поисковые системы уже давно используют систему ранжирования основанную на учёте места нахождения пользователя. Итоговая выдача на странице результатов поисковой машины по геозависимым запросам будут отличаться в зависимости от того, как поисковая система опредилила регион пользователя. При необходимости регион можно указать в настройках поиска:

![Настройки региона поиска у Google]({{ site.baseurl }}/images/google-region.png)

![Настройка региона поиска у Яндекс]({{ site.baseurl }}/images/yandex-region.png)

Но что делать, если вы используете AllSubmitter или пишете свой парсер поисковой выдачи? Изменить регион поиска у Яндекса достаточно просто. Нужно просто добавить GET-параметр *lr* к запросу. Значение — это числовой код региона из [списка](http://api.yandex.ru/xml/doc/dg/reference/regions.xml) доступного в документации к Яндекс.XML. У Google эта операция не так тривиальна.

Сменить регион поиска у Google можно так же добавив GET-параметр к строке запроса. Имя его *uule*, а вот с получением значения придётся повозиться. Значение строится по данной формуле:

```
prefix + lengthKey(canonicalName) + base64(canonicalName)
```

* *prefix* — префикс является постоянным и равен ‘w+CAIQICI’.

* *canonicalName* — полное название региона из [таблицы географических целей](https://developers.google.com/adwords/api/docs/appendix/geotargeting) у AdWords API.

* *lengthKey()* — ключ зависящий от длины canonicalName. Для большей наглядности и практичности приведу код функции:

  ```php
  function lengthKey($string) {
      $result = false;
      if (is_string($string)) {
          $keys = array();
          $keys = array_merge($keys, range('A', 'Z'));
          $keys = array_merge($keys, range('a', 'z'));
          $keys = array_merge($keys, range('0', '9'));
          $keys = array_merge($keys, array('-', ' '));
          $offset = strlen($string) % count($keys);
          $result = (isset($keys[$offset]))? $keys[$offset]: false;
      }
      return $result;
  }
  ```

* *base64()* — закодированная в base64 строка *canonicalName*.

Например, итоговое значение параметра при поиске в Москве (*Moscow,Moscow,Russia* исходя из данных Google Geographical Target):

```w+CAIQICIUTW9zY293LE1vc2NvdyxSdXNzaWE=```

При поиске в Санкт-Петербурге (*Saint Petersburg,Saint Petersburg,Russia*):

```w+CAIQICIoU2FpbnQgUGV0ZXJzYnVyZyxTYWludCBQZXRlcnNidXJnLFJ1c3NpYQ==```

Для того, чтобы убедится в правильности проделанных действий, можно сравнить результат поисковой выдачи по запросу ‘купить квартиру’ в [Москве](https://www.google.ru/search?q=%D0%BA%D1%83%D0%BF%D0%B8%D1%82%D1%8C%20%D0%BA%D0%B2%D0%B0%D1%80%D1%82%D0%B8%D1%80%D1%83&uule=w+CAIQICIUTW9zY293LE1vc2NvdyxSdXNzaWE=) и [Санкт-Петербурге](https://www.google.ru/search?q=%D0%BA%D1%83%D0%BF%D0%B8%D1%82%D1%8C%20%D0%BA%D0%B2%D0%B0%D1%80%D1%82%D0%B8%D1%80%D1%83&uule=w+CAIQICIoU2FpbnQgUGV0ZXJzYnVyZyxTYWludCBQZXRlcnNidXJnLFJ1c3NpYQ==).

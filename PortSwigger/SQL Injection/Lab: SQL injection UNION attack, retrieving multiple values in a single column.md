link: https://portswigger.net/web-security/learning-paths/sql-injection/sql-injection-retrieving-multiple-values-within-a-single-column/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column

Задание: This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.  
The database contains a different table called ```users```, with columns called ```username``` and ```password```.  
To solve the lab, perform a SQL injection UNION attack that retrieves all usernames and passwords, and use the information to log in as the ```administrator``` user.  

Сразу перенаправляем это все в burp suite и пробуем инъекцию в параметре category. Пробуем поставить ```'``` и проверить SQL инъекцию и как мы видим мы получили ошибку 500, а значит этот параметр уязвим к SQL.  

<img width="1230" height="713" alt="image" src="https://github.com/user-attachments/assets/c240cbd7-735b-4977-a9fc-6b438fcb5092" />  


Теперь нужно узнать количество столбцов, применяем ```'UNION SELECT NULL FROM users --``` -> снова получаем 500. Пробуем ```'UNION SELECT NULL, NULL FROM users --``` -> 200, отлично, значит в нашей задаче 2 столбца,теперь 
проверяем тип данных в столбцах, для этого применяем ```'UNION SELECT 'abc', NULL FROM users --``` -> 500, применяем ```'UNION SELECT NULL, 'abc' FROM users --``` -> 200, значит во втором столбце тип данных string и мы можем 
вытащить ник и пароль с помощью конкатенации ```'UNION SELECT NULL, username||'->'||password FROM users --``` и видим, что у нас на странице появились credentials:      


<img width="355" height="125" alt="image" src="https://github.com/user-attachments/assets/fe16c87b-61ef-4e5d-895e-d1479a78c07b" />  

Заходим под админом и лаба пройдена!

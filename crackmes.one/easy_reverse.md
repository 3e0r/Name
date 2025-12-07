**easy_reverse** - первая моя реверс задача，попробуем с ней разобраться и узнать пароль.
Сначала пробуем запустить ее в терминале:  
<img width="376" height="80" alt="image" src="https://github.com/user-attachments/assets/1c6c4ba9-abed-4226-b262-b9a064397a38" />  
Нужен пароль)  
Закидываем теперь этот бинарник в ```ghidra```, и находим функцию main, она выглядит так:  
<img width="889" height="535" alt="image" src="https://github.com/user-attachments/assets/153d243c-26f3-44ae-a643-99343b1bd3ce" />  
Для начала сделаем функцию стандартной, для этого возьмем ```int main(int argc, char *argv[])``` и вставим вместо нашей, но проблема в том, что  квадратные скобки аргумента v рассматривались не как массивы, а как часть нашего имени функции, поэтому нужно снова 
редактировать нашу функциональную сигнатуру на ```int main(int argc, char** argv)```, и как мы видим это намного сильно улучшило нашу визуализацию кода программы.  
<img width="878" height="582" alt="image" src="https://github.com/user-attachments/assets/95a3f894-f834-48c6-bfb5-f24e54b8553f" />  
Заметим, что ```sVar1``` - это длина первого аргумента. Поэтому можем его переименовать в ```arg1_length```.  
<img width="875" height="521" alt="image" src="https://github.com/user-attachments/assets/0dd95b56-ef82-4afa-a83b-5b6cd68a90b1" />  
Теперь это выглядит замечательно и следующей строкой замечаем, что 5-ый элемент должен быть **"@"**, пробуем))  
<img width="368" height="67" alt="image" src="https://github.com/user-attachments/assets/f1d304b2-5013-4089-9bf1-d3bfa8287906" />  
Отлично, задание выполнено!  
https://crackmes.one/crackme/5b8a37a433c5d45fc286ad83

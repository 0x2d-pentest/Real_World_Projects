# Skillfactory Permutation Brute-Force Vulnerability

## 🔍 Initialization

### Есть тест, предполагающий несколько вариантов правильных ответов.  
<div align="center">
  <img width="1875" height="934" alt="image" src="https://github.com/user-attachments/assets/4fe3ee0a-980a-485d-b90b-1546f7fe2757" />
</div>


## 🕵️ Enumeration

### Перехватываю отправляемый запрос в Burp Suite  
<div align="center">
  <img width="1916" height="902" alt="image" src="https://github.com/user-attachments/assets/757b4758-a9ab-4a0a-8cb2-0b8261f96189" />  
</div>  
  
  
#### Payload в URL encoded:  
```html
input_7e44eb6d37ea49b8844fa93a96ea014f_2_1[]=choice_0
```

#### Если ответ неправильный, то в `Response` возвращается `json`, содержащий `"success": "incorrect"`  

### Смотрю исходный код отмеченного чекбокса
<div align="center">  
  <img width="1875" height="929" alt="image" src="https://github.com/user-attachments/assets/32a54be2-f34c-4a89-965a-dc93c073440b" />  
</div>  

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

<div align="center">
  <img width="1918" height="900" alt="image" src="https://github.com/user-attachments/assets/1c0cd632-3829-4382-93b6-8875b7907269" />
</div> 


### Настраиваю вкладку `Settings` для `Intruder`.  
Добавляю остановку атаки, если в `Response` будет отсутствовать `"success": "incorrect"`.  
  

<div align="center">
  <img width="488" height="855" alt="image" src="https://github.com/user-attachments/assets/30fc9a4f-e1e8-409d-ab12-854d5fa5a19f" />
</div>  

### Также хочу отобразить дополнительную колонку в результатах вывода, с помощью `Grep - Extract`  

<div align="center">
  <img width="1178" height="880" alt="image" src="https://github.com/user-attachments/assets/4c82ce32-869f-4dac-a6ef-948c101be8f6" />
</div> 

### Сортирую результаты атаки по дополнительному столбцу и получаю правильную комбинацию ответов:  
<div align="center">
  <img width="1477" height="838" alt="image" src="https://github.com/user-attachments/assets/ecae7fe5-7289-4f81-9000-cdeb854b5046" />  
</div>  

## 🏁 Result
Если запускать атаку синхронно, то достаточно будет просто обновить страницу в браузере, вопрос автоматически будет отмечен, как правильно отвеченный.  
Но так как в данном случае атака происходила асинхронно, необходимо отметить правильные варианты и нажать **Отправить**  

<div align="center">
  <img width="1875" height="928" alt="image" src="https://github.com/user-attachments/assets/06ae3f70-882a-40ec-8ae7-2c372e851ca9" />
</div>

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

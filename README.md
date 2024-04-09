# Тестовое задание на стажировку AppSecCloudCamp
## 1. Вопросы для разогрева

1. Расскажите, с какими задачами в направлении безопасной разработки вы сталкивались?

   Защита информации на компьютере или сети компьютеров, в рамках университетского курса "Информационная безопасность и защита информации".
   Использовались такие средства на базе ОС Windows:
    - NTFS для назначения прав доступа для файлов и папок
    - Работа с командной строкой и написания bash-файлов
    - Работа с консолью администратора и управление правами пользователя и т.п
2. Если вам приходилось проводить security code review или моделирование угроз, расскажите, как это было?

   Опыта в проведении security code review и моделировании угроз нет
3. Если у вас был опыт поиска уязвимостей, расскажите, как это было?

   Есть опыт поиска уязвимостей, в основном в своем коде.
   При написании pet-проектов, когда разрабатывала какие-либо запросы к БД, старалась учесть возможные SQL-инъекциии.
   Так же использовала jwt-токенами в приложениях, где нужна была аутентификация пользователя, как способ защиты
4. Почему вы хотите участвовать в стажировке?

   Мне интересно развитаться в сфере безопасности приложений и анализировать код на какие-либо уязвимости. Получить опыт корпоративной разработки и работы в команде.

---

## 2. Security code review

### Часть 1. Security code review: GO

Анализ кода на GO с точки зрения безопасности и отчет по уязвимостям:
1. Хранение переменных для подключения к бд в коде
- Уязвимость находится в строке ```db, err = sql.Open("mysql", "user:password@/dbname") ```
- Через эту уязвимость может быть утечка кондиденциальной информации, которая храниться в бд. Так же будет сложно отследить у кого и когда был доступк базе данных.
- Пути решения:
   1) Создание отдельного конфигурационного файла для секретной информации и добавить его в .gitignore
   2) Использование переменных окружения. Второй вариант предпочтительнее т.к. не нужно хранить шаблон конфигурационного файла.
2. Возможность SQL-инъекции
- Уязвимость в строке ```query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", searchQuery) ```
- Эта уязвимость дапускает возможность SQL-инъекции, что дает возможность злоумышленнику, например изменить запрос и получить доступ к дополнительной информации
- Для исправлении уязвимости нужно использовать параметризированные запросы
3. Отсутствие проверки ввода пользователя
- Уязвимость в строке ``` searchQuery := r.URL.Query().Get("query") ```
- Производится проверка только на отсутствие данных, но нет проверки на некорректные или вредоносные данные, которые могут нанести вред системе
- Добавить валидацию ввода пользователя
4. Использование log.Fatal
- Можно обработать ошибки и не прекращать работу программы. При обработке ошибок, можно быстрее понять, где они происходят

```
package main

import (
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "github.com/go-sql-driver/mysql"
)

var db *sql.DB
var err error

func initDB() {
    db, err = sql.Open("mysql", "user:password@/dbname")
    if err != nil {
        log.Fatal(err)
    }

err = db.Ping()
if err != nil {
    log.Fatal(err)
    }
}

func searchHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != "GET" {
        http.Error(w, "Method is not supported.", http.StatusNotFound)
        return
    }

searchQuery := r.URL.Query().Get("query")
if searchQuery == "" {
    http.Error(w, "Query parameter is missing", http.StatusBadRequest)
    return
}

query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", searchQuery)
rows, err := db.Query(query)
if err != nil {
    http.Error(w, "Query failed", http.StatusInternalServerError)
    log.Println(err)
    return
}
defer rows.Close()

var products []string
for rows.Next() {
    var name string
    err := rows.Scan(&name)
    if err != nil {
        log.Fatal(err)
    }
    products = append(products, name)
}

fmt.Fprintf(w, "Found products: %v\n", products)
}

func main() {
    initDB()
    defer db.Close()

http.HandleFunc("/search", searchHandler)
fmt.Println("Server is running")
log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Часть 2: Security code review: Python

Анализ кода Python и отчет по уязвимостям:  
**Пример 2.1**  
Уязвимость в строке  
```output = Template('Hello ' + name + '! Your age is ' + age + '.').render()```  
Переменные пользовательского ввода в шаблоне (name, age) не очищины и не проверены на вредоносные данные перед конкатенацией.  
Может произойти внедрение зловредных данных.  
Для исправления уязвимости, нужно добавить проверку этих данных, и только потом производить конкатенацию  

**Пример 2.2**  
Уязвимость в строке  
```cmd = 'nslookup ' + hostname```  
Уязвимость командной инъекции из-за непосредственной конкатенации.  
Нужно очистить и проверить полученной значение hostname на безопасность  

**Пример №2.1**
```
from flask import Flask, request
from jinja2 import Template

app = Flask(name)

@app.route("/page")
def page():
    name = request.values.get('name')
    age = request.values.get('age', 'unknown')
    output = Template('Hello ' + name + '! Your age is ' + age + '.').render()
return output

if name == "main":
    app.run(debug=True)
```

**Пример №2.2**
```
from flask import Flask, request
import subprocess

app = Flask(name)

@app.route("/dns")
def dns_lookup():
    hostname = request.values.get('hostname')
    cmd = 'nslookup ' + hostname
    output = subprocess.check_output(cmd, shell=True, text=True)
return output
if name == "main":
    app.run(debug=True)
```

## 3. Моделировани угроз

Изучите диаграмму потоков данных (Data Flow Diagram, DFD) сервиса, обеспечивающего отправку информации в Telegram и Slack:

![DFD](https://github.com/appseccloudcamp/test-assignment/blob/main/test-dfd.png)

Краткое описание компонентов сервиса:
 - **User** - авторизованный пользователь системы. Может настраивать отправку уведомлений и загружать изображения для дальнейшего использования при отправке уведомлений;
 - **Microfront** - микрофронт, которые позволяет взаимодействовать с сервисом отправки информации;
 - **Backend application** - набор микросервисов реализующих бизнес-логику приложения и обеспечивающих взаимодействие со всеми внешними сервисами;
 - **Auth** - сервис отвечающий за аутентификацию и авторизацию клиентов сервиса отправки информации;
 - **S3** - объектное хранилище, предназначенное для хранения статического контента сервиса отправки информации;
 - **PostgreSQL** - база данных, предназначенная для хранения пользовательских конфигураций сервиса отправки информации.    

Проанализируйте диаграмму потоков данных приложения и ответьте на следующий вопросы:
 - Расскажите, какие потенциальные проблемы безопасности существуют для данного сервиса?
 - Расскажите, к каким последствиям может привести эксплуатация проблем, найденных вами?
 - Расскажите, какие способы исправления уязвимостей и смягчения рисков вы можете предложить по отмеченным вами проблемам безопасности?
 - Напишите список уточняющих вопросов, которые вы бы задали разработчикам данного сервиса?

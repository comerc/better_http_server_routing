# Улучшенная маршрутизация HTTP-серверов в Go 1.22

В Go 1.22 ожидается появление [интересного предложения](https://github.com/golang/go/issues/61410) - расширение возможностей по поиску шаблонов (pattern-matching) в мультиплексоре, используемом по умолчанию для обслуживания HTTP в пакете `net/http`.

Существующий мультиплексор ([http.ServeMux](https://pkg.go.dev/net/http#ServeMux)) обеспечивает рудиментарное сопоставление путей, но не более того. Это привело к появлению целой индустрии сторонних библиотек для реализации более мощных возможностей. Я рассматривал эти возможности в серии статей "REST-серверы на Go", в частях [1](https://eli.thegreenplace.net/2021/rest-servers-in-go-part-1-standard-library/) и [2](https://eli.thegreenplace.net/2021/rest-servers-in-go-part-2-using-a-router-package/).

Новый мультиплексор в версии 1.22 позволит значительно сократить отставание от пакетов сторонних разработчиков, обеспечив расширенное согласование. В этой небольшой заметке я кратко расскажу о новом мультиплексоре (mux). Я также вернусь к примеру из серии "REST-серверы на Go" и сравню, как новый stdlib mux справляется с `gorilla/mux`.

## Использование нового mux

Если вы когда-либо использовали сторонние пакеты mux/маршрутизаторов для Go (например, `gorilla/mux`), то использование нового стандартного mux будет простым и привычным. Начните с чтения [документации](https://pkg.go.dev/net/http@master#ServeMux) по нему - она краткая и понятная.

Давайте рассмотрим несколько базовых примеров использования. Наш первый пример демонстрирует некоторые из новых возможностей mux по сопоставлению шаблонов:

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /path/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "got path\n")
	})

	mux.HandleFunc("/task/{id}/", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id")
		fmt.Fprintf(w, "handling task with id=%v\n", id)
	})

	http.ListenAndServe("localhost:8090", mux)
}
```

Опытные программисты на Go сразу же заметят две новые особенности:

- В первом обработчике метод HTTP (в данном случае `GET`) указывается явно в составе шаблона. Это означает, что данный обработчик сработает только для `GET`-запросов к путям, начинающимся с `/path/`, а не для других HTTP-методов.
- Во втором обработчике во втором компоненте пути присутствует подстановочный знак - `{id}`, который ранее не поддерживался. Подстановочный знак будет соответствовать одному компоненту пути, и обработчик сможет получить доступ к найденному значению через метод `PathValue` запроса.

Поскольку Go 1.22 еще не выпущен, я рекомендую запускать этот пример с `gotip`. [Полный пример кода](https://github.com/eliben/code-for-blog/tree/master/2023/http-newmux-samples) с инструкциями по его выполнению. Давайте проверим работу этого сервера:

```bash
$ gotip run sample.go
```

А в отдельном терминале мы можем выполнить несколько вызовов curl для проверки:

```bash
$ curl localhost:8090/what/
404 page not found

$ curl localhost:8090/path/
got path

$ curl -X POST localhost:8090/path/
Method Not Allowed

$ curl localhost:8090/task/f0cd2e/
handling task with id=f0cd2e
```

Обратите внимание, что сервер отклоняет `POST`-запрос к `/path/`, в то время как `GET`-запрос (по умолчанию для `curl`) разрешен. Обратите также внимание на то, как подстановочный знак `id` получает значение при совпадении запроса. Еще раз рекомендую вам ознакомиться [с документацией по новому ServeMux](https://pkg.go.dev/net/http@master#ServeMux). Вы узнаете о таких дополнительных возможностях, как сопоставление путей с подстановочным символом `{id}...`, строгое сопоставление конца пути с `{$}` и другие правила.

Особое внимание в предложении было уделено возможным конфликтам между различными шаблонами. Рассмотрим такую схему:

```go
mux := http.NewServeMux()
mux.HandleFunc("/task/{id}/status/", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        fmt.Fprintf(w, "handling task status with id=%v\n", id)
})
mux.HandleFunc("/task/0/{action}/", func(w http.ResponseWriter, r *http.Request) {
        action := r.PathValue("action")
        fmt.Fprintf(w, "handling task 0 with action=%v\n", action)
})
```

А если сервер получит запрос на `/task/0/status/` - к какому обработчику он должен обратиться? Он соответствует обоим! Поэтому в новой документации по `ServeMux` тщательно описаны _правила старшинства_ для шаблонов, а также возможные конфликты. В случае конфликта регистрация впадает в панику. Действительно, для приведенного выше примера мы получаем что-то вроде:

```
panic: pattern "/task/0/{action}/" (registered at sample-conflict.go:14) conflicts with pattern "/task/{id}/status/" (registered at sample-conflict.go:10):
/task/0/{action}/ and /task/{id}/status/ both match some paths, like "/task/0/status/".
But neither is more specific than the other.
/task/0/{action}/ matches "/task/0/action/", but /task/{id}/status/ doesn't.
/task/{id}/status/ matches "/task/id/status/", but /task/0/{action}/ doesn't.
```

Сообщение подробное и полезное. Если мы столкнемся с конфликтами в сложных схемах регистрации (особенно когда паттерны регистрируются в нескольких местах исходного кода), то такие подробности будут очень ценны.

## Переделка моего сервера задач с новым mux

В серии статей "REST-серверы в Go" реализуется простой сервер для приложения задач/списка задач на Go, используя несколько различных подходов. [Часть 1](https://eli.thegreenplace.net/2021/rest-servers-in-go-part-1-standard-library/) начинается с "ванильного" подхода с использованием стандартной библиотеки, а [часть 2](https://eli.thegreenplace.net/2021/rest-servers-in-go-part-2-using-a-router-package/) переделывает тот же сервер с использованием маршрутизатора [gorilla/mux](https://github.com/gorilla/mux).

Сейчас самое время реализовать его еще раз, но уже с использованием улучшенного mux из Go 1.22; особенно интересно будет сравнить решение с использованием `gorilla/mux`.

Полный код этого проекта [доступен здесь](https://github.com/eliben/code-for-blog/tree/master/2021/go-rest-servers/stdlib-newmux). Рассмотрим несколько показательных примеров кода, начав с регистрации паттерна:

```go
mux := http.NewServeMux()
server := NewTaskServer()

mux.HandleFunc("POST /task/", server.createTaskHandler)
mux.HandleFunc("GET /task/", server.getAllTasksHandler)
mux.HandleFunc("DELETE /task/", server.deleteAllTasksHandler)
mux.HandleFunc("GET /task/{id}/", server.getTaskHandler)
mux.HandleFunc("DELETE /task/{id}/", server.deleteTaskHandler)
mux.HandleFunc("GET /tag/{tag}/", server.tagHandler)
mux.HandleFunc("GET /due/{year}/{month}/{day}/", server.dueHandler)
```

> Вы, наверное, заметили, что эти шаблоны не очень строги к частям пути, которые идут после интересующей нас части (например, `/task/22/foobar`). Это соответствует остальной части серии, но новый `http.ServeMux` позволяет легко ограничить пути с помощью подстановочного символа `{$}`, если это необходимо.

Как и в примере `gorilla/mux`, здесь мы используем специфические HTTP-методы для маршрутизации запросов (с одинаковым путем) к разным обработчикам; в старой версии `http.ServeMux` такие матчеры должны были обращаться к одному и тому же обработчику, который в зависимости от метода решал, что делать.

Рассмотрим также один из обработчиков:

```go
func (ts *taskServer) getTaskHandler(w http.ResponseWriter, req *http.Request) {
  log.Printf("handling get task at %s\n", req.URL.Path)

  id, err := strconv.Atoi(req.PathValue("id"))
  if err != nil {
    http.Error(w, "invalid id", http.StatusBadRequest)
    return
  }

  task, err := ts.store.GetTask(id)
  if err != nil {
    http.Error(w, err.Error(), http.StatusNotFound)
    return
  }

  renderJSON(w, task)
}
```

Он извлекает значение ID из `req.PathValue("id")`, аналогично подходу Gorilla, однако, поскольку у нас нет regexp, указывающего, что `{id}` соответствует только целым числам, нам приходится обращать внимание на ошибки, возвращаемые `strconv.Atoi`.

В целом, конечный результат очень похож на решение с использованием `gorilla/mux` из [второй части](https://eli.thegreenplace.net/2021/rest-servers-in-go-part-2-using-a-router-package/). Обработчики разделены гораздо лучше, чем в ванильном stdlib-подходе, поскольку теперь mux может выполнять более сложную маршрутизацию, не оставляя многие решения по маршрутизации на усмотрение самих обработчиков.

## Заключение

Вопрос "Какой пакет маршрутизаторов мне использовать?" всегда был FAQ для начинающих Go-программистов. Я полагаю, что после выхода Go 1.22 общие ответы на этот вопрос изменятся, поскольку многие сочтут новый stdlib mux достаточным для своих нужд, не прибегая к использованию пакетов сторонних разработчиков.

Другие будут придерживаться привычных пакетов сторонних разработчиков, и это совершенно нормально. Такие маршрутизаторы, как `gorilla/mux`, по-прежнему предоставляют больше возможностей, чем стандартная библиотека; кроме того, многие Go-программисты предпочитают использовать легкие фреймворки, такие как Gin, которые предоставляют не только маршрутизатор, но и дополнительные инструменты для создания веб-бэкендов.

В целом, это, безусловно, положительное изменение для всех пользователей Go. Расширение возможностей стандартной библиотеки - это положительный момент для всего сообщества, независимо от того, используют ли люди сторонние пакеты или ограничиваются стандартной библиотекой.

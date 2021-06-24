# Описание процедуры загрузки образа Fedora CoreOs для различных потоков

Загрузку и установку образа производит команда [coreos-installer](https://github.com/coreos/coreos-installer).

В зависимости от указанного потока (next, testing, stable) `coreos-installer` заправшивает мета-информацию по URL:
`https://builds.coreos.fedoraproject.org/streams/<имя_потока>.json`.

# Структура метаданных описания образов потока

Рассмотрим структуру метаданных описания образов потока `stable`.
Она расположена по URL: `https://builds.coreos.fedoraproject.org/streams/stable.json`.
```
{
    "stream": "stable",
    "metadata": {
        "last-modified": "2021-06-15T13:01:11Z"
    },
    "architectures": {
      ...
    }
}
```

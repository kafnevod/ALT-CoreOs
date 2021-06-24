# Описание процедуры загрузки образа Fedora CoreOs для различных потоков

Загрузку и установку образа производит команда [coreos-installer](https://github.com/coreos/coreos-installer).

В зависимости от указанного потока (next, testing, stable) `coreos-installer` заправшивает мета-информацию по URL:
`https://builds.coreos.fedoraproject.org/streams/<имя_потока>.json`.

## Структура метаданных описания образов потока

Рассмотрим структуру метаданных описания образов потока `stable`.
Она расположена по URL: `https://builds.coreos.fedoraproject.org/streams/stable.json`.
Верхний уровень JSON-описания выглядит следующим образом:
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
### Описание архитекутр (architectures)

```
    "architectures": {
        "x86_64": {
            "artifacts": {
                "aliyun": { ... },
                "aws": { ... },
                "azure": { ... },
                "digitalocean": { ... },
                "exoscale": { ... },
                "gcp": { ... },
                "ibmcloud": { ... },
                "metal": { ... },
                "openstack": { ... },
                "qemu": { ... },
                "vmware": { ... },
                "vultr": { ... }
            },
            "images": {
                "aws": {
                    "regions": {
                        "af-south-1": {
                            "release": "34.20210529.3.0",
                            "image": "ami-0d9658cc46b2bba75"
                        },
                        "ap-east-1": {
                            "release": "34.20210529.3.0",
                            "image": "ami-0af9744b58d7daa93"
                        },
                        ...
                    }
                },
                "gcp": {
                    "project": "fedora-coreos-cloud",
                    "family": "fedora-coreos-stable",
                    "name": "fedora-coreos-34-20210529-3-0-gcp-x86-64"
                }                }
            }
        }
}
```



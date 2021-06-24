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
В настоящий момент (`24.06.2021`) поддерживается только архитектура `x86_64`.
Данная архитектура содержит два подэлемента:
- `arfifacts` содержащий описания образов для различный `облачных платформ`, `голого железа` и `qemy`.
- `images` - содержащий дополнитаельныю информацию по облачным платформам `aws` и `gcp`.

### Описние платформ `artifacts`

Описание платформы формируется по шаблону:
```
        {
            "release": "34.20210529.3.0",
            "formats": {
            "<формат>": {
                "disk": {
                    "location": "https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/...",
                    "signature": "https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/...sig",
                    "sha256": "...",
                    "uncompressed-sha256": "..."
                }
            },
        }
```
В случае, если образ не сжат поле `uncompressed-sha256` отсутсвует.

Форматы образов для различных сред:
Среда развертывания | iso | ova | pxe | qcow2.xz | raw.xz | 4k.raw.xz | tar.gz | vhd.xz | vmdk.xz |
--------------------|-----|-----|-----|----------|--------|-----------|--------|--------|---------|
aliyun              |     |     |     |     V    |        |           |        |        |         |
aws                 |     |     |     |          |        |           |        |        |    V    |
azure               |     |     |     |          |        |           |        |    V   |         |
digitalocean        |     |     |     |      V   |        |           |        |        |         |
exoscale            |     |     |     |      V   |        |           |        |        |         |
gcp                 |     |     |     |          |        |           |   V    |        |         |
ibmcloud            |     |     |     |      V   |        |           |        |        |         |
metal               |   V |     |   V |          |    V   |     V     |        |        |         |
openstack           |     |     |     |      V   |        |           |        |        |         |
qemu                |     |     |     |      V   |        |           |        |        |         |
vmware              |     |   V |     |          |        |           |        |        |         |
vultr               |     |     |     |          |    V   |           |        |        |         |





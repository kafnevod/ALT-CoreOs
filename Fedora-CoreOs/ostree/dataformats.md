# OSTree data formats

    On the topic of “smart servers”
    The archive format
    archive efficiency
    Aside: the bare and bare-user formats
    Static deltas
    Static delta repository layout
    Static delta internal structure
        The delta superblock
    A delta part
    Fallback objects
        Licensing for this document:

## По теме «умных серверов»

Одно действительно важное различие между OSTree и git состоит в том, что у git есть «умный сервер». Даже при загрузке по https: // это не просто статический веб-сервер, а тот, который, например, динамически вычисляет и сжимает файлы пакетов для каждого клиента.

Напротив, автор OSTree считает, что для обновлений операционной системы многие развертывания захотят использовать простые статические веб-серверы, те же цели, для которых было разработано большинство систем пакетов. Основными преимуществами являются безопасность и эффективность вычислений. Такие сервисы, как Amazon S3 и CDN, являются канонической целью, 
также как и источником - стандартный статический сервер nginx. 

## Формат archive

В разделе [Анатомия репозитория OSTree](anatomy.md)  была представлена ​​концепция объектов, в которых объекты file/content суммируются по контрольной сумме и управляются индивидуально. (В отличие от системы пакетов, которая работает со сжатыми агрегатами).

Формат archive просто сжимает каждый контентный объект с помощью gzip. 
Объекты метаданных хранятся без сжатия. 
Это означает, что данный формат легко поддерживать через статический HTTP сервер. 
> Примечание: в конфигурационном файле репо по-прежнему используется исторический термин archive-z2 в качестве режима. Но это, по сути, указывает на современный формат archive.

Когда вы делаете новый коммит на новый контент, вы увидите новые файлы `.filez` в  подкаталогах каталога `objects/`.

## Эффективность формата archive

Преимущества формата archive:

- Легко понять и реализовать
- Может обслуживаться статическим веб-сервером напрямую по обычному протоколу HTTP.
- Клиенты могут скачивать/распаковывать обновления постепенно
- Эффективное использование пространства на сервере

Самым большим недостатком этого формата является то, что для выполнения обновления клиентом требуется один HTTP-запрос на каждый измененный файл. 
В некоторых сценариях это на самом деле совсем неплохо, особенно с методами уменьшения накладных расходов HTTP, такими как HTTP / 2.

Чтобы этот формат работал хорошо, вы должны разрабатывать свой контент таким образом, чтобы большие данные, 
которые нечасто изменяются (например, графические изображения), хранились отдельно от небольших, 
часто меняющихся данных (кода приложения).

Другие недостатки формата archive:

- Это очень плохо, когда клиенты выполняют начальное опрашивание (без HTTP / 2),
- Никто не знает общий размер (сжатый или несжатый) контента перед загрузкой всего объема


## bare и bare-user форматы

Самая распространенная в OSTree операция - `pull` из репозитория формата `archive` в репозитории формата `bare` и `user bare`. 
Последние два формата хранят данные на диске в исходном несжатом виде.
Другими словами, операция 'pull'  аналогична операции разжатия ((но не установке)) файлов rpm/deb-пакетов

Формат `bare-user`  немного особенный, поскольку в нем игнорируются идентификаторы uid / gid и xattrs из содержимого. 
Это в первую очередь полезно, если вы хотите иметь тот же контент, управляемый OSTree, который можно запускать в хост-системе или непривилегированном контейнере. 

## Статические дельты

OSTree изначально был ориентирован на `continuous delivery` модель, при которой клиентские системы должны регулярно обновляться. 
Однако многие поставщики ОС хотели бы предоставлять обновленный контент, например раз в месяц или реже.

Для этой модели мы можем намного лучше поддерживать пакетные обновления, чем базовое архивное репо. Однако мы по-прежнему хотим сохранить модель «только статический веб-сервер». 
Учитывая это, в рамках OSTree разработана концепция «статической дельты».

Эти дельты нацелены на то, чтобы быть разницей между двумя конкретными коммитами, включая дельты «bsdiff» и «rsync-style» в объекте содержимого. 
Статические дельты также поддерживаются от NULL, когда клиент может более эффективно загружать объект фиксации с нуля - это в основном полезно при использовании OSTree для контейнеров, а не образов ОС. 
Для образов ОС обычно загружают установочный ISO-образ или образ qcow2, который представляет собой отдельный файл, который уже содержит данные дерева.

По сути, мы тратим ресурсы  на стороне сервера (и единовременные затраты на вычисления) и повышаем эффективность использования полосы пропускания клиентской сети. 

## Статический макет дельта-репозитория

Поскольку статические дельты могут не существовать, клиенту сначала нужно попытаться найти их. 
Предположим, клиент хочет получить фиксацию ${new} во время выполнения ${current}.

В целях экономии места эти два коммита представляют собой «измененный base64» - символ `/` заменяется на `_`.

Как и коммиты, «каталог префиксов» используется для упрощения управления инструментами файловой системы.

Дельта подучает имя $ (mbase64 $from)-$(mbase64 $to), например, GpTyZaVut2jXFPWnO4LJiKEdRTvOw_mFUCtIKW1NIX0-L8f+VVDkEBKNc1Ncd+mDUrSVR4EyybQGCkuKtkDnTwk, которая в формате SHA256 представляется как 1a94f265a56eb768d714f5a73b82c988a11d453bcec3f985502b48296d4d217d-2fc7fe5550e410128d73535c77e98352b495478132c9b4060a4b8ab640e74f09.

Наконец, фактическое содержимое можно найти в deltas `deltas/$fromprefix/$fromsuffix-$to`. 

## Static delta internal structure

A delta is itself a directory. Inside, there is a file called superblock which contains metadata. The rest of the files will be integers bearing packs of content.

The file format of static deltas should be currently considered an OSTree implementation detail. Obviously, nothing stops one from writing code which is compatible with OSTree today. However, we would like the flexibility to expand and change things, and having multiple codebases makes that more problematic. Please contact the authors with any requests.

That said, one critical thing to understand about the design is that delta payloads are a bit more like “restricted programs” than they are raw data. There’s a “compilation” phase which generates output that the client executes.

This “updates as code” model allows for multiple content generation strategies. The design of this was inspired by that of Chromium: ChromiumOS Autoupdate.
The delta superblock

The superblock contains:

    arbitrary metadata
    delta generation timestamp
    the new commit object
    An array of recursive deltas to apply
    An array of per-part metadata, including total object sizes (compressed and uncompressed),
    An array of fallback objects

Let’s define a delta part, then return to discuss details:

### A delta part

A delta part is a combination of a raw blob of data, plus a very restricted bytecode that operates on it. Say for example two files happen to share a common section. It’s possible for the delta compilation to include that section once in the delta data blob, then generate instructions to write out that blob twice when generating both objects.

Realistically though, it’s very common for most of a delta to just be “stream of new objects” - if one considers it, it doesn’t make sense to have too much duplication inside operating system content at this level.

So then, what’s more interesting is that OSTree static deltas support a per-file delta algorithm called bsdiff that most notably works well on executable code.

The current delta compiler scans for files with matching basenames in each commit that have a similar size, and attempts a bsdiff between them. (It would make sense later to have a build system provide a hint for this - for example, files within a same package).

A generated bsdiff is included in the payload blob, and applying it is an instruction.

## Fallback objects

It’s possible for there to be large-ish files which might be resistant to bsdiff. A good example is that it’s common for operating systems to use an “initramfs”, which is itself a compressed filesystem. This “internal compression” defeats bsdiff analysis.

For these types of objects, the delta superblock contains an array of “fallback objects”. These objects aren’t included in the delta parts - the client simply fetches them from the underlying .filez object.

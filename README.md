# FRP mTLS LAB 🛡️

Полностью повторяемая лабораторная работа, показывающая  
как поднять **frps ↔ frpc** соединение только через **взаимную TLS-аутентификацию**  
(каждый участник предъявляет сертификат, подписанный вашим CA).

* **Без открытых портов для клиента** — оба контейнера живут в общей `bridge`-сети `frpnet`
* **Минимум образов** — `snowdreamtech/frps`, `snowdreamtech/frpc` и готовый echo-backend
* **Однокнопочная PKI** — скрипт `frpgen` 🔐 создаёт CA и подписывает узлы

---

## 1. Генерим сертификаты

### Корневой CA  (папка certs будет создана)
```bash
./frpgen root certs --overwrite
```

### Сертификат сервера (CN = server, SAN=DNS:server)
```bash
./frpgen client certs/server --ca certs
```

### Сертификат клиента  (CN = client, SAN=DNS:client)
```bash
./frpgen client certs/client --ca certs
```

> Важно
>
> * `CN` = имя, которым затем пользуемся в TLS-проверке
> * `frpgen` автоматически добавляет **SAN**.
>   Для сертификата `server` это `DNS:server`, поэтому в конфиге клиента
>   `transport.tls.serverName = "server"`.

Файлы, которые получились:

```
certs/
  FRP_CA.crt            # публичный корень – раздаётся всем
  FRP_CA.key            # приватный ключ CA (храните локально!!)
  server/
      server.crt
      server.key
  client/
      client.crt
      client.key
```

---

## 2. Запускаем frps

```bash
cd frps
docker compose up -d
```

*Откроет порты only:* **7000/tcp** (контрольный, внутри сети)
и **9003/tcp** (наш опубликованный сервис-эхо).

Состав `frps.toml` (фрагменты):

```toml
bindPort = 7000                     # контрольный канал

transport.tls.force = true          # принимать ТОЛЬКО TLS
transport.tls.certFile      = "/etc/frp/certs/frps.crt"
transport.tls.keyFile       = "/etc/frp/certs/frps.key"
transport.tls.trustedCaFile = "/etc/frp/certs/CA.crt"

allowPorts = [ { single = 9003 } ]  # клиенту разрешён ровно один remotePort
```

---

## 3. Запускаем frpc + backend

```bash
cd ../frpc
docker compose up -d
```

### Что тут происходит

| сервис      | роль                                                                              | порт |
| ----------- | --------------------------------------------------------------------------------- | ---- |
| **backend** | готовый образ `mendhak/http-https-echo:29`<br>слушает `8080/tcp` и отдаёт JSON    | 8080 |
| **frpc**    | устанавливает mTLS-туннель к `frps:7000`,<br>форвардит `local 8080 → remote 9003` | –    |

`frpc.toml` ключевые строки:

```toml
serverAddr = "frps"           # DNS-alias контейнера сервера
...
transport.tls.serverName = "server"   # ДОЛЖЕН совпасть с CN в server.crt
```

---

## 4. Проверяем

| должно работать  | команда                                                    | результат                              |
| ---------------- | ---------------------------------------------------------- | -------------------------------------- |
| Сервис через FRP | `curl http://127.0.0.1:9003`                               | JSON-эхо                               |
| Dashboard        | браузер → `http://127.0.0.1:7500`<br>логин **admin/admin** | видим 1 активный прокси, mTLS = `true` |

Если убрать у клиента `client.crt`, frps мгновенно разорвёт handshake и лог покажет
`bad certificate`.

---

## Разбор конфигов

### `frps/compose.yml`

| секция      | зачем                                          |
| ----------- | ---------------------------------------------- |
| `volumes:`  | монтируем certs внутрь контейнера              |
| `ports:`    | 7500 (dashboard)<br>9003 (echo-сервис)         |
| `networks:` | оба проекта в `frpnet` ⇒ не нужен publish 7000 (в реальной ситуации порт `7000` должен быть доступен клиенту и быть открыт) |

### `frpc/compose.yml`

| сервис      | особенность                                                     |
| ----------- | --------------------------------------------------------------- |
| **frpc**    | `volumes:` — свой .toml + client certs<br>`depends_on: backend` |
| **backend** | готовый echo-образ                        |

### Поле `transport.tls.serverName`

* Клиент во время TLS-handshake посылает **SNI = "server"**
* OpenSSL проверяет, что это совпало с **SAN DNS\:server** в сертификате frps
* Если не совпало — `tls: bad certificate`.

---

## Что почитать дальше

* FRP docs — [https://github.com/fatedier/frp](https://github.com/fatedier/frp)
* SAN vs CN в OpenSSL — [https://stackoverflow.com/questions/3112787/](https://stackoverflow.com/questions/3112787/)
* Паттерн **mTLS + token** (в конфиге уже можно добавить `auth.method = "token"`).

Happy tunnelling!🚀

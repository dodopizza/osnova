# Some docs for [OSNOVA](https://rbk.money/osnova/)

## Navigation

- [Overview](#overview)
- [Settings](#settings)
- [Domain objects](#domain-objects)
- [Adapter development](#adapter-development)

## Overview

- [Документация общая](https://rbk.money/osnova/)
- [Документация по апи](https://developer.rbk.money/api/)
- [Документация по объектам системы](https://github.com/rbkmoney/damsel/)

## Settings

Иерархия настроек и их типы.

Есть неочевидная вещь, что некоторые настройки обязательны к заполнению, хоть в интерфейсе и не указано, но без них просто работать ничего не будет, размечаю такие настройки звёздочками.

<pre>
<b>Name                              |  Type</b>
------------------------------------------------------------------------------
PaymentInstitution                | PaymentInstitutionObject
|- SystemAccountSet               | SystemAccountSetObject
|- DefaultContractTemplate        | ContractTemplateObject
|- WalletSystemAccountSet         | SystemAccountSetObject
|- DefaultWalletContractTemplate  | ContractTemplateObject
|- PaymentRoutingRules            | 
|-|- Policies                     | RoutingRulesObject
|-|-|- Candidates                 | RoutingCadidate
|-|-|-|- Terminal                 | TerminalObject
|-|-|-|-|- Provider               | ProviderObject
|-|-|-|-|-|- Proxy                | ProxyObject
|-|- Prohibitions                 | RoutingRulesObject, same as policies
|- Inspector                      | InspectorObject
</pre>

## Domain objects

Этот раздел надо предложить смёржить в [damsel](https://github.com/rbkmoney/damsel/).

### Party

Абстракция для юридического лица или мерчанта в реальном мире. 

Чтобы создать новый party нужно создать новый аккаунт в [keycloak](https://auth.rbk-pay.dodois.ru/) в realm external, добавить в группу merchants, party для этого аккаунта создастся автоматически.

В нашем бизнесе это один партнёр-франчайзи, у которого может быть 1+ пиццерий.

### Shop

Абстракция для магазина, в рамках магазина ведётся учет транзакий. 

Тестовый shop создаётся автоматически после создания [party](#party). 

В нашем бизнесе это будет одна, отдельно взятая пиццерия, донерная или дринкит.

### Proxy

Абстрация для сервисов интеграции с эквайрингами — банками, платёжными шлюзами и прочими провайдерами платёжных услуг.

### PaymentInstitution

Некий объект отражающий юридическое лицо самой системы, кто собственно осуществляет операции (юридически это не совсем так на самом деле, и там много сущностей, но в доменной модели сделано так). Соответственно на PaymentInstitution ссылаются все договора и условия. Отсюда начинаются все проверки и проще всего делать настройки начиная с этой сущности.

### ContractTemplate

Что-то вроде набора условий прописанных в договоре с мерчантом. 

## Adapter development

Пишем на джаве, т.к. весь необходимый тулинг уже готов.

Провайдеры бывают разные, например, для операций оплаты, выплат и другие, для начала рассмотрим как писать адаптеры для операций оплаты.

В любом случае, написание начинаем с того, что берем за основу [мок провайдер](https://github.com/rbkmoney/proxy-mocketbank), дальше в зависимости от типа.

### Payment

Обработка внешних запросов происходит по такой схеме:

(внешний мир) -> adapter -> hellgate -> machinegun -> hellgate -> adapter -> (внешний мир) 

Machinegun решает проблему обработки параллельных запросов и сохраняет данные.

Внешний обработчик — просто контроллер, см. /main/java/com/rbkmoney/proxy/mocketbank/controller/MocketBankController.java

Собирает инфу из запроса и кидает её в hellgate, затем формирует ответ для вызывающей стороны. Позже мы получим эти данные от hellgate во внутренний обработчик упорядоченно.

Это может быть редирект, джисон или пустой ответ в зависимости от запроса, например, если это post 3DS редирект, то скорее всего он в потоке клиента, и нужно сформировать редирект для клиента.

Обработка внутренних запросов происходит по той же схеме.

Внутренний обработчик — сервлет, см. /main/java/com/rbkmoney/proxy/mocketbank/controller/AdapterServletPayment.java

Обработка внутренних запросов начинается с /main/java/com/rbkmoney/proxy/mocketbank/handler/payment/PaymentServerHandler.java. Сначала выбирается нужный обработчик в зависимости от значения `context.getSession().getTarget()`, context, кстати, содержит всю необходиму информацию о платеже. Значением `context.getSession().getTarget()` может быть `processed`, `captured`, `cancelled`, `refunded`. Думаю, объяснять это не нужно.

Обработчик в свою очередь возвращает `PaymentProxyResult`, в котором нам сейчас интересен `Intent`. `Intent` отражает непосредственно результат обработки — `finish`, `suspend`, `sleep`. В случае `finish` считается, что запрошенная операция выполнена успешно, например, если запрашивали `processed`, то платёж авторизован, деньги захолдированы. В случае `suspend` мы ждём колбек или ввод данных от пользователя. В случае `sleep` адаптер будет вызван повторно через некоторое время.

В целом, зная это, уже всё должно быть понятно :) 






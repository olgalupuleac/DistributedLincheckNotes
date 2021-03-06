# Архитектура для стресс-режима
## Высокоуровневая архитектура MVP 
1. Пользователь пишет класс и наследует его от `Node` (то есть переопределяет метод `onMessage(message : Message)`). Также его класс в конструктор принимает instance интерфейса `Environment`, через который присходит отправка сообщений.
1. `Environment` позволяет отправить сообщение `Message`. Также по всей видимости он должен уметь возвращать глобальные параметры системы (количество сообщений от каждого процесса, отказавшие процессы и так далее). 
1. `Message` хранит в себе текст сообщения, `sender`, `receiver`, `headers` -- `HashMap<String, String>`, набор произвольных заголовков.
1. Пользователь задаёт конфигурацию системы в `DistributedOptions`. Параметрами являются:
    * Число узлов (называется `threads`, хотя тут было бы более подходящим другое название)
    * Вероятность, с которой доставляются сообщения: `networkReliability`
    * Сколько раз максимум может продублироваться одно сообщение: `duplicationRate`
    * Какой порядок доставки гарантирует система (синхронный, FIFO, асинхронный)
    * Могут ли в системе отказывать узлы (`allowNodeFails`)
    * Могут ли узлы восстанавливаться после падения (`supportNodeRecovery`)
    * Максимальное число узлов, которое может отказать (`maxNumberOfFailedNodes`)
    * Максимальное число сообщений, которые могут быть отправлены одним процессом
1. Помимо конфигурации системы, задаётся конфигурация теста. То есть:
    * Число итераций
    * Число вызовов функций в течении одной итерации
1. Далее по `DistributedOptions` создаётся `DistributedCTestConfiguration`, а потом из неё `DistributedStrategy`.
1. Внутри `DistributedStrategy` запускается `DistributedRunner` число итераций раз.
1. Внутри `Runner` каждый раз происходит запуск определённого сценария. 
## Как изнутри будет устроен `Runner` для MVP
### Запуск `Node`'ов
1. Есть набор `testInstances`, в котором хранятся `instance`'ы `Node`'ов. 
1. Для каждого `Node`'а создаётся свой `TestThreadExecution` 
1. По-видимому, по умолчанию исполнение запускается в отдельном потоке
1. При этом `onMessage` будут другие потоки вызывать. 
1. Думаю, что нужно сделать возможность наследоваться от `NodeWithReceiveImpl`, который хранит в себе очередь сообщений и реализует метод `receive` с таймаутом и без него. 
1. Нужно для каждого процесса сделать свой `Environment` в зависимости от настроек, переданных пользователем. 
1. В процессе `Node`'ы могут отказывать, и даже могут потом обратно срабатывать. 
### Создание `Environment`
Нужно, чтобы были соблюдались все свойства каналов связи. 
1. Потеря сообщений -- легко: генерируем рандомный `double` от 0 до 1, смотрим, больше или меньше он надёжности сети, если больше, то сообщение теряется. 
1. Дублирование сообщений -- легко: если сообщение должно быть доставлено, то генерируем число сообщений от 1 до `duplicationRate` и доставляем столько раз. 
1. Порядок доставки сообщений: уже не так легко. 
### Как должна изнутри выглядеть обработка сообщений. 
1. Поток `MessageBroker` крутится в цикле и проверяет, не появились ли новые сообщения. 
2. Для этого он достаёт из наследника интерфейса `MessageQueue` следующее сообщение.
3. `MessageQueue` отвечает за перемешивание сообщений. 

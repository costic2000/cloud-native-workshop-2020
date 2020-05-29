# Наблюдаемость с Istio

Sidecar прокси-контейнеры перехватывают весь траффик, что позволяет им 
проверять информацию и атрибуты HTTP траффика. Благодаря этому, Istio 
улучшает наблюдаемость микросервисов, не требуя изменения приложений.
Таким образом "из коробки" мы получаем мониторинг, логирование, распределенную 
трассировку и возможность строить service dependency graph.
 
Взглянем на pod-ы и service-ы, созданные в `istio-system` namespace:
```
kubectl get pods -n istio-system
kubectl get services -n istio-system
```
Теперь нам нужно создать новые заказы кофе в coffe-shop приложении, то есть
снова подключиться к приложению через gateway:
```
while true; do
curl <ip-address>:<node-port>/coffee-shop/resources/orders -i -XPOST \
    -H 'Content-Type: application/json' \
    -d '{"type":"Espresso"}' \
    | grep HTTP
sleep 1
done
```

Эти запросы будут создавать заказ кофе каждую секунду и следовательно,
будет генерировать постоянную нагрузку на наши микросервисы.

## Мониторинг (Grafana)


Наша установка Istio "из коробки" предоставляет мониторинг и 
Grafana—дашборды . Чтобы получить доступ к ним, мы должны установить 
соединение с pod-ом Grafana.

Мы могли бы создать выделенный service и gateway, который направляет
траффик к pod-у, но для тестовых целей мы выполним port forwarding 
с локального порта `3000` на Grafana pod:
```
kubectl -n istio-system port-forward \
    $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') \
    3000:3000 &
```
Как только эта переадресация будет выполнена, мы перейдем к
<http://localhost:3000> затем к Istio Mesh Dashboard,
щелкнув на Home в левом верхнем углу.

Вы можете изучить все технические метрики, которые доступны в Istio по умолчанию. 

## Service Graph (Kiali)

Наша установка Istio также поставляется с Service Graph, который показывает
зависимости и взаимодействия между отдельными сервисами.

Выполним port forwarding из локального порта `20001` к service graph pod-у:
```
kubectl -n istio-system port-forward \
    $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') \
    20001:20001 &
```
Откроем <http://localhost:20001/> и исследуем экземпляры service graph
в разделе "Graph". Изучите доступные опции в разделах
"Display" и "Graph Type".

## Трассировка (Jaeger)

Наша установка Istio также поставляется с распределенной трассировкой,
которая позволяет отследить отдельные запросы, которые произошли между
нашими микросервисами.

Распределенная трассировка — единственная функция наблюдения, которая
не работает из коробки без воздействия с нашей стороны. По умолчанию
Istio  не может узнать, что два отдельных HTTP-запроса: один между 
ingress gateway и coffe-shop service, и второй между приложениями 
coffe-shop и barista, на самом деле взаимосвязаны.

Внутри должно произойти следующее: приложение coffe-shop должно извлечь
и передать определенный заголовок трассировки, то есть HTTP headers
(`x-b3-traceid`, `x-b3-parentspanid`, `x-b3-spanid`, …). Sidecar-ы
затем могут наблюдать и сопоставлять эту дополнительную информацию.

Первоначально это означало бы, что наше приложение coffe-shop должно
было бы получить заголовки трассировки из входящего HTTP-запроса,
сохранить информацию в local request (thread) scope и добавить
ее в исходящий HTTP-запрос на клиенте, который подключается к сервису
barista снова. К счастью, с помощью MicroProfile OpenTracing, нам 
не нужно делать это вручную.

Наше запущенное приложение может быть настроено на использование 
MicroProfile OpenTracing, который передает эту информацию через 
входящие и исходящие HTTP-запросы, если заголовки трассировки были
доступны при первом запросе.

Для этого нам нужно только дать указание нашим серверам Open Liberty
активировать соответствующую функцию. Нам не нужно менять ни Java код, 
ни сборку приложения. Это настраивается исключительно на уровне инфраструктуры.

Взглянем на конфигурацию нашего coffe-shop приложения `server.xml`:
помимо `jakartaee-8.0`, она уже содержит `microProfile-3.0` и
`usr:opentracingZipkin-0.31`. Это позволит автоматически передавать
HTTP-заголовки трассировки.

```xml
    ...

    <featureManager>
        <feature>jakartaee-8.0</feature>
        <feature>microProfile-3.0</feature>
        <feature>usr:opentracingZipkin-0.31</feature>
    </featureManager>

    ...
```

Теперь выполним port forwarding с локального порта 16686 на Jaeger pod:
```
kubectl port-forward -n istio-system \
    $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') \
    16686:16686 &
```
Перейдем к <http://localhost:16686>, выберем `istio-ingressgateway` как сервис, 
и кликнем на кнопку *`Find Traces`*, чтобы посмотреть последние трейсы.
 
Если мы проверим правильные трейсы, то сможем увидеть, что наше coffe-shop
приложение синхронно и ассинхронно подключается к barista backend.

В [следующем разделе](06-istio-routing.md) мы рассмотрим как мы настраиваем Istio
для маршрутизации нашего mesh-траффика в соответствии с определенными критериями.
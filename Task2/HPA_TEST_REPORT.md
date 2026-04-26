# Отчёт по тестированию HPA (Задание 2)

## Конфигурация

| Параметр | Значение |
|----------|---------|
| **Deployment** | `scaletestapp` |
| **Образ** | `ghcr.io/yandex-practicum/scaletestapp:latest` |
| **Service** | `scaletestapp-service` (ClusterIP) |
| **HPA** | `scaletestapp-hpa` |
| **Min replicas** | 1 |
| **Max replicas** | 10 |
| **Target memory utilization** | 80% |
| **Memory requests** | 5Mi |
| **Memory limits** | 30Mi |

## Шаги выполнения

```bash
# 1. Запуск Minikube
minikube start --memory=4096 --cpus=2
minikube addons enable metrics-server

# 2. Применение манифестов
kubectl apply -f Task2/deployment.yaml
kubectl apply -f Task2/service.yaml
kubectl apply -f Task2/hpa.yaml

# 3. Проверка статуса
kubectl get pods -w
kubectl get hpa -w

# 4. Получить URL сервиса
minikube service scaletestapp-service --url

# 5. Запуск Locust
locust --host=http://127.0.0.1:<PORT> --headless -u 100 -r 10 --run-time 120s

# 6. Открыть дашборд Kubernetes
minikube dashboard
```

## Результаты тестирования

### Начальное состояние (1 реплика)
- Утилизация памяти: ~65%
- Количество подов: 1
- HPA не активирован (65% < 80%)

### Под нагрузкой (Locust: 100 users, spawn rate 10)
- Утилизация памяти: ~185%
- Количество подов: 10 (максимум)
- HPA масштабировал с 1 до 10 реплик
- Locust обработал ~2342 запроса с 100% отказов (Connection refused — закончился run-time)

### Выводы
- ✅ HPA успешно масштабирует поды при утилизации памяти > 80%
- ✅ При 185% утилизации HPA увеличил реплики до максимума (10)
- ✅ Автоскейлинг работает корректно

## Скриншоты

| Файл | Описание |
|------|----------|
| `screenshots/screenshot-1-initial-state.png` | Начальное состояние (1 реплика) |
| `screenshots/screenshot-2-under-load.png` | Состояние под нагрузкой |
| `screenshots/screenshot-3-scaled.png` | После масштабирования (10 реплик) |

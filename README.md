# Тестовый проект для интервью

Добрый день! \
Этот проект является результатом выполнения следующего тестового задания:

>Создать репозиторий, в котором будет написано простое web приложение hello world на любом языке. \
>В репозитории должно быть:
>1. Readme.md с описанием, как собирать и запускать
>1. Dockerfile
>1. Ресурсы для kubernetes по всем канонам (а может и helm chart, на усмотрение кандидата)

>Оцениваем самостоятельность решения, с пониманием каждой строчки.

### Состав репозитория

Репозиторий состоит из двух директорий:
* непосредственно само приложение на Go и Dockerfile к нему,
* а также helm chart для создания сущностей k8s.

### Порядок действий

> В рамках данного примера предлагается использовать Google Cloud Platform, в случае замены его на какой-либо другой облачный провайдер, — необходимо будет скорректировать выполнение команд апплета gcloud.

1. Определяем переменные с названием проекта и регионом развёртывания:
```
export PROJECT_ID=arrival-test-project && echo ${PROJECT_ID}
export REGION=europe-north1 && echo ${REGION}
```

2. Собираем Docker образ, создаём репозиторий в GCP, и пушим созданный образ в него:
```
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${PROJECT_ID}-repo/${PROJECT_ID}-app:v1 ./application/

gcloud artifacts repositories create ${PROJECT_ID}-repo --repository-format=docker --location=${REGION}

docker push ${REGION}-docker.pkg.dev/$PROJECT_ID/$PROJECT_ID-repo/${PROJECT_ID}-app:v1
```

3. Создаём кластер Kubernetes, формируем helm-пакет, пушим в репозиторий GCP, а затем создаём с его помощью объекты Kubernetes, описанные в helm chart'e _./helm/arrival-test-project/templates_:
```
gcloud container clusters create --zone ${REGION}-a arrival-test-project-cluster

gcloud auth print-access-token | helm registry login -u oauth2accesstoken --password-stdin https://${REGION}-docker.pkg.dev

helm package helm/arrival-test-project/ && \
  helm push arrival-test-project-0.1.0.tgz oci://${REGION}-docker.pkg.dev/${PROJECT_ID}/${PROJECT_ID}-repo

kubectl create namespace $PROJECT_ID && kubectl config set-context --current --namespace=$PROJECT_ID

helm install $PROJECT_ID oci://${REGION}-docker.pkg.dev/${PROJECT_ID}/${PROJECT_ID}-repo/${PROJECT_ID} --version 0.1.0 --namespace arrival-test-project
```

4. Создаём внешний балансировщик, и получаем IP-адрес для проверки работоспособности собранного и запущенного приложения:
```
kubectl expose deployment ${PROJECT_ID} --name=${PROJECT_ID}-lb --type=LoadBalancer --port 80 --target-port 8080

EXTERNAL_IP="$(kubectl get services | grep arrival-test-project-lb | awk '{print $4}')" && echo ${EXTERNAL_IP} && curl "${EXTERNAL_IP}"
```

### Результат работы приложения

Вывод команды curl:

```
35.228.119.8
Hello, world!
Version: 1.0.0
Hostname: arrival-test-project-6f8bbb945b-cgt58
```

# NewUnicorn

## Dockerize a Ruby on Rails application and deploy it on kubernates .  
Refrence  : https://semaphoreci.com/community/tutorials/dockerizing-a-ruby-on-rails-application

### **Prerequisites**:
- Installing Docker on your machine .
- Kubernate cluster .
  - you can use minikube or kubeadm to build a cluster on your local machines , kops to  build a cluster on a cloud provider like aws or gcloud
> In my solution i used Kubernetes Engine ( which provided my Google Cloud)

### **Generating a New Rails Application**
```bash
docker run -it --rm --user "$(id -u):$(id -g)" \
 -v "$PWD":/usr/src/app -w /usr/src/app rails:4 rails new --skip-bundle drkiq
```

The command above will create the application on our work station. Without  even needing Ruby installed on our work station.

### **make a few adjustments to our application to make it production ready.**
> use The Refrence which mentioned before to edit : Gemfile , config/database.yml , config/secrets.yml , config/application.rb and create config/unicorn.rb  and config/initializers/sidekiq.rb . 

### **Creating the Dockerfile**
```docker
FROM ruby:2.2.3-slim

MAINTAINER Ayman Behery <a.mhbehery@gmail.com>

# Install dependencies:
# - build-essential: To ensure certain gems can be compiled
# - nodejs: Compile assets
# - libpq-dev: Communicate with postgres through the postgres gem
# - postgresql-client-9.4: In case you want to talk directly to postgres
RUN apt-get update && apt-get install -qq -y build-essential nodejs libpq-dev postgresql-client-9.4 --fix-missing --no-install-recommends

# Set an environment variable to store where the app is installed to inside
# of the Docker image.
ENV INSTALL_PATH /drkiq
RUN mkdir -p $INSTALL_PATH

WORKDIR $INSTALL_PATH

COPY Gemfile Gemfile
RUN bundle install

COPY . .

VOLUME ["$INSTALL_PATH/public"]

CMD bundle exec rake RAILS_ENV=production DATABASE_URL=postgresql://user:pass@127.0.0.1/dbname SECRET_TOKEN=pickasecuretoken assets:precompile & bundle exec unicorn -c config/unicorn.rb & bundle exec sidekiq -C config/sidekiq.yml
```

### Build the docker image and tag it with your docker repository and push it 
```bash
docker build -t <repository>/<image name>:<release>  && docker push <repository>/<image name>:<release>
- docker build -t aymanbehery/newuincorn:latest && docker push aymanbehery/newuincorn:latest
```

> Note : you have to be in your project directory before build the image 

### **Creating kubernate Resources .**
create the  kubernetes/deployment.yaml :

- **Add service resources**
> note : if you created your  kubernate cluster on your local machine expose the drkiq service as NodePort , in my solution I exposed it as LoadBalance.

```yaml
kind: Service
apiVersion: v1
metadata:
  name:  drkiq
spec:
  selector:
    app: newunicorn
    type: drkiq
  type:  LoadBalancer 
  ports:
  - name:  app-port
    port:  80
    targetPort:  8000
---
kind: Service
apiVersion: v1
metadata:
  name:  postgres
spec:
  selector:
    app: newunicorn
    type: postgres
  type:   ClusterIP 
  ports:
  - name:  db-port
    port:  5432
    targetPort:  5432
---
kind: Service
apiVersion: v1
metadata:
  name:  redis
spec:
  selector:
    app: newunicorn
    type: redis
  type:   ClusterIP 
  ports:
  - name:  db-port
    port:  6379
    targetPort:  6379
```


- Add PersistentVolumeClaim resources :
```bash
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: drkiq-redis
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "10Gi"      
--- 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: drkiq-postgres
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "10Gi"
```

- Add ConfigMap resources:
> Using configMap to to pass the env variables to the container . 
```bash
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: drkiqenv
  labels:
    app: newunicorn
data:
  WORKER_PROCESSES: "1"
  LISTEN_ON: 0.0.0.0:8000
  DATABASE_URL: postgresql://drkiq:yourpassword@postgres:5432/drkiq?encoding=utf8&pool=5&timeout=5000
  CACHE_URL: redis://redis:6379/0
  JOB_WORKER_URL: redis://redis:6379/0
```
- Add secret resources :
> Using secret to pass the critical env variables to the container .
> Each item must be base64 encoded:

> `echo -n '<the  value>' | base64`
```bash
---
apiVersion: v1
kind: Secret
metadata:
  name:  token
data:
   SECRET_TOKEN: YXNlY3VyZXRva2Vud291bGRub3JtYWxseWdvaGVyZQo=
type: Opaque
```

- Add postgres and redis deployemnt :
> Mount volumes to postgres and redis deployemn to make them A stateful apps  that store information about what has happened or changed since it started running without losing any data. 
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: newunicorn
        type: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:9.4.5
        volumeMounts:
        - mountPath: /var/lib/redis/data
          name: drkiq-postgres-data
        env:
          - name: POSTGRES_USER
            value: drkiq
          - name: POSTGRES_PASSWORD
            value: yourpassword
      volumes:
        - name: drkiq-postgres-data
          persistentVolumeClaim:
            claimName: drkiq-postgres
---                         
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: newunicorn
        type: redis
    spec:
      containers:apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: newunicorn
        type: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:9.4.5
        volumeMounts:
        - mountPath: /var/lib/redis/data
          name: drkiq-postgres-data
        env:
          - name: POSTGRES_USER
            value: drkiq
          - name: POSTGRES_PASSWORD
            value: yourpassword
      volumes:
        - name: drkiq-postgres-data
          persistentVolumeClaim:
            claimName: drkiq-postgres
---                         
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: newunicorn
        type: redis
    spec:
      containers:
      - name: redis
        image: redis:3.0.5
        volumeMounts:
        - mountPath: /var/lib/redis/data
          name: drkiq-redis-data
      volumes:
        - name: drkiq-redis-data
          persistentVolumeClaim:
            claimName: drkiq-redis
      - name: redis
        image: redis:3.0.5
        volumeMounts:
        - mountPath: /var/lib/redis/data
          name: drkiq-redis-data
      volumes:
        - name: drkiq-redis-data
          persistentVolumeClaim:
            claimName: drkiq-redis
```

- Add drkiq ( The main a Ruby on Rails Application )  Deployemnt .
> In your Deployemnt use your image name , but you can use my image to test the deployment : aymanbehery/newunicorn:latest
```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: drkiq
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: newunicorn
        type: drkiq
    spec:
      containers:
      - name: drkiq
        image: aymanbehery/newunicorn:latest
        imagePullPolicy: Always
        env:
          - name:  SECRET_TOKEN
            valueFrom:
              secretKeyRef:
                name:  token
                key:  SECRET_TOKEN
          - name: WORKER_PROCESSES
            valueFrom:
              configMapKeyRef:
                name: drkiqenv
                key: WORKER_PROCESSES
          - name: JOB_WORKER_URL
            valueFrom:
              configMapKeyRef:
                name: drkiqenv
                key: JOB_WORKER_URL
          - name: CACHE_URL
            valueFrom:
              configMapKeyRef:
                name: drkiqenv
                key: CACHE_URL
          - name: LISTEN_ON
            valueFrom:
              configMapKeyRef:
                name: drkiqenv
                key: LISTEN_ON
          - name: DATABASE_URL
            valueFrom:
              configMapKeyRef:
                name: drkiqenv
                key: DATABASE_URL
```
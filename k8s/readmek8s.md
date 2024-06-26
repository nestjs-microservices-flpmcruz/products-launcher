# Helm commands
* Crear configuración `helm create <nombre>`
* Aplicar configuración inicial: `helm install <nombre> .`
* Aplicar actualizaciones: `helm upgrade <nombre> .`

# K8s commands
* Obtener pods, deployments y services: `kubectl get <pods | deployments | services>`
* Revisar todos pods: `kubectl describe pods`
* Revisar un pod: `kubectl describe pod <nombre>`
* Eliminar pod: `kubectl delete pod <nombre>`
* Revisar logs: `kubectl logs <nombre>`

```sh
# Crear deployment:
kubectl create deployment <nombre> --image=<registro/url/imagen> --dry-run=client -o yaml > deployment.yml

# Crear service
kubectl create service clusterip <nombre> --tcp=<8888> --dry-run=client -o yaml > service.yml 
**kubectl create service nodeport <nombre> --tcp=<3000> --dry-run=client -o yaml > service.yml**
```

# Secrets
```sh
# Crear secretos, varios a la vez, o uno por uno.
kubectl create secret generic <nombre> --from-literal=key=value
kubectl create secret generic secret1 --from-literal=key1=value1 --from-literal=key2=value2

# Obtener los secretos 
kubectl get secrets
# Ver el contenido de un secreto
kubectl get secrets <nombre> -o yaml

# Editar el secret
kubectl edit secret <nombre>
```

## Editar un secret
La forma más fácil es borrarlo y volverlo a crear pero si es más de un secret, no vamos a querer perder los demás.
Recordar que los secrets están en `base64`, por lo que si queremos editar un secret, debemos hacerlo en `base64`.

1. Editar el secret con `kubectl edit secret <nombre>` esto invocará el editor
2. Cambiar el valor (se puede usar un editor en [línea para convertir a base64](https://www.rapidtables.com/web/tools/base64-decode.html))
3. Tocar **i** para insertar líneas y editar el archivo
4. Poner el valor a decodificar en una nueva línea
5. Presionar **esc** y luego `:. ! base64 -D` para decodificar el valor
6. Presionar **i** para insertar o editar el valor
7. Presionar **esc** y luego `:. ! base64` para codificar el valor
8. Editar nuevamente el archivo **i** y dejar la línea en su posición
9. Presionar **esc** y luego **:wq** para guardar y salir


## Configurar secretos de Google Cloud para obtener las imágenes
```sh
# 1. Crear secreto:
kubectl create secret docker-registry gcr-json-key --docker-server=registry_of_images --docker-username=_json_key --docker-password="$(cat 'PATH\airy-gate-419018-3174139c7347.json')" --docker-email=addres@gmail.com

# 2. Path del secreto para que use la llave:
kubectl patch serviceaccounts default -p '{ "imagePullSecrets": [{ "name":"gcr-json-key" }] }'
```

## Exportar y aplicar configuraciones con archivos (secrets en este caso)
```sh
# Para exportar los archivos de configuración
kubectl get secret <nombre> -o yaml > <nombre>.yml

# Aplicar la configuración basado en el archivo
kubectl create -f <nombre>.yml
```


# Setup an ingress controller and Cert Manager in Digital Ocean
```sh
# https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-on-digitalocean-kubernetes-using-helm

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true
kubectl --namespace default get services -o wide -w nginx-ingress-ingress-nginx-controller

# Point the domain to the LoadBalancer IP

# Create an ingress resource
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-kubernetes-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: "hw1.your_domain_name"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service-name-you-want-to-expose
            port:
              number: 3000
```


# Secure the Ingress with Cert Manager
```sh
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.10.1 --set installCRDs=true

# Create a ClusterIssuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Email address used for ACME registration
    email: your_email_address
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Name of a secret used to store the ACME account private key
      name: letsencrypt-prod-private-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx

# Actualizar el Ingress para que use el certificado
annotations:
  kubernetes.io/ingress.class: nginx
  cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - hw1.your_domain_name.com
    secretName: kubernetes-tls

# Verificar el estado del certificado
kubectl describe certificate kubernetes-tls
```
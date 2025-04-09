
# 🛡️ Demo: Detección y Bloqueo de Procesos Sospechosos con Tetragon + eBPF en Kubernetes

Este repositorio contiene una demo completa del uso de **[Tetragon](https://tetragon.io/)** para observar y **bloquear procesos sospechosos en tiempo real** en Kubernetes utilizando **eBPF**.

💻 Esta prueba está diseñada para ejecutarse en un entorno **local con Minikube** (driver Docker) desde **macOS**, pero también puedes adaptarla a cualquier entorno compatible con eBPF.

---

## 🚀 ¿Qué hace esta demo?

- 🕵️‍♂️ Detecta la ejecución de shells (`bash`, `sh`) en contenedores Kubernetes.
- 🔥 Aplica una **TracingPolicy** para bloquearlas automáticamente (sigkill).
- 👀 Visualiza los eventos detectados por Tetragon en tiempo real vía logs.

---

## ⚙️ Requisitos

- macOS con:
  - Docker Desktop instalado y en ejecución
  - [Minikube](https://minikube.sigs.k8s.io/docs/) (`brew install minikube`)
  - [kubectl](https://kubernetes.io/docs/tasks/tools/)
  - [Helm](https://helm.sh/docs/intro/install/)

---

## 📦 Instalación paso a paso

### 1. Iniciar Minikube

```bash
minikube start --driver=docker
```

### 2. Montar `/proc` del host en Minikube (necesario en Docker)

```bash
minikube ssh -- 'sudo mkdir -p /procHost && sudo mount -o bind /proc /procHost'
```

### 3. Instalar Tetragon con Helm

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm install tetragon cilium/tetragon -n kube-system \
  --create-namespace \
  --set tetragon.hostProcPath=/procHost
```

---

## 🧪 Prueba de detección

### Crear un pod de prueba

```bash
kubectl run demo-pod --image=ubuntu:latest -- sleep 1h
```

### Ejecutar bash dentro del pod

```bash
kubectl exec -it demo-pod -- bash
```

### Ver los eventos detectados

```bash
kubectl logs -n kube-system -l k8s-app=tetragon -c export-stdout -f
```

---

## 🔐 Aplicar política de bloqueo

### Archivo: `block-bash.yaml`

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: block-bash
spec:
  kprobes:
  - call: "security_bprm_creds_from_file"
    syscall: false
    args:
    - index: 1
      type: "file"
    selectors:
    - matchArgs:
      - index: 1
        operator: "Equal"
        values:
        - "/bin/bash"
        - "/usr/bin/bash"
      matchActions:
      - action: Sigkill
```

### Aplicar la política

```bash
kubectl apply -f block-bash.yaml
```

### Intentar ejecutar bash nuevamente

```bash
kubectl exec -it demo-pod -- bash
```

🔴 Debería ser bloqueado automáticamente por Tetragon.

---

## 🧹 Limpieza

```bash
kubectl delete -f block-bash.yaml
kubectl delete pod demo-pod
helm uninstall tetragon -n kube-system
minikube stop
```

---

## 📄 Documentación útil

- [Tetragon Docs](https://tetragon.io/docs/)
- [Cilium TracingPolicy API](https://docs.cilium.io/en/stable/network/tracing/)
- [Tetragon GitHub](https://github.com/cilium/tetragon)

---

## 🙋‍♂️ Autor

**Álvaro Nicolás Navarro Castro**  
🔗 [LinkedIn](https://www.linkedin.com/in/alvaronicolasnavarrocastro/)  
🌐 [Darts TI](https://www.darts-ti.cl/) – Soluciones DevOps, Cloud y Seguridad para tu negocio

---

> ¿Te sirvió este repositorio? ¡Dale una ⭐ en GitHub o comparte con tu equipo!

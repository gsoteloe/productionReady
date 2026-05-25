---
title: "Por qué los pods no se conectan: un modelo mental para el networking en Kubernetes"
date: 2026-03-16
tags: ["kubernetes", "networking", "debugging", "SRE"]
description: "Antes de correr kubectl describe, necesitas un modelo de cómo fluye el tráfico en Kubernetes. Este artículo construye ese modelo desde los fundamentos."
---

A todos nos ha pasado: la primera vez que un pod no puede conectarse a otro, la reacción natural es buscar
síntomas: revisar logs, describir el pod, reiniciar deployments. A veces funciona.
Pero el problema con ese enfoque es que depende del síntoma visible, no de un
entendimiento del sistema. Cuando el síntoma cambia —y en producción siempre cambia—
el proceso empieza de cero.

Este artículo no es como tal un checklist de troubleshooting, sino más bien **el modelo mental** que debería
preceder a cualquier checklist.

## El contrato fundamental de networking en Kubernetes

Kubernetes hace una promesa explícita sobre la red muy bien documentada en su especificación:

- Cualquier pod puede comunicarse con cualquier otro pod sin NAT.
- Los nodos pueden comunicarse con cualquier pod sin NAT.
- La IP que un pod ve como propia es la misma que los demás ven.

Esto parece obvio hasta que lo comparas con cómo funciona Docker por default, donde
cada contenedor vive en una red privada y necesita port mappings para ser alcanzable.
Kubernetes elimina esa complejidad, pero alguien (llamado *CNI*) tiene que implementar ese contrato.

## La cadena que mueve un paquete

Cuando el proceso en `pod-a` hace `curl http://pod-b:8080`, el paquete pasa por
varias capas antes de llegar. Entender esas capas es lo que separa el debugging
sistemático de uno improvisado.

```
pod-a (proceso)
  └── veth pair
      └── dataplane del nodo (CNI + kube-proxy / eBPF)
          └── routing entre nodos
              └── veth pair de pod-b
                  └── pod-b (proceso)
```

*veth pair*: el par de interfaces virtuales que conecta el namespace de red del pod con el nodo.
*dataplane del nodo*: el CNI (Cilium, Flannel, Calico) más kube-proxy, responsable del forwarding entre pods y de traducir el ClusterIP a una IP de pod real cuando el destino es un Service.

Cada capa tiene su propio modo de fallo. Un paquete que no llega puede estar
"muriendo" en cualquiera de esos puntos, y los síntomas son distintos en cada caso.

### La veth pair

Cuando el CNI crea un pod, genera un par de interfaces virtuales: una vive dentro
del namespace de red del pod (`eth0`), la otra en el nodo (`cali1234abcd` en  el caso de Calico,
`lxc1234abcd` en el de Cilium). Son los dos extremos del mismo cable virtual.

Si esta interfaz no existe o está en el estado incorrecto, el pod ni siquiera puede
enviar un paquete. Lo verías como timeout inmediato desde dentro del pod, incluso
hacia su propio gateway.

```bash
# Dentro del pod
ip addr show eth0
ip route show

# En el nodo, buscar la interfaz correspondiente
ip link show | grep -E 'cali|lxc|veth'
```

### El CNI

El CNI es el componente que más varía entre clusters. En un cluster con Cilium, el forwarding entre pods ocurre en eBPF, bypaseando el stack de red tradicional del kernel (netfilter, iptables). 

En un cluster con Flannel, el tráfico entre nodos se encapsula en VXLAN. Esto importa porque cambia qué herramientas son útiles para observar el tráfico.

`tcpdump` sigue siendo útil en ambos casos: puedes capturar en la interfaz lxc* del nodo y ver el tráfico. Su limitación es de contexto: ves paquetes, pero no sabes qué política los bloqueó ni a qué identidad pertenece cada endpoint. Hubble expone exactamente eso: flujos con identidades, decisiones de política, y drops con la razón explícita.

```bash
# Con Cilium: observar flujos en tiempo real
hubble observe --pod pod-a --follow

# Verificar que el CNI está operacional
cilium status
cilium endpoint list
```

### El Service y la resolución de ClusterIP

Aquí está la fuente de confusión más común: el `ClusterIP` de un Service no es
una IP real. No hay ningún proceso escuchando en esa dirección. Es una IP virtual
que kube-proxy (o Cilium en modo eBPF) intercepta y traduce a la IP de uno de
los pods del Endpoint.

Cuando haces `curl http://mi-servicio:8080`, lo que ocurre es:

1. DNS resuelve `mi-servicio` al ClusterIP (por ejemplo, `10.96.43.21`).
2. El paquete sale del pod con destino `10.96.43.21:8080`.
3. kube-proxy (via iptables) o Cilium (via eBPF) intercepta ese paquete antes
   de que salga del nodo.
4. La IP destino se reescribe a la IP de un pod real del Service.
5. El paquete viaja hacia ese pod.

Este mecanismo tiene una implicación importante: si un Service no tiene Endpoints,
el paquete llega a la regla de iptables/eBPF y no tiene un destino válido. El síntoma
observable depende de cómo el dataplane implemente la ausencia de backends: algunas
implementaciones emiten un TCP RST; otras, dejan el paquete en blackhole produciendo
timeout; otras responden con ICMP unreachable. 
```bash
# Verificar que el Service tiene endpoints
kubectl get endpoints mi-servicio

# Si está vacío, el problema está en el selector del Service o en los pods
kubectl describe service mi-servicio
kubectl get pods -l app=mi-app  # usar los labels del selector
```

### DNS: el culpable silencioso

Antes de que el paquete se mueva, el nombre tiene que resolverse. CoreDNS vive
en el cluster como un Deployment y es el *resolver* de todos los pods. Dos fuentes
de problemas que no son obvias:

**El valor de `ndots`**. Por defecto, Kubernetes configura `ndots: 5` en el
`/etc/resolv.conf` de cada pod. Esto significa que cualquier nombre con menos
de 5 puntos se busca primero añadiendo los search domains (`svc.cluster.local`,
`cluster.local`, etc.) antes de intentarlo como nombre absoluto. Un query a
`api.github.com` genera cuatro queries DNS adicionales antes de resolverse como
nombre externo. Un servicio interno como `redis.internal` genera aún más, porque
primero se prueban todas las combinaciones con sufijos del cluster. La solución
rápida cuando sabes que un nombre es absoluto: añadir un punto final
(`curl http://api.github.com./`) fuerza resolución directa sin buscar sufijos.

**CoreDNS sobrecargado**. En clusters grandes, CoreDNS puede convertirse en un
cuello de botella. Los síntomas son timeout en resolución DNS que se manifiestan
como errores de conexión intermitentes que no reproducen de forma consistente.

```bash
# Verificar resolución DNS desde un pod
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  nslookup mi-servicio.mi-namespace.svc.cluster.local

# Ver el resolv.conf que Kubernetes inyecta
kubectl exec pod-a -- cat /etc/resolv.conf

# Logs de CoreDNS
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

## El framework de diagnóstico

Con el modelo claro, el proceso de debugging tiene una estructura natural:
descartar capas de adentro hacia afuera.

**Paso 1 — ¿El pod puede conectarse a sí mismo?**

```bash
kubectl exec pod-a -- curl localhost:8080
```

Si falla, el problema es la aplicación, no la red.

**Paso 2 — ¿Puede resolver DNS?**

```bash
kubectl exec pod-a -- nslookup mi-servicio
```

Si falla, el problema está en CoreDNS o en la configuración DNS del pod.

**Paso 3 — ¿El Service tiene Endpoints?**

```bash
kubectl get endpoints mi-servicio
```

Si está vacío, hay dos lugares donde buscar: el selector del Service o el estado de los pods.

```bash
kubectl describe service mi-servicio
kubectl get pods -l app=mi-app  # usar los labels del selector
```

**El caso que más se pasa por alto: pods Running pero no Ready**

Un pod puede estar en estado `Running` y ser completamente invisible al Service al mismo tiempo.
Kubernetes excluye de los Endpoints cualquier pod cuya readiness probe esté fallando, aunque
el proceso esté vivo. Es el escenario más silencioso de todos: no hay error evidente,
los pods aparecen en verde, y la conexión simplemente no llega.

```bash
# Ver el estado de Ready específicamente
kubectl get pods -l app=mi-app -o wide
kubectl describe pod pod-b | grep -A 10 'Conditions'

# Si hay readiness probe fallando, los eventos lo muestran
kubectl describe pod pod-b | grep -A 5 'Events'
```

> En clusters modernos (Kubernetes 1.21+), `kubectl get endpoints` es un alias sobre
> Endpoint Slices. El comando sigue funcionando igual; si quieres ver la representación
> real: `kubectl get endpointslices -l kubernetes.io/service-name=mi-servicio`.

**Paso 4 — ¿Puede conectarse al ClusterIP directamente?**

```bash
CLUSTER_IP=$(kubectl get svc mi-servicio -o jsonpath='{.spec.clusterIP}')
kubectl exec pod-a -- curl $CLUSTER_IP:8080
```

Si falla pero el DNS resuelve y hay Endpoints, el problema está en kube-proxy
o en la capa de eBPF del CNI.

**Paso 5 — ¿Puede conectarse directamente a la IP del pod destino?**

```bash
POD_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')
kubectl exec pod-a -- curl $POD_IP:8080
```

Si esto funciona pero el ClusterIP no, el problema está en el Service o en
kube-proxy. Si tampoco funciona, el problema está en el CNI o en el routing
entre nodos.

## Lo que este modelo no cubre

Dos capas pueden bloquear tráfico incluso cuando todo lo anterior funciona: **Network Policies**
(un deny-all es invisible hasta el Paso 5, donde el paquete simplemente no llega) y
**service meshes** como Linkerd, donde un proxy sidecar mal configurado o un certificado
mTLS expirado rompe conectividad que a nivel de red está perfecta. Cilium con Hubble hace
ambos casos observables: puedes ver los drops y la política o identidad que los causa.
Ambos temas merecen su propio artículo.

## Un ejemplo real de fallo

El escenario más común que ilustra todo lo anterior junto:

Un equipo reporta que su servicio `payments-api` dejó de responder después de un
deploy. Los pods están en estado `Running`. El Service existe. Los logs no muestran
errores de aplicación. El on-call reinicia los pods y ocurre exactamente lo mismo.

El framework aplicado en orden:

```bash
# Paso 1: ¿hay endpoints?
kubectl get endpoints payments-api
# NAME           ENDPOINTS   AGE
# payments-api   <none>      2m
```

Endpoints vacíos con pods en Running. Readiness probe:

```bash
kubectl describe pod payments-api-7d9f8b-xkp2q | grep -A 15 'Conditions\|Readiness'
# Readiness:  http-get http://:8080/healthz delay=0s timeout=1s period=10s
# ...
# Readiness probe failed: HTTP probe failed with statuscode: 503
```

El nuevo deploy introdujo una dependencia a una variable de entorno que no estaba
configurada en el entorno de staging. La aplicación arrancaba, pero `/healthz`
devolvía 503 porque no podía conectarse a su base de datos. Kubernetes, correctamente,
mantenía el pod fuera de los Endpoints.

El reinicio no ayudaba porque el problema no era el pod sino la configuración.
Sin el modelo mental, ese diagnóstico **podría tomar horas**.

## Conclusión

El networking en Kubernetes parece complejo solo cuando se observa como un conjunto de componentes separados. En realidad es un pipeline determinístico porque cada fallo tiene una ubicación exacta. 

El debugging deja de ser intuición cuando puedes responder: ¿en qué punto se murió el paquete?

El modelo no elimina la complejidad; la hace navegable.

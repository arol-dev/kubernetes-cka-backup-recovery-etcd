# Estrategias de Backup y Recuperación de etcd Lab - CKA Course

Este laboratorio práctico está diseñado para aprender a implementar estrategias de backup y restauración de etcd, un componente esencial en el control plane de Kubernetes. Asegúrate de seguir cada paso cuidadosamente y practicar varias veces para estar completamente preparado para el examen.

## Prerrequisitos

- **VirtualBox** (se necesita **Python** y **pywin32** como prerrequisitos).
- **Vagrant**.
- **MobaXterm** para sesiones SSH.

## Requisitos

- Acceso a un nodo de control (controlplane) con permisos para ejecutar comandos de Kubernetes y etcd.
- Cliente `etcdctl` instalado.
- Conocimiento básico del uso de `kubectl` y edición de archivos YAML.

## Objetivos

1. Realizar una copia de seguridad (snapshot) de etcd.
2. Restaurar la copia de seguridad en un nuevo directorio de datos.
3. Modificar el manifiesto de etcd para usar el nuevo directorio.

## Contenido del Repositorio

Este repositorio incluye:

- Una carpeta `scripts` con dos scripts que proporcionan soporte durante el laboratorio.
- Un fichero `Vagrantfile` que permite automatizar el despliegue de tres VMs en VirtualBox.

Las VMs consisten en:

- 1 nodo master.
- 2 nodos worker.

## Paso 1: Despliegue de las VMs

1. Clona el repositorio en tu entorno local:

   ```bash
   git clone https://github.com/arol-dev/kubernetes-cka-backup-recovery-etcd.git
   cd kubernetes-cka-backup-recovery-etcd
   ```

2. Dentro del repositorio, ejecuta el siguiente comando para desplegar las VMs:

   ```bash
   vagrant up
   ```

   Esto comenzará a desplegar tres VMs en VirtualBox: un nodo master y dos worker nodes. Espera unos minutos para que el proceso termine.

3. Verifica el estado de las VMs con:

   ```bash
   vagrant status
   ```

   Asegúrate de que las tres máquinas estén en estado `running`.

4. Obtén la configuración SSH para conectarte a las máquinas:

   ```bash
   vagrant ssh-config
   ```

   Guarda los detalles proporcionados, ya que los necesitarás en el siguiente paso.

## Paso 2: Conectar a las VMs con MobaXterm

1. Abre **MobaXterm** y utiliza la configuración SSH obtenida anteriormente para conectarte a las tres máquinas.
   - No se requiere un usuario específico, deja el campo vacío.
   - Si se te solicita usuario o contraseña, utiliza la cadena `vagrant`.

## Paso 3: Desplegar la Aplicación BookInfo de Istio

1. Crea el namespace para BookInfo:
   ```bash
   kubectl create ns bookinfo
   ```
2. Despliega la aplicación:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
   ```

3. Verifica que los pods se hayan desplegado correctamente en el namespace bookinfo.
   ```bash
   kubectl get pods -n bookinfo
   ```

4. Expone la aplicación localmente (opcional):
   ```bash
   kubectl port-forward svc/productpage 9080:9080 -n bookinfo
   ```
5. Prueba la aplicación desde el navegador (opcional):
   - [http://127.0.0.1:9080/productpage](http://127.0.0.1:9080/productpage)

6. Si la aplicación se ve lenta o incorrecta, reinicia el `coredns` (opcional):
   ```bash
   kubectl -n kube-system rollout restart deployment coredns
   ```

## Paso 4: Instalación del Cliente etcdctl

Para realizar operaciones de backup y restauración, necesitas el cliente `etcdctl`. En el **nodo Master**, instálalo con:

```bash
sudo apt install etcd-client
```

Verifica la version de etcd-client:

```bash
etcdctl version
```

## Paso 5: Realizar una Copia de Seguridad de etcd

1. **Identificar el pod de etcd**: Ejecuta el siguiente comando para identificar el pod etcd en el cluster.

   ```bash
   kubectl get pods -n kube-system
   ```

2. **Obtener las ubicaciones de los certificados y la clave**: Necesitarás los certificados para realizar una copia de seguridad. Usa:

   ```bash
   kubectl describe pod etcd-controlplane -n kube-system
   ```

   Busca el valor de la opción `--listen-client-urls` para la URL del endpoint. El certificado del servidor se encuentra en `/etc/kubernetes/pki/etcd/server.crt`, definido por la opción `--cert-file`. 
   El certificado CA se puede encontrar en `/etc/kubernetes/pki/etcd/ca.crt`, especificado por la opción `--trusted-ca-file`.

3. **Realizar la copia de seguridad**:

   Ejecuta el siguiente comando para realizar la copia de seguridad en un archivo snapshot.

   ```bash
   sudo ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key
   ```

   Este comando crea un snapshot de etcd en la ruta `/opt/etcd-backup.db`.

   Verificamos que el snapshot se haya creado correctamente, ejecutando el siguiente comando:

   ``bash
   sudo etcdctl --write-out=table snapshot status /opt/etcd-backup.db
   ```

   Ten en cuenta que la opción `--endpoints` no es necesaria, ya que estamos ejecutando el comando en el mismo servidor que etcd.

## Paso 6: Restaurar la Copia de Seguridad

1. **Eliminar el deployment anterior** (opcional):

   ```bash
   kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo   
   ```

2. **Restaurar la copia de seguridad**: Para restaurar la copia de seguridad, utiliza el siguiente comando:

   ```bash
   sudo ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
      --data-dir=/var/lib/from-backup
   ```

   Este comando restaurará el snapshot en el directorio `/var/lib/from-backup`. 

   El parametro `--data-dir` de `etcdctl` se utiliza para especificar la **ubicación del directorio de datos** donde etcd almacena su información. Este directorio contiene todos los datos de estado persistente de etcd, incluyendo:

   - **Registros de transacciones** (logs) que se utilizan para mantener la consistencia de las operaciones en etcd.
   - **Snapshots** que representan una copia completa del estado del cluster en un momento dado.

   En el contexto de **restaurar una copia de seguridad (snapshot)** de etcd, el parámetro `--data-dir` indica dónde se deben almacenar los datos restaurados. Esto es importante porque, al realizar la restauración, estamos creando un nuevo estado inicial de etcd a partir de una copia de seguridad, y necesitamos un directorio vacío para poder volcar los datos restaurados. 

   En general:

   - **Backup**: Cuando tomas un snapshot, etcdctl almacena la copia en un archivo específico, pero no interactúa directamente con el directorio de datos.
   - **Restore**: Cuando restauras desde un snapshot, el parámetro `--data-dir` permite definir un nuevo directorio donde se reconstituirá el estado del cluster a partir del snapshot.

## Paso 7: Modificar el Manifiesto de etcd

Para aplicar la restauración en el clúster, debes modificar el manifiesto del pod etcd para que apunte al nuevo directorio de datos.

1. **Editar el manifiesto**:

   ```bash
   sudo vi /etc/kubernetes/manifests/etcd.yaml
   ```

2. **Cambiar el directorio de datos**: 

- Modifica el valor del atributo `--data-dir` en la seccion `containers.command` del valor original `/var/lib/etcd` a `/var/lib/from-backup`.
- Modifica el valor del atributo `volumeMounts.mountPath` con el nombre `etcd-data` del valor original `/var/lib/etcd` a `/var/lib/from-backup`.
- Modifica el valor del atributo `spec.volumes.hostPath` con el nombre `etcd-data` del valor original `/var/lib/etcd` a `/var/lib/from-backup`.

3. **Guardar los cambios**: Cuando se guarda este archivo, el pod de etcd se destruirá y se volverá a crear automáticamente, utilizando los nuevos datos restaurados.

## Paso 8: Verificación

1. **Verificar el estado del pod**:

   ```bash
   kubectl get pods -n kube-system -o wide | grep etcd
   ```

   Asegúrate de que el pod esté en estado `Running`.

2. **Verificar los registros**:

   ```bash
   kubectl logs -n kube-system etcd-controlplane
   ```

   Revisa los registros para asegurarte de que no haya errores relacionados con la restauración.

   ```bash
   kubectl get pods -n bookinfo
   ``` 

   Deberías ver que los pods de la aplicación BookInfo se han desplegado nuevamente porque estaban presentes en el momento en que se creó el snapshot.

## Consejos Finales

- Practica el proceso varias veces para que puedas realizarlo rápidamente en el examen.
- Familiarízate con la estructura del archivo YAML de etcd.
- Recuerda siempre modificar el valor del atributo `spec.volumes.hostPath` con el nombre `etcd-data` en el manifiesto para aplicar la restauración.

¡Espero que este laboratorio te sea útil en tu preparación para el examen CKA! Si tienes alguna pregunta o deseas practicar más escenarios, no dudes en mencionarlo.
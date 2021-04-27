# Linux-5.3---Privilege-Escalation-
Linux 5.3 - Privilege Escalation via io_uring Offload of sendmsg() onto Kernel Thread with Kernel Creds 


Desde el compromiso 0fa03c624d8f ("io_uring: agregar soporte para sendmsg ()", primero en v5.3),
io_uring tiene soporte para llamar asincrónicamente a sendmsg ().
Las tareas del espacio de usuario sin privilegios pueden enviar la cola de envío IORING_OP_SENDMSG
entradas, que hacen que se llame a sendmsg () en el contexto syscall en el
tarea original o, si no pudo enviar un mensaje sin bloquear, en
un hilo de trabajo del núcleo.

El problema es que sendmsg () puede terminar mirando las credenciales del
llamar a la tarea por varias razones; por ejemplo:

 - sendmsg () con no nulo, no abstracto -> msg_name en un AF_UNIX no conectado
   El socket de datagrama termina realizando comprobaciones de acceso al sistema de archivos
 - sendmsg () con SCM_CREDENTIALS en un socket AF_UNIX termina mirando
   procesar credenciales
 - sendmsg () con no nulo -> msg_name en un socket AF_NETLINK termina funcionando
   verificaciones de capacidad contra el proceso de llamada

Cuando la solicitud se ha transferido a una tarea de trabajo del kernel, todas estas comprobaciones
se realizan con las credenciales del trabajador, que son el kernel predeterminado
creds, con UID 0 y capacidades completas.

Para forzar a io_uring a entregar una solicitud a un hilo de trabajo del kernel, un atacante
puede abusar del hecho de que el campo de código de operación del SQE se lee varias veces, con
accede a la estructura msghdr en el medio: el atacante puede enviar primero un SQE
de tipo IORING_OP_RECVMSG cuya estructura msghdr está en una región userfaultfd, y
luego, cuando se active userfaultfd, cambie el tipo a IORING_OP_SENDMSG.

Aquí hay un reproductor para Linux 5.3 que demuestra el problema agregando un
Dirección IPv4 a la interfaz de bucle invertido sin tener los privilegios necesarios
para eso. 

||----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------||
A mi modo de ver, la forma más fácil de solucionar este problema sería probablemente tomar un
referencia a las credenciales de la persona que llama con get_current_cred () en
io_uring_create (), luego deje que el código de entrada de todos los hilos de trabajo del kernel
instale permanentemente estos como sus credenciales subjetivas con override_creds ().
(O tal vez commit_creds () - eso significaría que realmente podría ver el
usuario propietario de estos hilos en la salida de algo como "ps aux". Sobre el
Por otro lado, no estoy seguro de cómo eso afecta cosas como el envío de señales, así que
override_creds () podría ser más seguro.) Significaría que no puede usar un
io_uring instancia a través de algo así como una transición setuid () que cae
privilegios, pero probablemente eso no sea un gran problema?

Si bien el error de seguridad solo se introdujo mediante la adición de IORING_OP_SENDMSG,
Probablemente sería beneficioso marcar un cambio de este tipo para respaldar todos los
camino a v5.1, cuando se agregó io_uring, creo, por ejemplo, el gancho SELinux que es
llamado desde rw_verify_area () hasta ahora siempre ha atribuido todas las operaciones de E / S
al contexto del kernel, que no es realmente un problema de seguridad, pero podría, por ejemplo,
causar denegaciones inesperadas dependiendo de la política de SELinux.

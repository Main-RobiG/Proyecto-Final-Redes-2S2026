# Manual Técnico — Configuración de Servicios Red/Servidor

## Proyecto Final — Fase 1

---

## Carátula

**Universidad:** [Universidad De San Carlos De Guatemala (USAC)]
**Facultad/Escuela:** Escuela de Ciencias y Sistemas
**Curso:** [Practicas Iniciales]
**Catedrático/Auxiliar:** [Inga. Floriza Felipa
Ávila de Medinilla]

**Integrantes:**
- [Sergio Roberto Gudiel Sian] — Servidor de Usuarios (AD, DNS, DHCP, GPOs)
- [Rudin Alexander Lopez Salvatierra] — File Server

**Fecha de entrega:** 24 de septiembre de 2026

---

## Tabla de contenido

1. [Introducción](#introducción)
2. [Objetivos](#objetivos)
3. [Arquitectura de la red](#arquitectura-de-la-red)
4. [1.1 Servidor de Virtualización](#11-servidor-de-virtualización)
5. [1.2 Servidor de Identidad y Autenticación (Active Directory)](#12-servidor-de-identidad-y-autenticación-active-directory)
6. [1.3 Servidor de Asignación Dinámica de IP (DHCP)](#13-servidor-de-asignación-dinámica-de-ip-dhcp)
7. [1.4 Servidor de Archivos (File Server)](#14-servidor-de-archivos-file-server)
8. [Pruebas de funcionamiento](#pruebas-de-funcionamiento)
9. [Conclusiones](#conclusiones)

---

## Introducción

El presente manual documenta la configuración de una infraestructura de red empresarial implementada en un entorno virtualizado, utilizando dos servidores Windows Server 2022 distribuidos en dos máquinas físicas independientes, interconectadas mediante una red virtual privada (Radmin VPN). El objetivo de esta Fase 1 es implementar los servicios de identificación centralizada (Active Directory y DNS), asignación dinámica de direcciones IP (DHCP), y gestión de recursos compartidos (File Server), integrados bajo el dominio `servidor1.com`.

---

## Objetivos

- Implementar un controlador de dominio (AD DS) con resolución de nombres DNS propia.
- Configurar políticas de grupo (GPO) que se apliquen automáticamente a los usuarios del dominio.
- Configurar un servidor DHCP que asigne direcciones IP dinámicamente dentro de un rango definido.
- Configurar un servidor de archivos con recursos compartidos y permisos diferenciados.
- Integrar un cliente Linux (Ubuntu) al dominio, demostrando autenticación centralizada.

---

## Arquitectura de la red

| Rol | Equipo | Sistema Operativo | IP |
|---|---|---|---|
| Controlador de Dominio (DC1) | Laptop 1 | Windows Server 2022 | 192.168.0.20 |
| Servidor miembro / File Server (SRV2) | Laptop 2 | Windows Server 2022 | 192.168.0.21 |
| Cliente Linux | Laptop 1 (VM) | Ubuntu Desktop | Dinámica (reservada 192.168.0.30) |
| Interconexión | Radmin VPN | — | Red virtual `26.x.x.x` |

**Dominio:** `servidor1.com`
**Rango DHCP:** 192.168.0.1 – 192.168.0.100

![Diagrama o captura de las dos VMs en VirtualBox](./images/01-virtualbox-vms.png)

---

## 1.1 Servidor de Virtualización

Se utilizaron dos computadoras físicas, cada una con Oracle VirtualBox instalado, alojando una VM de Windows Server 2022 y, en el caso de la Laptop 1, adicionalmente una VM de Ubuntu Desktop como cliente.

![Windows Server instalado - información de versión](./images/02-windows-server-instalado.png)

![Ubuntu Desktop instalado - información de versión](./images/03-ubuntu-instalado.png)

### Configuración inicial: nombre de equipo y red

Al instalar Windows Server, el sistema asigna automáticamente un nombre de equipo genérico (aleatorio). Este se renombró a **SRV1**, nombre bajo el cual el servidor sería identificado dentro del dominio como controlador. Adicionalmente, se configuró el adaptador de red en modo **Red interna** (compartida con el cliente Ubuntu) y se le asignó una dirección **IP estática 192.168.0.20/24**, con el DNS preferido apuntando al propio servidor — requisito indispensable para que los servicios que dependen de resolución de nombres (AD DS, DHCP, File Server) funcionen de manera consistente.

![Panel del Administrador del servidor recién instalado](./images/01a-panel-administrador-servidor.png)

![Nombre de equipo original asignado por el instalador](./images/01b-nombre-equipo-original.png)

![Nombre de equipo cambiado a SRV1](./images/01c-nombre-equipo-renombrado.png)

![Configuración de IP estática y DNS preferido apuntando al propio servidor](./images/01d-ip-estatica-dns.png)

---

## 1.2 Servidor de Identidad y Autenticación (Active Directory)

### ¿Qué es y para qué sirve?

Active Directory Domain Services (AD DS) es el servicio de Microsoft que centraliza la gestión de usuarios, equipos y políticas de seguridad dentro de una red. En vez de crear cuentas de usuario por separado en cada computadora, AD permite que un usuario tenga una sola identidad (`usuario1@servidor1.com`) válida en toda la red. El servicio DNS es el que traduce el nombre del dominio (`servidor1.com`) a la dirección IP real del servidor, y es un requisito técnico para que Active Directory funcione correctamente (AD depende de DNS para localizar sus propios controladores de dominio).

### Instalación y promoción a controlador de dominio

Se instaló el rol **Servicios de dominio de Active Directory (AD DS)** desde el Administrador del servidor. Una vez instalado el rol, se promovió el servidor a controlador de dominio mediante la creación de un **nuevo bosque** con nombre de dominio raíz `servidor1.com`, habilitando en el mismo proceso el servicio de **Servidor DNS** y el **Catálogo global (GC)** — ambos necesarios para que el controlador de dominio pueda localizarse a sí mismo y resolver nombres dentro de la red.

![Rol de AD DS instalado, con el enlace para promoverlo a controlador de dominio](./images/04a-rol-adds-instalado.png)

![Opciones del controlador de dominio: creación del nuevo bosque servidor1.com, con Servidor DNS y Catálogo global habilitados](./images/04b-opciones-controlador-dominio.png)

![Resumen de revisión de opciones antes de instalar: dominio servidor1.com, nombre NetBIOS SERVIDOR1 y servicio DNS incluido](./images/04c-revision-opciones-instalacion.png)

### Validación del controlador de dominio con dcdiag

Tras completar la promoción y el reinicio del servidor, se ejecutó el comando `dcdiag` desde símbolo del sistema como administrador. Esta herramienta ejecuta una batería de pruebas de diagnóstico (conectividad, replicación, servicios, particiones del esquema, DNS, etc.) para confirmar que el controlador de dominio quedó completamente operativo y sin errores tras la instalación.

![Resultado de dcdiag (parte 1): pruebas de conectividad, replicación y servicios superadas](./images/04e-dcdiag-parte1.png)

![Resultado de dcdiag (parte 2): pruebas de particiones (Schema, Configuration, ForestDnsZones, DomainDnsZones) y del dominio servidor1.com superadas](./images/04f-dcdiag-parte2.png)

![Panel final del Administrador del servidor mostrando AD DS y DNS instalados y en funcionamiento](./images/04h-panel-final-adds-dns.png)

### Configuración del dominio y DNS

Con el controlador de dominio ya operativo, se verificó el estado del dominio y la resolución de nombres mediante comandos de PowerShell, así como visualmente desde el Administrador de DNS, confirmando la zona de búsqueda directa `servidor1.com` con sus registros SOA (Inicio de autoridad), NS (Servidor de nombres) y A (Host) apuntando correctamente a `192.168.0.20`.

![Administrador de DNS mostrando la zona servidor1.com con sus registros SOA, NS y Host (A)](./images/04d-dns-zona-servidor1.png)

![Verificación del dominio con Get-ADDomain](./images/04-get-addomain.png)

![Resolución DNS del dominio con nslookup](./images/06-nslookup-dominio.png)

![Configuración de IP y DNS del servidor](./images/07-ipconfig-dns.png)

### Cuentas de usuario

Por defecto, Active Directory exige que toda contraseña de usuario cumpla con requisitos de complejidad (combinación de mayúsculas, minúsculas, números y símbolos, longitud mínima, etc.), definidos en la política **Default Domain Policy**. Dado que el proyecto requiere que las cuentas de usuario utilicen exactamente la contraseña `Password`, la cual no cumple dichos requisitos, fue necesario **deshabilitar la directiva de complejidad de contraseñas** desde la Administración de directivas de grupo antes de crear las cuentas.

![Directiva "La contraseña debe cumplir los requisitos de complejidad" deshabilitada en la Default Domain Policy](./images/04g-politica-contrasena-deshabilitada.png)

Con la directiva ajustada, se crearon las cuentas `usuario1` y `usuario2` dentro de la unidad organizativa de Usuarios del dominio, ambas con su UserPrincipalName (UPN) correspondiente y habilitadas para inicio de sesión.

![Cuentas usuario1 y usuario2 creadas en Usuarios y equipos de Active Directory](./images/05a-usuarios-adyc-consola.png)

![Propiedades de la cuenta usuario1, con inicio de sesión usuario1@servidor1.com](./images/05b-propiedades-usuario1.png)

![Propiedades de la cuenta usuario2, con inicio de sesión usuario2@servidor1.com](./images/05c-propiedades-usuario2.png)

![Listado de usuarios del dominio con UPN, verificado por PowerShell](./images/05-get-adusers.png)

### Políticas de Grupo (GPO)

Las Directivas de Grupo (GPO) permiten aplicar configuraciones automáticas a todos los usuarios y equipos del dominio, sin tener que configurar cada máquina manualmente. Se configuraron dos GPO:

**GPO 1 — Fondo de pantalla institucional:** obliga a que, al iniciar sesión cualquier usuario del dominio, se establezca el logo de la universidad como fondo de escritorio. La imagen se aloja en una carpeta compartida en red (`\\SRV1\Recursos\fondo.jpg`) para que sea accesible desde cualquier equipo del dominio.

![Carpeta compartida Recursos creada](./images/13-carpeta-recursos-compartida.png)

![Configuración de la GPO de fondo de pantalla](./images/08-gpo-fondopantalla-config.png)

**GPO 2 — Política de navegador (Google Chrome / Microsoft Edge):** establece la página de inicio institucional y bloquea la instalación de extensiones no autorizadas, mediante plantillas administrativas ADMX importadas al servidor.

![Configuración de página de inicio en Chrome](./images/10-gpo-chrome-homepage.png)

![Bloqueo de extensiones en Chrome](./images/11-gpo-chrome-extensions-block.png)

![Bloqueo de extensiones en Edge](./images/12-gpo-edge-extensions-block.png)

![Listado completo de GPOs configuradas](./images/09-gpo-lista-completa.png)

### Cliente Linux integrado al dominio

Se unió una máquina cliente Ubuntu Desktop al dominio mediante `realmd`/`sssd`, permitiendo la autenticación centralizada de usuarios del dominio directamente desde Linux.

![Login exitoso con usuario1 desde Ubuntu](./images/14-login-usuario1-ubuntu.png)

![Verificación de sesión de dominio activa (whoami / id)](./images/15-whoami-id-ubuntu.png)

---

## 1.3 Servidor de Asignación Dinámica de IP (DHCP)

### ¿Qué es y para qué sirve?

DHCP (Dynamic Host Configuration Protocol) automatiza la asignación de direcciones IP a los equipos de la red, evitando tener que configurar cada IP manualmente. Además de la IP, el servidor también entrega automáticamente otros datos necesarios para la conectividad, como el servidor DNS a utilizar y la puerta de enlace.

### Configuración del ámbito (scope)

Se configuró un ámbito DHCP con el rango 192.168.0.1 – 192.168.0.100, excluyendo la IP del propio servidor (192.168.0.20) para evitar conflictos.

![Ámbito DHCP configurado](./images/16-dhcp-scope.png)

![Rango de exclusión configurado](./images/17-dhcp-exclusion.png)

### Opciones del DHCP

Se configuraron las opciones de DNS, dominio y gateway que se entregan automáticamente a cada cliente.

![Opciones DHCP configuradas (DNS, dominio, gateway)](./images/18-dhcp-options.png)

### Reservas de IP

Se reservaron direcciones IP fijas para equipos clave de la red (cliente Ubuntu y servidor de archivos), garantizando que siempre reciban la misma IP.

![Reservas DHCP configuradas](./images/19-dhcp-reservations.png)

---

## 1.4 Servidor de Archivos (File Server)

### ¿Qué es y para qué sirve?

Un servidor de archivos centraliza el almacenamiento de documentos, permitiendo que varios usuarios accedan a ellos desde la red con distintos niveles de permiso, en lugar de guardar copias sueltas en cada computadora. Se utilizan dos capas de permisos: los permisos de **recurso compartido (SMB)**, que controlan el acceso por red, y los permisos **NTFS**, que controlan el acceso a nivel de sistema de archivos — ambos deben configurarse de forma coherente.

### Carpeta Pública

Recurso compartido accesible para todos los usuarios autenticados del dominio, con permisos de lectura, creación, modificación y eliminación.

![Carpeta pública creada y compartida](./images/20-fileserver-carpetas-creadas.png)

![Configuración del recurso compartido](./images/22-fileserver-share-config.png)

### Carpeta Privada

Recurso compartido restringido, accesible únicamente por el usuario `privado`, con permisos exclusivos de lectura, modificación y eliminación.

![Permisos NTFS configurados en carpeta privada](./images/21-fileserver-permisos-ntfs.png)

### Prueba de acceso

Se validó el correcto funcionamiento de los permisos accediendo a ambas carpetas desde otra máquina de la red.

![Prueba de acceso a las carpetas compartidas](./images/23-fileserver-prueba-acceso.png)

---

## Pruebas de funcionamiento

Para validar que todos los servicios funcionan de manera integrada, se realizaron las siguientes pruebas:

**1. Servidor miembro unido al dominio:** se confirmó que el servidor de archivos (SRV2) quedó correctamente unido a `servidor1.com` como servidor miembro.

![SRV2 listado como equipo del dominio](./images/24-servidor-miembro-unido-dominio.png)

**2. Aplicación de GPOs sobre un usuario real:** se inició sesión con `usuario1@servidor1.com` en una máquina Windows del dominio, confirmando que el fondo de pantalla institucional y la página de inicio del navegador se aplican automáticamente, sin intervención manual.

![Fondo de pantalla aplicado al loguear como usuario del dominio](./images/25-login-usuario-fondo-pantalla.png)

![Página de inicio institucional cargada automáticamente en el navegador](./images/26-chrome-homepage-aplicada.png)

---

## Conclusiones

La implementación de esta infraestructura permitió comprender de forma práctica cómo interactúan entre sí los distintos servicios de una red empresarial: la resolución de nombres (DNS) es un prerrequisito técnico para Active Directory, las políticas de grupo dependen de que los clientes estén correctamente unidos al dominio y sean Windows (no aplican a clientes Linux, que solo participan en la autenticación vía Kerberos/LDAP a través de `sssd`), y el servidor DHCP debe coordinarse cuidadosamente con las IPs fijas ya asignadas en la red para evitar conflictos de direccionamiento.

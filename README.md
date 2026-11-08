# Hypervisor-GPU-Passthrough
Arquitectura Proxmox Ampere-Ready: Disociación de almacenamiento (Tier 1/Tier 2) e inyección PCIe (Passthrough) de GPU NVIDIA RTX 3060 Ti para cargas de inferencia IA/Pentesting.
# Arquitectura Ampere-Ready: GPU Passthrough & Storage Disaggregation en Proxmox VE

**Rol:** Cloud Infrastructure Engineer / SysAdmin
**Tecnologías:** Proxmox VE 9, KVM/QEMU, NVIDIA Ampere (RTX 3060 Ti), IOMMU, SPICE Protocol, UEFI/OVMF.

## 🎯 Objetivo Arquitectónico
Desplegar una máquina virtual en Proxmox VE optimizada a nivel de hardware para cómputo acelerado (PCIe Passthrough de una NVIDIA RTX 3060 Ti destinada a inferencia de IA y pentesting), saneando el almacenamiento del hipervisor y estableciendo un entorno de gestión remota de alta resolución con portapapeles bidireccional.

## 🛠️ Especificación Técnica del Cómputo (Passthrough)

La inyección directa de hardware moderno (NVIDIA Arquitectura Ampere) en máquinas virtuales requiere abandonar las configuraciones heredadas (Legacy) que Proxmox ofrece por defecto. Se aplicó la siguiente matriz de configuración estricta:

* **Machine (q35):** Emulación del chipset Intel ICH9. Obligatorio; el antiguo `i440fx` carece del soporte nativo de topología PCIe necesario para mapear GPUs modernas.
* **BIOS (OVMF / UEFI):** Requisito ineludible. Habilita el direccionamiento de memoria ampliado (64-bit BARs), sin el cual el driver del kernel `nvidia-peermem` falla con errores de comunicación irrecuperables (Error 43 / Fallo de inicialización).
* **Memory (4096 MiB Fija):** Se desactivó el *Memory Ballooning*. Las asignaciones dinámicas de RAM colisionan con el bloqueo de memoria física requerido por el PCI Passthrough.
* **Display (SPICE / qxl):** Al inyectar la GPU física, se prescindió del protocolo noVNC estándar en favor de SPICE, permitiendo redimensionado dinámico, portapapeles bidireccional y delegación gráfica.

## 🗄️ Disociación de Almacenamiento (Tier 1 vs Tier 2)

Para evitar que los diccionarios masivos (ej. `RockYou2024`) o los datasets de IA colapsasen el hipervisor, se implementó una estrategia de almacenamiento desagregado (Disaggregated Storage):

1. **Tier 1 (Transaccional - NVMe/SSD):** Aprovisionamiento exclusivo de `local-lvm` para el sistema operativo base y ejecutables.
2. **Tier 2 (Volumétrico - HDD):** Almacenamiento mecánico montado en Proxmox como recurso de Directorio (`dir`).
3. **Bind/Network Mount:** Los *datasets* pesados se exponen a la máquina virtual invitada a través de un recurso compartido (SMB/NFS), protegiendo la vida útil del almacenamiento Tier 1 de operaciones de I/O masivas.

## 🚧 Lecciones de Integración (Troubleshooting de QEMU)

Durante el despliegue, se resolvieron dos incidentes críticos a nivel de sistema invitado (Linux):

1. **Discordancia del Layout de Teclado (X11 vs TTY):** Se documentó un comportamiento anómalo donde el *Display Manager* y la consola TTY divergían en el procesamiento de secuencias `AltGr`. Se resolvió forzando el reseteo del mapa de símbolos gráfico y de consola de forma unificada:
   > `sudo dpkg-reconfigure keyboard-configuration`
   > `sudo localectl set-x11-keymap es`
2. **Ciclo de Vida de Servicios Paravirtualizados:** Se corrigió el error conceptual de intentar habilitar el *QEMU Guest Agent* mediante unidades `[Install]` en systemd. Los demonios de canal VirtIO reaccionan ante eventos del kernel (detección por `udev`), requiriendo únicamente inicialización en caliente.

## 🛡️ SecOps
**Topología de Cómputo Compartido:** Mantener la GPU aislada del hipervisor gráfico primario permite conmutar el PCI Device dinámicamente entre distintas VMs (ej. Servidor de inferencia LLM en reposo vs. Nodo Hashcat bajo demanda) sin necesidad de reiniciar el host físico físico, maximizando el ROI del hardware.

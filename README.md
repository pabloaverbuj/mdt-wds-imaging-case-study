# Zero-Touch Windows 11 Imaging — MDT + WDS (Cloud-Only, Sin AD)

> Case study de un pipeline de masterización de Windows 11 Pro con **Microsoft Deployment Toolkit + WDS/PXE** para una organización **100% cloud (Entra ID + Intune), sin Active Directory on-premises**.

![status](https://img.shields.io/badge/status-producci%C3%B3n-green)
![OS](https://img.shields.io/badge/target-Windows%2011%20Pro-0078D6)
![MDT](https://img.shields.io/badge/MDT-8456-5C2D91)
![identity](https://img.shields.io/badge/identity-Entra%20ID%20only-0078D6)

> [!NOTE]
> Repositorio de muestra. Nombres de host, IPs y la topología son **ilustrativos**; la infraestructura real está anonimizada.

## El problema

Masterizar equipos Windows 11 nuevos a escala en una organización **sin domain controllers**. El clásico MDT asume Active Directory (domain join, cuentas de servicio de dominio, OUs). Acá la identidad es **cloud-only**: hay que desacoplar el imaging del dominio y resolver la identidad *después*, con Entra ID Join + enrolamiento en Intune.

## Solución — arquitectura

```
LAN de clientes 10.0.0.0/24          Segmento de servidores (aislado)
        │                                     │
   [ cliente PXE ] ──── MikroTik ─────────────┤  (interface dedicada)
                        regla de acceso         │
                        inter-segmento          └── DEPLOY-SRV
                                                    ├─ MDT 8456 + Windows ADK
                                                    ├─ WDS (PXE, RemoteInstall)
                                                    └─ DHCP con opciones PXE
                                                       next-server → DEPLOY-SRV
                                                       opt 66/67/60
```

El servidor de deployment vive en un **segmento aislado**. El cliente arranca por red (PXE), WDS entrega el WinPE de MDT, un wizard selecciona la Task Sequence y se instala el SO.

## Componentes

| Componente | Rol |
|------------|-----|
| MDT 8456 | Orquestación de Task Sequences |
| Windows ADK | WinPE, DISM, herramientas de deploy |
| WDS | Servidor PXE (standalone, responde a todos los clientes) |
| DHCP | Entrega IP + opciones PXE (66/67/60) en el segmento |
| MikroTik (router de borde) | Routing + regla de acceso al segmento de servidores |

## El desafío interesante: dual-WAN rompía el PXE

El router corría **policy-routing dual-WAN con PCC** (balanceo de carga entre dos proveedores). Efecto colateral: el tráfico **entre segmentos internos** quedaba marcado por las reglas PCC y se enrutaba **hacia el WAN** en vez de quedarse en la LAN → se fugaba a internet y el cliente nunca alcanzaba al server de deployment.

**Diagnóstico:** las reglas mangle PCC usaban `dst-address-type=!local`, que no distingue "otra red interna" de "internet".

**Fix:** una regla `accept` en `prerouting` que corta el tráfico inter-segmento **antes** de que lo evalúen las reglas PCC:

```routeros
/ip firewall mangle
add chain=prerouting action=accept \
    src-address=<LAN> dst-address=<segmento-servidores> \
    place-before=0 comment="Trafico interno: no balancear entre WANs"
```

Lección: el orden de las reglas mangle es todo. Diseño correcto — el tráfico interno nunca debería balancearse entre WANs.

## Pipeline de masterización (6 etapas)

> [!IMPORTANT]
> Sin Active Directory: **no** hay `JoinDomain`, `DomainAdmin` ni `MachineObjectOU`. La cuenta de servicio de MDT es una cuenta **local** del Windows Server.

1. **Imaging** — MDT + WDS + PXE instala Windows 11 Pro.
2. **PPKG** — Provisioning package (Windows Configuration Designer) con bulk enrollment token → **Entra ID Join** automático en OOBE.
3. **MDM Enrollment** — Enrola en Intune vía cuenta admin del tenant, **sin** consumir licencia Intune del usuario final.
4. **Sync de políticas** — BitLocker (clave en Entra ID), Windows Hello for Business, Compliance, Update Rings, Defender.
5. **Apps** — IntuneWin32 vía Intune + PSADT.
6. **Entrega** al usuario final.

## Media offline para testing en VM

Para validar sin tocar la red de producción se genera una **Media MDT autónoma**: una ISO self-contained (Win11 Pro + Task Sequence embebidos) que bootea en una VM Gen2 (UEFI + Secure Boot + TPM) **sin conexión de red** y se auto-configura.

> [!WARNING]
> La Media usa **sus propias** reglas (`Content\Deploy\Control\CustomSettings.ini` y `Bootstrap.ini`), no las del deployment share — sin credenciales de red ni `DeployRoot`, porque se auto-localiza. `Update-MDTMedia` las preserva si ya existen.

## Lección aprendida: `TaskSequences.xml` no se edita a mano

El provider de MDT parsea el índice `TaskSequences.xml` sin tirar error, pero **no enumera** entradas con el mínimo defecto de formato. Procedimiento confiable para registrar una TS por script:

- [x] Vaciar el índice a `<tss></tss>`
- [x] `New-PSDrive` con `MDTProvider` (Windows PowerShell 5.1 + snap-in `Microsoft.BDD.PSSnapIn`)
- [x] `Import-MDTTaskSequence`
- [x] `Remove-PSDrive` para **flushear a disco** (sin esto el XML no persiste)
- [x] Verificar en un proceso nuevo

<details>
<summary>Administración remota con WinRM cerrado</summary>

Cuando WinRM (5985) y DCOM (135) están cerrados en el firewall del server, el canal confiable es **PsExec** corriendo como SYSTEM (`-s`), que reutiliza la sesión SMB ya autenticada en vez de pedir un logon interactivo. La lógica va en un `.ps1` ejecutado por ruta y la salida se lee por SMB.

</details>

## Stack / Skills demostradas

`MDT 8456` · `Windows ADK / WinPE / DISM` · `WDS + PXE` · `DHCP (opt 66/67/60)` · `MikroTik RouterOS (mangle, PCC, policy-routing)` · `Entra ID Join` · `Intune / MDM` · `PowerShell` · `Windows Configuration Designer (PPKG)` · `BitLocker` · `Windows Hello for Business`

## Roadmap

- [x] PXE del deployment share funcionando end-to-end
- [x] Media offline autónoma para VM
- [x] Resuelto el routing inter-segmento con dual-WAN
- [ ] Automatizar PPKG + MDM enrollment dentro de la Task Sequence
- [ ] Auto-incremento secuencial del nombre de equipo
- [ ] Inyección de drivers por modelo

## Licencia

MIT

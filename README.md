<!--

Propósito: Guía “bonita” para montar un entorno iDempiere 8.2 con direnv + .envrc
-->

# iDempiere 8.2 — Entorno aislado con direnv + `.envrc` 

> Un entorno por carpeta: Maven, repo Maven, workspace Eclipse y p2 **aislados** dentro del proyecto.

---

## Vista rápida (lo esencial)

| Objetivo | Comando |
|---|---|
| Instalar direnv | `sudo apt install direnv` |
| Activar en Bash | `eval "$(direnv hook bash)"` (en `~/.bashrc`) |
| Autorizar el `.envrc` | `direnv allow` |
| Probar Maven del entorno | `mvn -v` |
| Abrir Eclipse del entorno | `eclipse-here` |

---

## 1) Requisitos

- Java 11 instalado en el sistema.
- Carpeta raíz del entorno (ejemplo: `~/Documents/idempiere82/`).
- Archivos para copiar dentro del entorno:
  - Apache Maven 3.6.3 (se copiará dentro de `apache-maven-3.6.3/`).
  - Eclipse (se copiará dentro de `eclipse/`).
  - Repo de iDempiere 8.2 (se clonará/pondrá dentro de `sources/idempiere/`).

---

## 2) Instalar direnv + hook del shell

### Ubuntu/Debian (instalación)
```bash
sudo apt update
sudo apt install direnv
Activar direnv en Bash
Edita ~/.bashrc y agrega al final:

bash
eval "$(direnv hook bash)"
Luego recarga:

bash
source ~/.bashrc
Activar direnv en Zsh (opcional)
Edita ~/.zshrc y agrega:

text
eval "$(direnv hook zsh)"
3) Estructura esperada de carpetas
Crea la carpeta raíz y coloca dentro el .envrc. El script se encarga de crear varias carpetas automáticamente, pero tú debes copiar Maven/Eclipse y clonar iDempiere.

text
idempiere82/
├─ .envrc
├─ .local-bin/                  (auto) wrappers: mvn, eclipse-here, eclipse-choose
├─ .m2/                         (auto) repo Maven aislado
│  ├─ repository/
│  └─ settings.xml
├─ .mvn/                        (auto)
│  └─ maven.config
├─ .p2/                         (auto) p2 aislado para Eclipse
├─ workspace-8/                 (auto) workspace aislado
├─ apache-maven-3.6.3/          (manual) aquí va Maven real (debe existir bin/mvn)
├─ eclipse/                     (manual) aquí va Eclipse real (debe existir eclipse)
└─ sources/
   └─ idempiere/                (manual) repo/clon de iDempiere 8.2
4) Qué debes copiar (manual)
Maven 3.6.3
Tu carpeta debe terminar así:

text
apache-maven-3.6.3/
└─ bin/
   └─ mvn
Eclipse
Tu carpeta debe terminar así:

text
eclipse/
└─ eclipse
iDempiere 8.2
Debe quedar así:

text
sources/idempiere/
  (contenido del repo)
Ejemplo (ajusta URL/branch según tu repo):

bash
cd ~/Documents/idempiere82/sources
git clone <URL_DEL_REPO_IDEMPIERE_8_2> idempiere
5) Pegar el .envrc y habilitarlo
Entra a la carpeta raíz del entorno:

bash
cd ~/Documents/idempiere82
Crea/pega tu .envrc dentro de esa carpeta.

Autoriza la carga:

bash
direnv allow
Si editas el .envrc, normalmente tendrás que ejecutar direnv allow otra vez.

6) Pruebas rápidas (sanity checks)
Ver variables clave
bash
echo "$IDEM8_ROOT"
echo "$JAVA_HOME"
echo "$IDEMPIERE_HOME"
echo "$IDEMPIERE_REPO"
Verificar Maven del entorno
bash
mvn -v
Lanzar Eclipse del entorno
bash
eclipse-here
7) Errores típicos y solución corta
Caso A: “.envrc is blocked”
Ejecuta:

bash
direnv allow
Caso B: “Maven no encontrado”
Revisa que exista:

text
apache-maven-3.6.3/bin/mvn
Caso C: “Eclipse no encontrado”
Revisa que exista:

text
eclipse/eclipse
Caso D: Java 11 no detectado
Instala Java 11 o ajusta el bloque de autodetección en el .envrc.

8) Checklist final
 Direnv instalado.

 Hook activado en tu shell (~/.bashrc o ~/.zshrc).

 .envrc pegado en la raíz del entorno.

 direnv allow ejecutado.

 Maven copiado en apache-maven-3.6.3/ (con bin/mvn).

 Eclipse copiado en eclipse/ (con ejecutable eclipse).

 iDempiere 8.2 en sources/idempiere/.

Orden recomendado (cero fricción)
Crear carpeta raíz.

Copiar Maven y Eclipse dentro.

Clonar iDempiere dentro de sources/.

Pegar .envrc.

direnv allow.

Probar mvn -v y eclipse-here.


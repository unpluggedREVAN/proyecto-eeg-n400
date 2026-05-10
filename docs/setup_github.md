# Guía: cómo subir este repo a GitHub

> Esta guía es **una sola vez**. Después solo es `git pull` / `git push` cuando agreguemos notebooks.

## Opción 1 — Lo más simple (recomendado)

### Paso 1 — Crear el repo en GitHub

1. Ir a https://github.com/new
2. Repository name: `proyecto-eeg-n400`
3. Visibilidad: **Private** (es trabajo de curso) o Public si su profe pide acceso público.
4. **NO marcar** "Initialize with README" / "Add .gitignore" / "Add license" — ya los tenemos.
5. Click "Create repository".

GitHub mostrará una pantalla con comandos. Ignorar esa pantalla y seguir aquí.

### Paso 2 — Subir el contenido desde su computadora

Descomprimir el ZIP que les pasé. Abrir una terminal **dentro de la carpeta `proyecto-eeg-n400/`** y correr:

```bash
git init
git add .
git commit -m "Initial commit — esqueleto del repo + Notebook 00"
git branch -M main
git remote add origin https://github.com/<TU_USUARIO>/proyecto-eeg-n400.git
git push -u origin main
```

Reemplazar `<TU_USUARIO>` por su usuario de GitHub. Si pide autenticación, usar un **Personal Access Token** (no la contraseña):
- https://github.com/settings/tokens → "Generate new token (classic)"
- Permisos: solo `repo`
- Copiar el token y pegarlo como contraseña cuando git lo pida.

Listo. El repo ya está en GitHub.

### Paso 3 — Agregar a su compañero como colaborador

1. En GitHub, ir al repo → Settings → Collaborators → Add people.
2. Buscar el usuario del compañero y mandar invitación.

### Paso 4 — Clonar el repo dentro de Google Drive

Esto es para que los notebooks de Colab puedan leerlo. Hay dos formas:

**Forma A — desde su computadora (más simple):**
1. En su computadora, ir a `Google Drive` carpeta sincronizada o usar el navegador.
2. Crear la carpeta `MyDrive/IA_Proyecto_EEG/`.
3. Subir la carpeta `proyecto-eeg-n400/` completa adentro.

**Forma B — desde una celda de Colab (más limpio, opcional):**
```python
from google.colab import drive
drive.mount('/content/drive')

%cd /content/drive/MyDrive/IA_Proyecto_EEG/
!git clone https://github.com/<TU_USUARIO>/proyecto-eeg-n400.git
```

### Paso 5 — Descargar el dataset y colocarlo

El dataset (Hayes & Magne 2025) viene aparte. Colocarlo en:

```
MyDrive/IA_Proyecto_EEG/dataset/
├── Behavioral Data/
├── EEG Data/
├── Experimental Task/
└── Scripts/
```

### Estructura final esperada en Drive

```
MyDrive/IA_Proyecto_EEG/
├── dataset/                ← datos del paper
├── proyecto-eeg-n400/      ← este repo
│   ├── configs/
│   ├── notebooks/
│   ├── docs/
│   └── ... 
├── reports/                ← se crean al correr los notebooks
├── figures/
├── features/
├── models/
└── logs/
```

---

## Cómo trabajar de aquí en adelante

### Cuando Claude entregue un notebook nuevo:

1. Descargar el archivo `.ipynb` que Claude les pasa.
2. Ponerlo en `proyecto-eeg-n400/notebooks/` localmente.
3. Subirlo a GitHub:
   ```bash
   git add notebooks/XX_nombre.ipynb
   git commit -m "Agrega Notebook XX — descripción breve"
   git push
   ```
4. En Drive, hacer `git pull` desde una celda de Colab para que se sincronice:
   ```python
   %cd /content/drive/MyDrive/IA_Proyecto_EEG/proyecto-eeg-n400
   !git pull
   ```

### Cuando ejecuten un notebook y se generen outputs:

Los outputs (CSVs, figuras, modelos) **NO** van al repo — están en `.gitignore`. Esto es a propósito:
- Pesan mucho.
- Se regeneran al correr el notebook.
- Solo el código y la metodología viven en GitHub.

Lo único que sí se sube al repo después de ejecutar un notebook es:
- El propio `.ipynb` con las salidas embebidas (figuras y prints quedan dentro del notebook).
- Actualizaciones a `docs/methodology.md` si cambiaron alguna decisión.

---

## Opción 2 — Trabajando enteramente desde Colab (alternativa)

Si prefieren no usar terminal:

1. En GitHub crear el repo vacío como en Paso 1.
2. En Colab, ejecutar:
   ```python
   from google.colab import drive
   drive.mount('/content/drive')
   
   %cd /content/drive/MyDrive/IA_Proyecto_EEG
   
   # Subir el ZIP a Drive y descomprimirlo aquí
   !unzip -q proyecto-eeg-n400.zip
   %cd proyecto-eeg-n400
   
   !git init
   !git add .
   !git -c user.email="tu@email.com" -c user.name="Tu Nombre" commit -m "Initial commit"
   !git branch -M main
   !git remote add origin https://<TU_TOKEN>@github.com/<TU_USUARIO>/proyecto-eeg-n400.git
   !git push -u origin main
   ```

---

## Verificación rápida

Después de subir, abrir https://github.com/<TU_USUARIO>/proyecto-eeg-n400 y verificar que se vean:

- README.md (debería renderizarse en la página principal)
- configs/main_config.yaml
- notebooks/00_setup_and_verify.ipynb
- docs/methodology.md
- requirements.txt, .gitignore, LICENSE

Si todo está ahí: listo, podemos arrancar.

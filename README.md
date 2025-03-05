# **Ejercicio: Despliegue Automatizado de una Aplicación Web con Ansible**

## **1. Descripción del Ejercicio**
En este ejercicio, deberan **automatizar la instalación y configuración de una aplicación web distribuida** utilizando Ansible. La aplicación consiste en una **web Flask con una base de datos PostgreSQL**, desplegada en un entorno de múltiples servidores.

Para completar el ejercicio, deberán:\
✅ Configurar un servidor web con **Flask** y **Gunicorn**.  
✅ Configurar un servidor de base de datos con **PostgreSQL**.  
✅ Usar **roles de Ansible** para organizar la configuración.  
✅ Asegurar que la aplicación es accesible vía navegador.  



## **2. Entorno de Trabajo**
El laboratorio se compone de **tres máquinas virtuales**:

| Máquina            | Función           | Sistema Operativo | IP Estática          |
|--------------------|------------------|------------------|----------------------|
| **awx-control-node** | Servidor de Ansible (Bastión) | Ubuntu Server 22.04 | `192.168.*.101` |
| **debian-node1**   | Servidor de Base de Datos | Debian 11 | `192.168.*.102` |
| **ubuntu-node2**   | Servidor Web (Flask) | Ubuntu 22.04 | `192.168.*.103` |

 **Acceso a las máquinas:**
- Usuario: `ansible`  
- Contraseña: `credicoop`  
- Acceso SSH con llaves preconfigurado.  

## **3. Estructura de Carpetas en Ansible**
El código de Ansible debe organizarse con la siguiente estructura:

```
ansible-lab/
├── inventory.ini
├── ansible.cfg
├── site.yml
├── roles/
│   ├── webapp/        # Configuración del Servidor Web
│   │   ├── tasks/
│   │   ├── templates/
│   │   ├── handlers/
│   │   ├── vars/
│   ├── postgresql/    # Configuración del Servidor de Base de Datos
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── vars/
```
**Reglas obligatorias:**
- **No escribir lógica en `site.yml`**. Todo debe estar en roles.
- **Utilizar variables para evitar valores hardcodeados**.


## **4. Objetivo**
El objetivo es **automatizar el despliegue** de una aplicación Flask con PostgreSQL utilizando Ansible.


## **5. Tareas a Realizar**

### **Configuración de PostgreSQL**
Configura el servidor `debian-node1` para que aloje la base de datos. Debe:\
✅ Instalar **PostgreSQL 13** y sus dependencias.  
✅ Permitir conexiones remotas.  
✅ Crear la base de datos `flaskdb`.  
✅ Crear el usuario `ansible` con contraseña `credicoop`.  
✅ Asignar permisos sobre `flaskdb` al usuario `ansible`.  

📌 **Variables Sugeridas (`roles/postgresql/vars/main.yml`)**
```yaml
db_host: debian-node1
db_name: flaskdb
db_user: ansible
db_password: credicoop

postgresql_version: "13"
postgresql_packages:
  - postgresql
  - postgresql-contrib
postgresql_dependencies:
  - python3-psycopg2
  - python3-apt
```


### **Configuración del Servidor Web**
Configura `ubuntu-node2` para servir la aplicación Flask. Debe:\
✅ Instalar Flask y dependencias en un **entorno virtual**.  
✅ Configurar Gunicorn como servicio.  
✅ Servir una aplicación Flask que cuente las visitas a la base de datos.  
✅ Asegurar que la aplicación es accesible en `http://ubuntu-node2:5000`.  

**Variables Sugeridas (`roles/webapp/vars/main.yml`)**
```yaml
webapp_user: flask
webapp_group: www-data
webapp_directory: /opt/webapp
webapp_port: 5000
gunicorn_workers: 3
webapp_packages:
  - python3
  - python3-pip
  - python3-venv
  - nginx
  - libpq-dev
```

**Dependencias (`roles/webapp/files/requirements.txt`)**
```
flask
psycopg2-binary
gunicorn
```


### **Configurar Flask y Gunicorn**
El código de la aplicación Flask se encuentra en un template que deberán copiar a `app.py` en el servidor web.

**Template Flask (`roles/webapp/templates/app.py.j2`)**
```python
from flask import Flask, render_template
import psycopg2

app = Flask(__name__, template_folder="/opt/webapp/templates", static_folder="/opt/webapp/static")

def get_db_connection():
    conn = psycopg2.connect(
        dbname="{{ db_name }}",
        user="{{ db_user }}",
        password="{{ db_password }}",
        host="{{ db_host }}"
    )
    return conn

@app.route('/')
def home():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('CREATE TABLE IF NOT EXISTS visits (id SERIAL PRIMARY KEY, timestamp TIMESTAMP DEFAULT now());')
    conn.commit()
    cur.execute('INSERT INTO visits DEFAULT VALUES;')
    conn.commit()
    cur.execute('SELECT COUNT(*) FROM visits;')
    count = cur.fetchone()[0]
    cur.close()
    conn.close()
    return render_template('index.html', visits=count)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port={{ webapp_port }})
```

---

### **Interfaz Gráfica**
Debajo se encuentran los templates para el index y el css, deberás copiarlos a los directorios correspondientes en el servidor web.

📌 **Template HTML (`roles/webapp/templates/index.html.j2`)**
```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>App web distribuída via Ansible</title>
    <link rel="stylesheet" href="{% raw %}{{ url_for('static', filename='style.css') }}{% endraw %}">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">
</head>
<body>
    <div class="container text-center">
        <h1 class="mt-5">App web distribuída via Ansible</h1>
        <p class="lead">Esta página ha sido visitada <strong>{% raw %}{{ visits }}{% endraw %}</strong> veces.</p>
        <button class="btn btn-primary mt-3" onclick="window.location.reload();">Recargar</button>
    </div>
</body>
</html>
```

**Estilos (`roles/webapp/templates/style.css.j2`)**
```css
body {
    background-color: #f8f9fa;
    font-family: 'Arial', sans-serif;
}

h1 {
    color: #007bff;
    margin-top: 50px;
}

.lead {
    font-size: 1.5rem;
    color: #333;
}

button {
    padding: 10px 20px;
    font-size: 1.2rem;
    background-color: #007bff;
    color: white;
    border-radius: 5px;
    cursor: pointer;
}
```

---

## **6. Validación**
Una vez desplegado el entorno, prueben:
```bash
curl http://ubuntu-node2:5000
```
Debe mostrar:
```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Flask App</title>
    <link rel="stylesheet" href="/static/style.css">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">
</head>
<body>
    <div class="container text-center">
        <h1 class="mt-5">App web distribuída via Ansible</h1>
        <p class="lead">Esta página ha sido visitada <strong>10</strong> veces.</p>
        
        <!-- Botón para recargar la página -->
        <button class="btn btn-primary mt-3" onclick="window.location.reload();">Recargar</button>
    </div>
</body>
</html>
```
O pueden acceder desde un navegador a la url http://ubuntu-node2:5000 / http://[ip-ubuntu-node2]:5000


## **7. Documentación Relevante**
### 🔹 **Ansible**
- **Conceptos básicos de Ansible** → [https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html](https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html)
- **Organización de roles en Ansible** → [https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)
- **Uso de variables en Ansible** → [https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html)
- **Handlers en Ansible** → [https://docs.ansible.com/ansible/latest/user_guide/playbooks_handlers.html](https://docs.ansible.com/ansible/latest/user_guide/playbooks_handlers.html)
- **Templates con Jinja2 en Ansible** → [https://docs.ansible.com/ansible/latest/user_guide/playbooks_templating.html](https://docs.ansible.com/ansible/latest/user_guide/playbooks_templating.html)

---

### 🔹 **PostgreSQL**
- **Instalación y configuración de PostgreSQL** → [https://www.postgresql.org/docs/13/tutorial-install.html](https://www.postgresql.org/docs/13/tutorial-install.html)
- **Autenticación y conexión remota (`pg_hba.conf`)** → [https://www.postgresql.org/docs/13/auth-pg-hba-conf.html](https://www.postgresql.org/docs/13/auth-pg-hba-conf.html)
- **Creación de bases de datos y usuarios** → [https://www.postgresql.org/docs/13/sql-createdatabase.html](https://www.postgresql.org/docs/13/sql-createdatabase.html)

---

### 🔹 **Flask**
- **Introducción a Flask** → [https://flask.palletsprojects.com/en/3.0.x/quickstart/](https://flask.palletsprojects.com/en/3.0.x/quickstart/)
- **Uso de plantillas en Flask (Jinja2)** → [https://flask.palletsprojects.com/en/3.0.x/templates/](https://flask.palletsprojects.com/en/3.0.x/templates/)
- **Conexión de Flask con PostgreSQL usando `psycopg2`** → [https://www.psycopg.org/docs/usage.html](https://www.psycopg.org/docs/usage.html)
- **Deploy de Flask con Gunicorn** → [https://flask.palletsprojects.com/en/3.0.x/deploying/gunicorn/](https://flask.palletsprojects.com/en/3.0.x/deploying/gunicorn/)

---

### 🔹 **Gunicorn y Systemd**
- **Cómo usar Gunicorn para servir aplicaciones Flask** → [https://docs.gunicorn.org/en/stable/run.html](https://docs.gunicorn.org/en/stable/run.html)
- **Configuración de Gunicorn con Systemd** → [https://docs.gunicorn.org/en/stable/deploy.html#systemd](https://docs.gunicorn.org/en/stable/deploy.html#systemd)

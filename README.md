# Laboratorio 06: Aplicación Web con Flask, Blueprint, Flask-WTF, WTForms y SQLAlchemy ORM

**Duración:** 100 minutos

Este tutorial te guiará paso a paso en la creación de una aplicación web sencilla utilizando Flask, Flask-WTF, WTForms y SQLAlchemy ORM versión 2. Crearemos dos entidades: `Lugar` y `Persona`, y cubriremos operaciones CRUD (crear, leer, actualizar y eliminar) con funcionalidades de búsqueda y validación de formularios.

## Estructura del Proyecto

```bash
flask_app/
│   config.py
│   requirements.txt
│   run.py
│
└───app
    │   __init__.py
    │
    ├───lugar
    │       forms.py
    │       models.py
    │       routes.py
    │       __init__.py
    │
    ├───persona
    │       forms.py
    │       models.py
    │       routes.py
    │       __init__.py
    │
    ├───static
    └───templates
        │   base.html
        │   index.html
        │
        ├───lugar
        │       form.html
        │       index.html
        │
        └───persona
                form.html
                index.html
```

## 1. Configuración Inicial

### 1.1. Crea un entorno virtual y activa el entorno:

```bash
python3 -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate  # Windows
```

### 1.2. Instala las dependencias necesarias:

```bash
pip install email_validator Flask Flask-Migrate Flask-SQLAlchemy Flask-WTF SQLAlchemy WTForms
```

Crea el archivo `requirements.txt` para facilitar la instalación en el futuro:

```bash
pip freeze > requirements.txt
```

### 1.3. Crea el archivo `config.py`:

```python
import os
basedir = os.path.abspath(os.path.dirname(__file__))

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'you-will-never-guess'
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
        'sqlite:///' + os.path.join(basedir, 'app.db')
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

## 2. Inicia la Aplicación

### 2.1. Crea `app/__init__.py`:
```python
from flask_migrate import Migrate
from flask import Flask, render_template
from config import Config
from flask_sqlalchemy import SQLAlchemy
from flask_wtf.csrf import CSRFProtect

# Inicialización de las extensiones
db = SQLAlchemy()
migrate = Migrate()
csrf = CSRFProtect()

def create_app():
    app = Flask(__name__)
    
    # Configuración de la aplicación desde el archivo config.py
    app.config.from_object(Config)
    
    # Inicializar las extensiones
    db.init_app(app)
    migrate.init_app(app, db)
    csrf.init_app(app)
    
    # Registrar los blueprints
    from app.lugar import bp as lugar_bp
    app.register_blueprint(lugar_bp, url_prefix='/lugares')

    from app.persona import bp as persona_bp
    app.register_blueprint(persona_bp, url_prefix='/personas')

    
    @app.route('/')
    def home():
        return render_template('index.html')

    return app
```

### 2.2. Crea `run.py`:

```python
from .app import create_app

app = create_app()

if __name__ == '__main__':
    # app = create_app()
    app.run()
```

## 3. Modelos de Datos

### 3.1. Modelo para `Lugar`

Crea `app/lugar/models.py`:

```python
from app import db
from sqlalchemy import (BLOB, Table, Column, String, Boolean, Integer, DateTime, ForeignKey, Text, UniqueConstraint,
                        Index, Date, Numeric, Double, func, CheckConstraint)
from sqlalchemy.orm import relationship, Mapped, mapped_column
from datetime import datetime, date
from typing import Optional

class Lugar(db.Model):
    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(64))
    codigo_postal: Mapped[str] = mapped_column(String(10))
    personas: Mapped[list['Persona']] = relationship(back_populates='lugar')

    def __repr__(self):
        return f'<Lugar {self.nombre}>'
  
```

### 3.2. Modelo para `Persona`

Crea `app/persona/models.py`:

```python
from app import db
from sqlalchemy import (BLOB, Table, Column, String, Boolean, Integer, DateTime, ForeignKey, Text, UniqueConstraint,
                        Index, Date, Numeric, Double, func, CheckConstraint)
from sqlalchemy.orm import relationship, Mapped, mapped_column
from datetime import datetime, date
from typing import Optional

class Persona(db.Model):
    id: Mapped[int] = mapped_column(primary_key=True)
    numero_identidad: Mapped[str] = mapped_column(String(20))
    nombre: Mapped[str] = mapped_column(String(64))
    apellidos: Mapped[str] = mapped_column(String(64))
    fecha_nacimiento = db.Column(db.Date, nullable=False)
    correo: Mapped[str] = mapped_column(String(120))
    cantidad_mascotas = db.Column(db.Integer, nullable=False)
    semana_inicio: Mapped[str] = mapped_column(String(10))
    lugar_id: Mapped[int] = mapped_column(Integer, ForeignKey('lugar.id'))
    lugar: Mapped['Lugar'] = relationship(back_populates='personas')



    def __repr__(self):
        return f'<Persona {self.nombre} {self.apellidos}>'
```

## 4. Formularios con Flask-WTF y WTForms

### 4.1. Formularios para `Lugar`

Crea `app/lugar/forms.py`:

```python
from flask_wtf import FlaskForm
from wtforms import StringField, SubmitField
from wtforms.validators import DataRequired

class LugarForm(FlaskForm):
    nombre = StringField('Nombre', validators=[DataRequired()])
    codigo_postal = StringField('Código Postal', validators=[DataRequired()])
    submit = SubmitField('Guardar')
```

### 4.2. Formularios para `Persona`

Crea `app/persona/forms.py`:

```python
from flask_wtf import FlaskForm
from wtforms import StringField, IntegerField, DateField, SelectField, SubmitField
from wtforms.validators import DataRequired, Email, NumberRange
from app.lugar.models import Lugar

class PersonaForm(FlaskForm):
    numero_identidad = StringField('Número de Identidad', validators=[DataRequired()])
    nombre = StringField('Nombre', validators=[DataRequired(message="Este campo es obligatorio")])
    apellidos = StringField('Apellidos', validators=[DataRequired()])
    fecha_nacimiento = DateField('Fecha de Nacimiento', validators=[DataRequired()])
    correo = StringField('Correo', validators=[DataRequired(), Email(message="Dirección de correo electrónico inválida")])
    cantidad_mascotas = IntegerField('Cantidad de Mascotas', validators=[DataRequired(), NumberRange(min=0)])
    semana_inicio = StringField('Semana de Inicio', validators=[DataRequired()])
    lugar_id = SelectField('Lugar', coerce=int)
    submit = SubmitField('Guardar')

    def __init__(self):
        super().__init__()
        self.lugar_id.choices = [(lugar.id, lugar.nombre) for lugar in Lugar.query.all()]
```

## 5. Rutas y Vistas

### 5.1. Crea el init de `Lugar`

Crea `app/lugar/__init__.py`:

```python
from flask import Blueprint

bp = Blueprint('lugar', __name__)

from app.lugar import routes

```

### 5.2. Rutas para `Lugar`

Crea `app/lugar/routes.py`:

```python
from sqlite3 import IntegrityError
from flask import render_template, redirect, url_for, flash
from app import db
from app.lugar import bp
from app.lugar.forms import LugarForm
from app.lugar.models import Lugar

@bp.route('/')
def index():
    lugares = Lugar.query.all()
    return render_template('lugar/index.html', lugares=lugares)

@bp.route('/crear', methods=['GET', 'POST'])
def crear():
    form = LugarForm()
    if form.validate_on_submit():
        lugar = Lugar(nombre=form.nombre.data, codigo_postal=form.codigo_postal.data)
        db.session.add(lugar)
        db.session.commit()
        flash('Lugar creado con éxito.')
        return redirect(url_for('lugar.index'))
    return render_template('lugar/form.html', form=form)

@bp.route('/editar/<int:id>', methods=['GET', 'POST'])
def editar(id):
    lugar = Lugar.query.get_or_404(id)
    form = LugarForm(obj=lugar)
    if form.validate_on_submit():
        form.populate_obj(lugar)
        db.session.commit()
        flash('Lugar actualizado con éxito.')
        return redirect(url_for('lugar.index'))
    return render_template('lugar/form.html', form=form)

@bp.route('/eliminar/<int:id>', methods=['POST'])
def eliminar(id):
    lugar = Lugar.query.get_or_404(id)
    
    # Verificar si hay personas asociadas a este lugar
    if lugar.personas:
        flash('No puedes eliminar este lugar porque está asociado a una o más personas.', 'error')
        return redirect(url_for('lugar.index'))

    try:
        db.session.delete(lugar)
        db.session.commit()
        flash('Lugar eliminado con éxito.')
    except IntegrityError:
        db.session.rollback()
        flash('Error al eliminar el lugar.', 'error')
    
    return redirect(url_for('lugar.index'))


```

### 5.3. Crea el init de `Persona`

Crea `app/persona/__init__.py`:

```python
from flask import Blueprint

bp = Blueprint('persona', __name__)

from app.persona import routes
```


### 5.4. Rutas para `Persona`
Crea `app/persona/routes.py`:

```python
from sqlite3 import IntegrityError
from flask import render_template, redirect, url_for, flash, request
from app import db
from app.persona import bp
from app.persona.forms import PersonaForm
from app.persona.models import Persona
from app.lugar.models import Lugar

@bp.route('/')
def index():
    personas = Persona.query.all()
    return render_template('persona/index.html', personas=personas)


@bp.route('/crear', methods=['GET', 'POST'])
def crear():
    form = PersonaForm()
    
    if form.validate_on_submit():
        # Verificar si el número de identidad ya existe en la base de datos
        persona_existente = Persona.query.filter_by(numero_identidad=form.numero_identidad.data).first()
        
        if persona_existente:
            flash('El número de identidad ya existe. Por favor ingresa uno diferente.', 'error')
            return render_template('persona/form.html', form=form)
        
        # Si no existe, proceder a crear la nueva persona
        persona = Persona(
            numero_identidad=form.numero_identidad.data,
            nombre=form.nombre.data,
            apellidos=form.apellidos.data,
            fecha_nacimiento=form.fecha_nacimiento.data,
            correo=form.correo.data,
            cantidad_mascotas=form.cantidad_mascotas.data,
            semana_inicio=form.semana_inicio.data,
            lugar_id=form.lugar_id.data
        )
        
        db.session.add(persona)
        db.session.commit()
        flash('Persona creada con éxito.', 'success')
        return redirect(url_for('persona.index'))
    
    return render_template('persona/form.html', form=form)



@bp.route('/editar/<int:id>', methods=['GET', 'POST'])
def editar(id):
    persona = Persona.query.get_or_404(id)
    form = PersonaForm()

    if form.validate_on_submit():
        # Verificar si el número de identidad ya existe en otra persona
        persona_existente = Persona.query.filter_by(numero_identidad=form.numero_identidad.data).filter(Persona.id != id).first()
        
        if persona_existente:
            flash('El número de identidad ya existe. Por favor ingresa uno diferente.', 'error')
            return render_template('persona/form.html', form=form)
        
        # Si el número de identidad no está duplicado, proceder con la actualización
        persona.numero_identidad = form.numero_identidad.data
        persona.nombre = form.nombre.data
        persona.apellidos = form.apellidos.data
        persona.fecha_nacimiento = form.fecha_nacimiento.data
        persona.correo = form.correo.data
        persona.cantidad_mascotas = form.cantidad_mascotas.data
        persona.semana_inicio = form.semana_inicio.data
        persona.lugar_id = form.lugar_id.data
        
        db.session.commit()
        flash('Persona actualizada con éxito.', 'success')
        return redirect(url_for('persona.index'))

    # Pre-popular el formulario con los datos de la persona actual
    form.numero_identidad.data = persona.numero_identidad
    form.nombre.data = persona.nombre
    form.apellidos.data = persona.apellidos
    form.fecha_nacimiento.data = persona.fecha_nacimiento
    form.correo.data = persona.correo
    form.cantidad_mascotas.data = persona.cantidad_mascotas
    form.semana_inicio.data = persona.semana_inicio
    form.lugar_id.data = persona.lugar_id

    return render_template('persona/form.html', form=form)


@bp.route('/eliminar/<int:id>', methods=['GET', 'POST'])
def eliminar(id):
    persona = Persona.query.get_or_404(id)
    form = PersonaForm()
    if request.method == 'POST':
        db.session.delete(persona)
        db.session.commit()
        flash('Persona eliminada con éxito.')
        return redirect(url_for('persona.index'))
    return render_template('persona/eliminar.html', persona=persona, form=form)

```

## 6. Plantillas (Templates) con Bootstrap

### 6.1. `base.html`

Crea `app/templates/base.html`:

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}II0703{% endblock %}</title>
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
        <a class="navbar-brand" href="{{ url_for('home') }}">Flask App</a>
        <div class="collapse navbar-collapse">
            <ul class="navbar-nav mr-auto">
                <li class="

nav-item">
                    <a class="nav-link" href="{{ url_for('lugar.index') }}">Lugares</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="{{ url_for('persona.index') }}">Personas</a>
                </li>
            </ul>
        </div>
    </nav>

    <div class="container mt-4">
        {% with messages = get_flashed_messages() %}
        {% if messages %}
            <div class="alert alert-success">
                {% for message in messages %}
                    <p>{{ message }}</p>
                {% endfor %}
            </div>
        {% endif %}
        {% endwith %}
        {% block content %}{% endblock %}
    </div>

    <script src="https://code.jquery.com/jquery-3.5.1.slim.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@4.5.2/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

### 6.2. `index.html`

Crea `app/templates/index.html`:

```html
{% extends 'base.html' %}

{% block title %}Bienvenido a la Aplicación{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-6 offset-md-3 text-center">
        <h1 class="mt-5">Bienvenido a la Aplicación de Gestión</h1>
        <p class="lead">Seleccione una de las siguientes opciones:</p>
        <div class="mt-4">
            <a href="{{ url_for('lugar.index') }}" class="btn btn-lg btn-primary mb-3">Gestionar Lugares</a>
            <br>
            <a href="{{ url_for('persona.index') }}" class="btn btn-lg btn-success">Gestionar Personas</a>
        </div>
    </div>
</div>
{% endblock %}
```

### 6.3. `index.html` para `Lugar`

Crea `app/templates/lugar/index.html`:

```html
{% extends 'base.html' %}

{% block title %}Lista de Lugares{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-12">
        <h2 class="mt-4">Lugares</h2>
        <a href="{{ url_for('lugar.crear') }}" class="btn btn-primary mb-3">Agregar Lugar</a>
        <table class="table table-striped">
            <thead>
                <tr>
                    <th>Nombre</th>
                    <th>Código Postal</th>
                    <th>Acciones</th>
                </tr>
            </thead>
            <tbody>
                {% for lugar in lugares %}
                <tr>
                    <td>{{ lugar.nombre }}</td>
                    <td>{{ lugar.codigo_postal }}</td>
                    <td>
                        <a href="{{ url_for('lugar.editar', id=lugar.id) }}" class="btn btn-sm btn-warning">Editar</a>
                        <form action="{{ url_for('lugar.eliminar', id=lugar.id) }}" method="POST" style="display:inline;">
                            <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
                            <button type="submit" class="btn btn-sm btn-danger">Eliminar</button>
                        </form>
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
</div>
{% endblock %}
```

### 6.4. `form.html` para `Lugar`

Crea `app/templates/lugar/form.html`:

```html
{% extends 'base.html' %}

{% block title %}Agregar/Editar Lugar{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-6 offset-md-3">
        <h2 class="mt-4">{{ 'Editar' if form.nombre.data else 'Agregar' }} Lugar</h2>
        <form method="POST">
            {{ form.hidden_tag() }}
            <div class="form-group">
                {{ form.nombre.label(class="form-label") }}
                {{ form.nombre(class="form-control") }}
            </div>
            <div class="form-group">
                {{ form.codigo_postal.label(class="form-label") }}
                {{ form.codigo_postal(class="form-control") }}
            </div>
            <div class="form-group">
                {{ form.submit(class="btn btn-primary") }}
            </div>
        </form>
        <a href="{{ url_for('lugar.index') }}" class="btn btn-secondary">Volver</a>
    </div>
</div>
{% endblock %}
```

### 6.5. `index.html` para `Persona`

Crea `app/templates/persona/index.html`:

```html
{% extends 'base.html' %}

{% block title %}Lista de Personas{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-12">
        <h2 class="mt-4">Personas</h2>
        <a href="{{ url_for('persona.crear') }}" class="btn btn-primary mb-3">Agregar Persona</a>
        <table class="table table-striped">
            <thead>
                <tr>
                    <th>Nombre Completo</th>
                    <th>Correo</th>
                    <th>Cantidad de Mascotas</th>
                    <th>Semana de Inicio</th>
                    <th>Lugar</th>
                    <th>Acciones</th>
                </tr>
            </thead>
            <tbody>
                {% for persona in personas %}
                <tr>
                    <td>{{ persona.nombre }} {{ persona.apellidos }}</td>
                    <td>{{ persona.correo }}</td>
                    <td>{{ persona.cantidad_mascotas }}</td>
                    <td>{{ persona.semana_inicio }}</td>
                    <td>{{ persona.lugar.nombre }}</td>
                    <td>
                        <a href="{{ url_for('persona.editar', id=persona.id) }}" class="btn btn-sm btn-warning">Editar</a>
                        <form action="{{ url_for('persona.eliminar', id=persona.id) }}" method="POST" style="display:inline;">
                            <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
                            <button type="submit" class="btn btn-sm btn-danger">Eliminar</button>
                        </form>
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
</div>
{% endblock %}
```

### 6.6. `form.html` para `Persona`

Crea `app/templates/persona/form.html`:

```html
{% extends 'base.html' %}

{% block title %}Agregar/Editar Persona{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-6 offset-md-3">
        <h2 class="mt-4">{{ 'Editar' if form.nombre.data else 'Agregar' }} Persona</h2>
        <form method="POST">
            {{ form.hidden_tag() }}
            <div class="form-group">
                {{ form.numero_identidad.label(class="form-label") }}
                {{ form.numero_identidad(class="form-control") }}
                {% if form.numero_identidad.errors %}
                    <div class="text-danger">
                        {% for error in form.numero_identidad.errors %}
                            <p>{{ error }}</p>
                        {% endfor %}
                    </div>
                {% endif %}
            </div>
            <div class="form-group">
                {{ form.nombre.label(class="form-label") }}
                {{ form.nombre(class="form-control") }}
            </div>
            <div class="form-group">
                {{ form.apellidos.label(class="form-label") }}
                {{ form.apellidos(class="form-control") }}
            </div>
            <div class="form-group">
                {{ form.fecha_nacimiento.label(class="form-label") }}
                {{ form.fecha_nacimiento(class="form-control") }}
            </div>


            <div class="form-group">
                {{ form.correo.label(class="form-label") }}
                {{ form.correo(class="form-control") }}
                {% if form.correo.errors %}
                    <div class="text-danger">
                        {% for error in form.correo.errors %}
                            <p>{{ error }}</p>
                        {% endfor %}
                    </div>
                {% endif %}
            </div>


            <div class="form-group">
                {{ form.cantidad_mascotas.label(class="form-label") }}
                {{ form.cantidad_mascotas(class="form-control") }}
            </div>
            <div class="form-group">
                {{ form.semana_inicio.label(class="form-label") }}
                {{ form.semana_inicio(class="form-control") }}
            </div>
            <div class="form-group">
                {{ form.lugar_id.label(class="form-label") }}
                {{ form.lugar_id(class="form-control") }}
            </div>
            <div class="form-group">
                {{ form.submit(class="btn btn-primary") }}
            </div>
        </form>
        <a href="{{ url_for('persona.index') }}" class="btn btn-secondary">Volver</a>
    </div>
</div>
{% endblock %}
```
## 7. Inicializar la base de datos 

**Este paso se debe ejecutar antes de correr por primera vez el servidor Flask**

Hay que asegurarse  que todo esté configurado correctamente, sigue estos pasos:

### Comandos correctos para migraciones:

Una vez que tengas `Flask-Migrate` instalado y configurado en tu aplicación, el flujo correcto para trabajar con las migraciones es el siguiente:

1. **Inicializa el sistema de migraciones** (si no lo has hecho ya):
   
   ```bash
   flask db init
   ```

2. **Crea una nueva migración** (después de definir los modelos en tu aplicación):

   ```bash
   flask db migrate -m "Descripción de la migración"
   ```

3. **Aplica la migración a la base de datos**:

   ```bash
   flask db upgrade
   ```


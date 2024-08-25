# bhavana-demo
This is my first repository. In that user registration and authencation.
task_manager/
├── app/
│   ├── __init__.py
│   ├── models.py
│   ├── routes.py
│   └── auth.py
├── tests/
│   ├── test_auth.py
│   ├── test_tasks.py
├── Dockerfile
├── docker-compose.yml
├── config.py
└── run.py

**app/__init__.py**
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_jwt_extended import JWTManager

db = SQLAlchemy()

def create_app():
    app = Flask(__name__)
    app.config.from_object('config.Config')

    db.init_app(app)
    JWTManager(app)

    with app.app_context():
        from . import routes, auth
        app.register_blueprint(routes.bp)
        app.register_blueprint(auth.bp)
        db.create_all()

    return app
    
**app/models.py**
from datetime import datetime
from . import db

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password_hash = db.Column(db.String(128), nullable=False)

class Task(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(120), nullable=False)
    description = db.Column(db.Text, nullable=True)
    status = db.Column(db.String(50), nullable=False, default='Todo')
    priority = db.Column(db.String(50), nullable=True)
    due_date = db.Column(db.Date, nullable=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, onupdate=datetime.utcnow)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    user = db.relationship('User', backref=db.backref('tasks', lazy=True))

**app/auth.py**
from flask import Blueprint, request, jsonify
from werkzeug.security import generate_password_hash, check_password_hash
from flask_jwt_extended import create_access_token
from .models import User
from . import db

bp = Blueprint('auth', __name__)

@bp.route('/register', methods=['POST'])
def register():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')

    if not username or not password:
        return jsonify({"message": "Missing username or password"}), 400

    if User.query.filter_by(username=username).first():
        return jsonify({"message": "User already exists"}), 400

    hashed_password = generate_password_hash(password)
    new_user = User(username=username, password_hash=hashed_password)
    db.session.add(new_user)
    db.session.commit()

    return jsonify({"message": "User registered successfully"}), 201

@bp.route('/login', methods=['POST'])
def login():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')

    user = User.query.filter_by(username=username).first()

    if user and check_password_hash(user.password_hash, password):
        token = create_access_token(identity=user.id)
        return jsonify({"token": token}), 200

    return jsonify({"message": "Invalid credentials"}), 401
    
**app/routes.py**
from flask import Blueprint, request, jsonify
from flask_jwt_extended import jwt_required, get_jwt_identity
from .models import Task, User
from . import db

bp = Blueprint('routes', __name__)

@bp.route('/tasks', methods=['POST'])
@jwt_required()
def create_task():
    data = request.get_json()
    user_id = get_jwt_identity()

    new_task = Task(
        title=data['title'],
        description=data.get('description'),
        status=data.get('status', 'Todo'),
        priority=data.get('priority'),
        due_date=data.get('due_date'),
        user_id=user_id
    )
    db.session.add(new_task)
    db.session.commit()

    return jsonify({"message": "Task created successfully"}), 201

@bp.route('/tasks', methods=['GET'])
@jwt_required()
def get_tasks():
    user_id = get_jwt_identity()
    tasks = Task.query.filter_by(user_id=user_id)

    # Filtering and searching
    status = request.args.get('status')
    priority = request.args.get('priority')
    if status:
        tasks = tasks.filter_by(status=status)
    if priority:
        tasks = tasks.filter_by(priority=priority)
    search_query = request.args.get('q')
    if search_query:
        tasks = tasks.filter(Task.title.contains(search_query) | Task.description.contains(search_query))

    return jsonify([task.to_dict() for task in tasks]), 200

@bp.route('/tasks/<int:id>', methods=['PUT'])
@jwt_required()
def update_task(id):
    user_id = get_jwt_identity()
    task = Task.query.filter_by(id=id, user_id=user_id).first()

    if not task:
        return jsonify({"message": "Task not found"}), 404

    data = request.get_json()
    task.title = data.get('title', task.title)
    task.description = data.get('description', task.description)
    task.status = data.get('status', task.status)
    task.priority = data.get('priority', task.priority)
    task.due_date = data.get('due_date', task.due_date)

    db.session.commit()
    return jsonify({"message": "Task updated successfully"}), 200

@bp.route('/tasks/<int:id>', methods=['DELETE'])
@jwt_required()
def delete_task(id):
    user_id = get_jwt_identity()
    task = Task.query.filter_by(id=id, user_id=user_id).first()

    if not task:
        return jsonify({"message": "Task not found"}), 404

    db.session.delete(task)
    db.session.commit()
    return jsonify({"message": "Task deleted successfully"}), 200
    
**config.py**
import os

class Config:
    SECRET_KEY = os.getenv('SECRET_KEY', 'your_secret_key')
    SQLALCHEMY_DATABASE_URI = os.getenv('DATABASE_URI', 'sqlite:///tasks.db')
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    JWT_SECRET_KEY = os.getenv('JWT_SECRET_KEY', 'your_jwt_secret_key')

**run.py**
from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000)

**Dockerfile**
    # Use an official Python runtime as a parent image
FROM python:3.9-slim

# Set the working directory
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Make port 5000 available to the world outside this container
EXPOSE 5000

# Run the application
CMD ["python", "run.py"]

**docker-compose.yml**
version: '3.7'

services:
  db:
    image: postgres:13
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: taskmanager
    volumes:
      - pgdata:/var/lib/postgresql/data

  web:
    build: .
    command: python run.py
    volumes:
      - .:/app
    ports:
      - "5000:5000"
    environment:
      DATABASE_URI: "postgresql://postgres:password@db:5432/taskmanager"
    depends_on:
      - db

volumes:
  pgdata:

**requirements.txt**
Flask==2.1.1
Flask-SQLAlchemy==2.5.1
Flask-JWT-Extended==4.3.1
psycopg2-binary==2.9.3


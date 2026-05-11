# plataforma-bet-
jogue com responsabilidade
# config.py
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-key-change-in-production'
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or 'sqlite:///bets.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    JWT_SECRET_KEY = os.environ.get('JWT_SECRET_KEY') or 'jwt-secret-change-in-production'
    JWT_ACCESS_TOKEN_EXPIRES = 3600  # 1 hora
    REDIS_URL = os.environ.get('REDIS_URL') or 'redis://localhost:6379/0'
    STRIPE_API_KEY = os.environ.get('STRIPE_API_KEY')
    LOG_LEVEL = os.environ.get('LOG_LEVEL', 'INFO')

    # Configurações de segurança
    SECURITY_PASSWORD_SALT = os.environ.get('SECURITY_PASSWORD_SALT') or 'security-salt-change-in-production'
    SECURITY_REMEMBER_COOKIE_DURATION = 3600 * 24 * 30  # 30 dias
    # models.py
from flask_sqlalchemy import SQLAlchemy
from flask_bcrypt import Bcrypt
from datetime import datetime

db = SQLAlchemy()
bcrypt = Bcrypt()

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(120), nullable=False)
    balance = db.Column(db.Float, default=0.0)
    is_active = db.Column(db.Boolean, default=True)
    is_admin = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    def set_password(self, password):
        self.password_hash = bcrypt.generate_password_hash(password).decode('utf-8')
    
    def check_password(self, password):
        return bcrypt.check_password_hash(self.password_hash, password)

class Game(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(200), nullable=False)
    odds = db.Column(db.Float, nullable=False)
    start_time = db.Column(db.DateTime, nullable=False)
    status = db.Column(db.String(50), default='active')  # active, ended, canceled
    category = db.Column(db.String(100), nullable=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

class Bet(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    game_id = db.Column(db.Integer, db.ForeignKey('game.id'), nullable=False)
    amount = db.Column(db.Float, nullable=False)
    odds = db.Column(db.Float, nullable=False)
    status = db.Column(db.String(50), default='pending')  # pending, won, lost, refunded
    payout = db.Column(db.Float, default=0.0)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    user = db.relationship('User', backref=db.backref('bets', lazy=True))
    game = db.relationship('Game', backref=db.backref('bets', lazy=True))

class LoginHistory(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    ip_address = db.Column(db.String(45), nullable=False)
    user_agent = db.Column(db.Text, nullable=True)
    login_time = db.Column(db.DateTime, default=datetime.utcnow)
    success = db.Column(db.Boolean, default=True)
    
    user = db.relationship('User', backref=db.backref('login_history', lazy=True))# routes.py
from flask import Blueprint, request, jsonify
from flask_jwt_extended import jwt_required, get_jwt_identity
from models import User, Game, Bet, LoginHistory
from config import Config

api_bp = Blueprint('api', __name__)

@api_bp.route('/register', methods=['POST'])
def register():
    data = request.json
    user = User.query.filter_by(username=data['username']).first()
    if user:
        return jsonify({"message": "Nome de usuário já existe"}), 400
        
    user = User.query.filter_by(email=data['email']).first()
    if user:
        return jsonify({"message": "Email já cadastrado"}), 400
        
    new_user = User(username=data['username'], email=data['email'])
    new_user.set_password(data['password'])
    db.session.add(new_user)
    db.session.commit()
    
    return jsonify({"message": "Usuário registrado com sucesso"}), 201

@api_bp.route('/login', methods=['POST'])
def login():
    data = request.json
    user = User.query.filter_by(username=data['username']).first()
    if user and user.check_password(data['password']):
        access_token = create_access_token(identity=user.id)
        # Registrar tentativa de login
        login_record = LoginHistory(
            user_id=user.id,
            ip_address=request.remote_addr,
            user_agent=request.headers.get('User-Agent'),
            success=True
        )
        db.session.add(login_record)
        db.session.commit()
        return jsonify({"access_token": access_token}), 200
    else:
        # Registrar tentativa de login falho
        login_record = LoginHistory(
            user_id=user.id if user else None,
            ip_address=request.remote_addr,
            user_agent=request.headers.get('User-Agent'),
            success=False
        )
        db.session.add(login_record)
        db.session.commit()
        return jsonify({"message": "Credenciais inválidas"}), 401

@api_bp.route('/games', methods=['GET'])
@jwt_required()
def list_games():
    page = int(request.args.get('page', 1))
    per_page = int(request.args.get('per_page', 10))
    category = request.args.get('category', '')
    
    query = Game.query
    
    if category:
        query = query.filter_by(category=category)
        
    query = query.order_by(Game.start_time.desc())
    games = query.paginate(page=page, per_page=per_page, error_out=False)
    
    return jsonify({
        "games": [{
            "id": g.id,
            "name": g.name,
            "odds": g.odds,
            "start_time": g.start_time.isoformat(),
            "status": g.status,
            "category": g.category
        } for g in games.items],
        "total_pages": games.pages
    })

@api_bp.route('/bet', methods=['POST'])
@jwt_required()
def place_bet():
    current_user_id = get_jwt_identity()
    data = request.json
    
    user = User.query.get(current_user_id)
    if user.balance < data['amount']:
        return jsonify({"message": "Saldo insuficiente"}), 400
        
    game = Game.query.get(data['game_id'])
    if not game or game.status != 'active':
        return jsonify({"message": "Jogo inválido ou encerrado"}), 400
        
    bet = Bet(
        user_id=current_user_id,
        game_id=data['game_id'],
        amount=data['amount'],
        odds=game.odds
    )
    db.session.add(bet)
    
    user.balance -= data['amount']
    db.session.commit()
    
    return jsonify({"message": "Aposta realizada com sucesso"}), 201

@api_bp.route('/admin/login-history', methods=['GET'])
@jwt_required()
def get_login_history():
    current_user_id = get_jwt_identity()
    user = User.query.get(current_user_id)
    if not user.is_admin:
        return jsonify({"message": "Acesso negado"}), 403
    
    page = int(request.args.get('page', 1))
    per_page = int(request.args.get('per_page', 10))
    user_filter = request.args.get('user', '')
    date_from = request.args.get('date_from', '')
    date_to = request.args.get('date_to', '')
    
    query = LoginHistory.query
    
    if user_filter:
        user_obj = User.query.filter_by(username=user_filter).first()
        if user_obj:
            query = query.filter_by(user_id=user_obj.id)
    
    if date_from:
        query = query.filter(LoginHistory.login_time >= date_from)
    if date_to:
        query = query.filter(LoginHistory.login_time <= date_to)
    
    query = query.order_by(LoginHistory.login_time.desc())
    login_history = query.paginate(page=page, per_page=per_page, error_out=False)
    
    result = []
    for record in login_history.items:
        result.append({
            "id": record.id,
            "username": record.user.username if record.user else "Usuário não encontrado",
            "ip_address": record.ip_address,
            "user_agent": record.user_agent,
            "login_time": record.login_time.isoformat(),
            "success": record.success
        })
    
    return jsonify({
        "login_history": result,
        "total_pages": login_history.pages,
        "current_page": page
    })
    # app.py
from flask import Flask
from flask_migrate import Migrate
from flask_jwt_extended import JWTManager
from flask_cors import CORS
from config import Config
from models import db, bcrypt
from routes import api_bp

def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)
    
    # Inicializar extensões
    db.init_app(app)
    bcrypt.init_app(app)
    jwt = JWTManager(app)
    
    # Configurar CORS
    CORS(app, resources={
        r"/api/*": {
            "origins": ["http://localhost:3000"],
            "methods": ["GET", "POST", "PUT", "DELETE"],
            "allow_headers": ["Content-Type", "Authorization"]
        }
    })
    
    # Registrar blueprints
    app.register_blueprint(api_bp, url_prefix='/api')
    
    # Configurar migrações
    migrate = Migrate()
    migrate.init_app(app, db)
    
    # Middleware para registrar login
    @app.before_request
    def log_login():
        if request.endpoint == 'login':
            user = User.query.filter_by(username=request.json['username']).first()
            if user:
                LoginHistory(
                    user_id=user.id,
                    ip_address=request.remote_addr,
                    user_agent=request.headers.get('User-Agent'),
                    success=False
                ).save()
    
    return app

if __name__ == '__main__':
    app = create_app()
    app.run(debug=True)
    // src/App.js
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import Header from './components/Header';
import LoginForm from './components/LoginForm';
import RegisterForm from './components/RegisterForm';
import BetForm from './components/BetForm';
import LoginHistory from './components/LoginHistory';
import './App.css';

function App() {
  return (
    <Router>
      <div className="App">
        <Header />
        <Routes>
          <Route path="/login" element={<LoginForm />} />
          <Route path="/register" element={<RegisterForm />} />
          <Route path="/bet" element={<BetForm />} />
          <Route path="/admin/login-history" element={<LoginHistory />} />
        </Routes>
      </div>
    </Router>
  );
}

export default App;
// src/components/Header.js
import React from 'react';
import { Link } from 'react-router-dom';

const Header = () => {
  return (
    <header>
      <nav>
        <Link to="/">Início</Link>
        <Link to="/login">Login</Link>
        <Link to="/register">Cadastro</Link>
        <Link to="/bet">Fazer Aposta</Link>
        <Link to="/admin/login-history">Admin - Histórico de Login</Link>
      </nav>
    </header>
  );
};

export default Header;
// src/components/LoginForm.js
import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';

const LoginForm = () => {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const navigate = useNavigate();

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    try {
      const response = await fetch('/api/login', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({username, password})
      });
      
      const data = await response.json();
      
      if (response.ok) {
        localStorage.setItem('token', data.access_token);
        navigate('/bet');
      } else {
        setError(data.message);
      }
    } catch (err) {
      setError('Erro ao conectar com o servidor');
    }
  };

  return (
    <div className="login-form">
      <h2>Login</h2>
      {error && <div className="error">{error}</div>}
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          placeholder="Nome de usuário"
          value={username}
          onChange={(e) => setUsername(e.target.value)}
          required
        />
        <input
          type="password"
          placeholder="Senha"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
        />
        <button type="submit">Entrar</button>
      </form>
    </div>
  );
};

export default LoginForm;// src/components/RegisterForm.js
import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';

const RegisterForm = () => {
  const [username, setUsername] = useState('');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');
  const [error, setError] = useState('');
  const navigate = useNavigate();

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    if (password !== confirmPassword) {
      setError('Senhas não coincidem');
      return;
    }
    
    try {
      const response = await fetch('/api/register', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({username, email, password})
      });
      
      const data = await response.json();
      
      if (response.ok) {
        navigate('/login');
      } else {
        setError(data.message);
      }
    } catch (err) {
      setError('Erro ao conectar com o servidor');
    }
  };

  return (
    <div className="register-form">
      <h2>Cadastro</h2>
      {error && <div className="error">{error}</div>}
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          placeholder="Nome de usuário"
          value={username}
          onChange={(e) => setUsername(e.target.value)}
          required
        />
        <input
          type="email"
          placeholder="Email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
        <input
          type="password"
          placeholder="Senha"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
        />
        <input
          type="password"
          placeholder="Confirmar Senha"
          value={confirmPassword}
          onChange={(e) => setConfirmPassword(e.target.value)}
          required
        />
        <button type="submit">Cadastrar</button>
      </form>
    </div>
  );
};

export default RegisterForm;// src/components/BetForm.js
import React, { useState, useEffect } from 'react';

const BetForm = () => {
  const [games, setGames] = useState([]);
  const [selectedGame, setSelectedGame] = useState(null);
  const [betAmount, setBetAmount] = useState('');
  const [balance, setBalance] = useState(0);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    fetchGames();
    fetchUserBalance();
  }, []);

  const fetchGames = async () => {
    setLoading(true);
    try {
      const response = await fetch('/api/games');
      const data = await response.json();
      setGames(data.games);
    } catch (err) {
      console.error('Erro ao buscar jogos:', err);
    } finally {
      setLoading(false);
    }
  };

  const fetchUserBalance = async () => {
    const token = localStorage.getItem('token');
    try {
      const response = await fetch('/api/user/balance', {
        headers: {'Authorization': `Bearer ${token}`}
      });
      const data = await response.json();
      setBalance(data.balance);
    } catch (err) {
      console.error('Erro ao buscar saldo:', err);
    }
  };

  const handlePlaceBet = async () => {
    if (!selectedGame || !betAmount) {
      alert('Selecione um jogo e informe o valor da aposta');
      return;
    }

    const token = localStorage.getItem('token');
    try {
      const response = await fetch('/api/bet', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
          game_id: selectedGame.id,
          amount: parseFloat(betAmount)
        })
      });

      const data = await response.json();
      alert(data.message);
      if (response.ok) {
        fetchUserBalance();
      }
    } catch (err) {
      console.error('Erro ao fazer aposta:', err);
    }
  };

  if (loading) return <div>Carregando...</div>;

  return (
    <div className="bet-form">
      <h2>Fazer Aposta</h2>
      <div>
        <label>Jogo:</label>
        <select onChange={(e) => setSelectedGame(games.find(g => g.id === parseInt(e.target.value)))}>
          <option value="">Selecione um jogo</option>
          {games.map(game => (
            <option key={game.id} value={game.id}>
              {game.name} - Odds: {game.odds}
            </option>
          ))}
        </select>
      </div>
      <div>
        <label>Valor da Aposta:</label>
        <input 
          type="number" 
          value={betAmount}
          onChange={(e) => setBetAmount(e.target.value)}
          placeholder="R$"
        />
      </div>
      <button onClick={handlePlaceBet}>Realizar Aposta</button>
      <div>
      docker-compose.yml
      version: "3.9"

services:

  postgres:
    image: postgres:16
    container_name: nexora_postgres
    restart: always

    environment:
      POSTGRES_USER: nexora
      POSTGRES_PASSWORD: nexora123
      POSTGRES_DB: nexora_db

    ports:
      - "5432:5432"

    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
  docker-compose.yml
        <strong>Saldo disponível: R$ {balance.toFixed(2)}</strong>
      </div>
    </div>
  );
};

export default BetForm;// src/components/LoginHistory.js
import React, { useState

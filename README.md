# sss
"""
E-Mobility Logistics BI System for Kenya
Complete Single-File Flask Application
BBIT Final Year Project
"""

from flask import Flask, render_template, request, redirect, url_for, flash, jsonify, send_file
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, logout_user, login_required, current_user
from flask_bcrypt import Bcrypt
from datetime import datetime, timedelta
import secrets
import pandas as pd
import json
import io
from functools import wraps
import os

# ==================== APP CONFIGURATION ====================

app = Flask(__name__)

# Security
app.config['SECRET_KEY'] = 'e-mobility-bi-system-secret-key-2024'

# Database Configuration (SQLite for easy setup - can switch to MySQL)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///emobility_bi.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

# Initialize extensions
db = SQLAlchemy(app)
bcrypt = Bcrypt(app)
login_manager = LoginManager()
login_manager.init_app(app)
login_manager.login_view = 'login'
login_manager.login_message = 'Please log in to access this page.'

# ==================== DATABASE MODELS ====================

class User(UserMixin, db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), unique=True, nullable=False)
    email = db.Column(db.String(100), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    full_name = db.Column(db.String(100))
    role = db.Column(db.String(20), default='User')  # Admin, Manager, User
    is_active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    last_login = db.Column(db.DateTime)
    
    def get_id(self):
        return str(self.id)
    
    def is_admin(self):
        return self.role == 'Admin'
    
    def is_manager(self):
        return self.role in ['Admin', 'Manager']

class Vehicle(db.Model):
    __tablename__ = 'vehicles'
    
    id = db.Column(db.Integer, primary_key=True)
    registration_number = db.Column(db.String(20), unique=True, nullable=False)
    model = db.Column(db.String(50), nullable=False)
    battery_capacity_kwh = db.Column(db.Float, default=0)
    current_battery_level = db.Column(db.Float, default=100.00)
    status = db.Column(db.String(20), default='Available')
    purchase_date = db.Column(db.Date)
    last_maintenance_date = db.Column(db.Date)
    next_maintenance_date = db.Column(db.Date)
    total_distance_km = db.Column(db.Float, default=0)
    total_energy_consumed_kwh = db.Column(db.Float, default=0)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

class Route(db.Model):
    __tablename__ = 'routes'
    
    id = db.Column(db.Integer, primary_key=True)
    route_name = db.Column(db.String(100), nullable=False)
    origin = db.Column(db.String(100), nullable=False)
    destination = db.Column(db.String(100), nullable=False)
    distance_km = db.Column(db.Float, nullable=False)
    estimated_time_minutes = db.Column(db.Integer)
    energy_required_kwh = db.Column(db.Float)
    cost_per_km = db.Column(db.Float, default=20.00)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

class PaymentRequest(db.Model):
    __tablename__ = 'payment_requests'
    
    id = db.Column(db.Integer, primary_key=True)
    request_number = db.Column(db.String(50), unique=True, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'))
    vehicle_id = db.Column(db.Integer, db.ForeignKey('vehicles.id'))
    amount = db.Column(db.Float, nullable=False)
    payment_type = db.Column(db.String(20), default='Charging')
    description = db.Column(db.Text)
    status = db.Column(db.String(20), default='Pending')
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, onupdate=datetime.utcnow)
    
    requester = db.relationship('User', backref='payment_requests')

class Authorization(db.Model):
    __tablename__ = 'authorizations'
    
    id = db.Column(db.Integer, primary_key=True)
    payment_request_id = db.Column(db.Integer, db.ForeignKey('payment_requests.id'))
    authorized_by = db.Column(db.Integer, db.ForeignKey('users.id'))
    authorization_level = db.Column(db.Integer, default=1)
    status = db.Column(db.String(20), default='Pending')
    comments = db.Column(db.Text)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

class Committal(db.Model):
    __tablename__ = 'committals'
    
    id = db.Column(db.Integer, primary_key=True)
    payment_request_id = db.Column(db.Integer, db.ForeignKey('payment_requests.id'))
    committed_by = db.Column(db.Integer, db.ForeignKey('users.id'))
    amount = db.Column(db.Float)
    scheduled_date = db.Column(db.Date)
    status = db.Column(db.String(20), default='Pending')
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

class Transaction(db.Model):
    __tablename__ = 'transactions'
    
    id = db.Column(db.Integer, primary_key=True)
    payment_request_id = db.Column(db.Integer, db.ForeignKey('payment_requests.id'))
    transaction_reference = db.Column(db.String(100), unique=True)
    amount = db.Column(db.Float)
    transaction_date = db.Column(db.DateTime, default=datetime.utcnow)
    status = db.Column(db.String(20), default='Pending')
    payment_method = db.Column(db.String(50))
    notes = db.Column(db.Text)

class LogisticsData(db.Model):
    __tablename__ = 'logistics_data'
    
    id = db.Column(db.Integer, primary_key=True)
    vehicle_id = db.Column(db.Integer, db.ForeignKey('vehicles.id'))
    route_id = db.Column(db.Integer, db.ForeignKey('routes.id'))
    driver_name = db.Column(db.String(100))
    start_time = db.Column(db.DateTime)
    end_time = db.Column(db.DateTime)
    actual_distance_km = db.Column(db.Float)
    energy_consumed_kwh = db.Column(db.Float)
    revenue = db.Column(db.Float)
    cost = db.Column(db.Float)
    delivery_status = db.Column(db.String(20), default='Pending')
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    vehicle = db.relationship('Vehicle', backref='logistics_data')
    route = db.relationship('Route', backref='logistics_data')

class MaintenanceRecord(db.Model):
    __tablename__ = 'maintenance_records'
    
    id = db.Column(db.Integer, primary_key=True)
    vehicle_id = db.Column(db.Integer, db.ForeignKey('vehicles.id'))
    maintenance_type = db.Column(db.String(50))
    cost = db.Column(db.Float)
    date = db.Column(db.Date)
    description = db.Column(db.Text)
    next_maintenance_date = db.Column(db.Date)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

class BIReport(db.Model):
    __tablename__ = 'bi_reports'
    
    id = db.Column(db.Integer, primary_key=True)
    report_name = db.Column(db.String(100))
    report_type = db.Column(db.String(20))
    report_data = db.Column(db.Text)  # Store as JSON string
    generated_by = db.Column(db.Integer, db.ForeignKey('users.id'))
    generated_at = db.Column(db.DateTime, default=datetime.utcnow)
    file_path = db.Column(db.String(255))

# ==================== HELPER FUNCTIONS ====================

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

def role_required(*roles):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if not current_user.is_authenticated:
                return redirect(url_for('login'))
            if current_user.role not in roles:
                flash('You do not have permission to access this page.', 'error')
                return redirect(url_for('dashboard'))
            return f(*args, **kwargs)
        return decorated_function
    return decorator

def init_database():
    """Initialize database with sample data"""
    # Create all tables
    db.create_all()
    
    # Check if admin user exists
    if not User.query.filter_by(username='admin').first():
        # Create admin user
        admin = User(
            username='admin',
            email='admin@greenwheels.co.ke',
            password_hash=bcrypt.generate_password_hash('Admin123!').decode('utf-8'),
            full_name='System Administrator',
            role='Admin'
        )
        db.session.add(admin)
        
        # Create manager
        manager = User(
            username='manager1',
            email='manager@greenwheels.co.ke',
            password_hash=bcrypt.generate_password_hash('Admin123!').decode('utf-8'),
            full_name='Operations Manager',
            role='Manager'
        )
        db.session.add(manager)
        
        # Create regular user
        user = User(
            username='user1',
            email='user@greenwheels.co.ke',
            password_hash=bcrypt.generate_password_hash('User123!').decode('utf-8'),
            full_name='John Kamau',
            role='User'
        )
        db.session.add(user)
        
        # Sample Vehicles
        vehicles_data = [
            ('KCE 123A', 'Nissan e-NV200', 40.0, 'Available', 12500.0),
            ('KCE 456B', 'BYD e6', 60.0, 'In Route', 8900.0),
            ('KCE 789C', 'Renault Kangoo ZE', 33.0, 'Charging', 15000.0),
            ('KCE 101D', 'Tesla Semi', 500.0, 'Available', 5000.0),
            ('KCE 202E', 'Ford E-Transit', 68.0, 'Maintenance', 8000.0)
        ]
        
        for reg, model, battery, status, distance in vehicles_data:
            vehicle = Vehicle(
                registration_number=reg,
                model=model,
                battery_capacity_kwh=battery,
                current_battery_level=85.0,
                status=status,
                total_distance_km=distance
            )
            db.session.add(vehicle)
        
        # Sample Routes
        routes_data = [
            ('Nairobi-Mombasa', 'Nairobi', 'Mombasa', 485, 480, 97.0, 25.0),
            ('Nairobi-Nakuru', 'Nairobi', 'Nakuru', 160, 180, 32.0, 20.0),
            ('Nairobi-Kisumu', 'Nairobi', 'Kisumu', 350, 360, 70.0, 22.0),
            ('Nairobi-Nyeri', 'Nairobi', 'Nyeri', 150, 180, 30.0, 18.0),
            ('Mombasa-Malindi', 'Mombasa', 'Malindi', 120, 150, 24.0, 20.0)
        ]
        
        for name, origin, dest, distance, time, energy, cost in routes_data:
            route = Route(
                route_name=name,
                origin=origin,
                destination=dest,
                distance_km=distance,
                estimated_time_minutes=time,
                energy_required_kwh=energy,
                cost_per_km=cost
            )
            db.session.add(route)
        
        # Sample Logistics Data
        vehicles_list = Vehicle.query.all()
        routes_list = Route.query.all()
        
        if vehicles_list and routes_list:
            for i in range(5):
                log = LogisticsData(
                    vehicle_id=vehicles_list[i % len(vehicles_list)].id,
                    route_id=routes_list[i % len(routes_list)].id,
                    driver_name=f'Driver {i+1}',
                    start_time=datetime.now() - timedelta(days=i),
                    end_time=datetime.now() - timedelta(days=i-1),
                    actual_distance_km=routes_list[i % len(routes_list)].distance_km,
                    energy_consumed_kwh=routes_list[i % len(routes_list)].energy_required_kwh,
                    revenue=routes_list[i % len(routes_list)].distance_km * 30,
                    cost=routes_list[i % len(routes_list)].distance_km * routes_list[i % len(routes_list)].cost_per_km,
                    delivery_status='Completed' if i > 0 else 'In Progress'
                )
                db.session.add(log)
        
        db.session.commit()
        print("Database initialized with sample data!")

# ==================== AUTHENTICATION ROUTES ====================

@app.route('/')
def index():
    if current_user.is_authenticated:
        return redirect(url_for('dashboard'))
    return redirect(url_for('login'))

@app.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('dashboard'))
    
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        
        user = User.query.filter_by(username=username).first()
        
        if user and bcrypt.check_password_hash(user.password_hash, password):
            if user.is_active:
                login_user(user)
                user.last_login = datetime.utcnow()
                db.session.commit()
                flash(f'Welcome back, {user.full_name}!', 'success')
                return redirect(url_for('dashboard'))
            else:
                flash('Your account has been deactivated.', 'error')
        else:
            flash('Invalid username or password.', 'error')
    
    return render_template_string(LOGIN_TEMPLATE)

@app.route('/register', methods=['GET', 'POST'])
def register():
    if current_user.is_authenticated:
        return redirect(url_for('dashboard'))
    
    if request.method == 'POST':
        username = request.form.get('username')
        email = request.form.get('email')
        password = request.form.get('password')
        confirm_password = request.form.get('confirm_password')
        full_name = request.form.get('full_name')
        
        if password != confirm_password:
            flash('Passwords do not match.', 'error')
            return redirect(url_for('register'))
        
        if User.query.filter_by(username=username).first():
            flash('Username already exists.', 'error')
            return redirect(url_for('register'))
        
        if User.query.filter_by(email=email).first():
            flash('Email already registered.', 'error')
            return redirect(url_for('register'))
        
        user = User(
            username=username,
            email=email,
            password_hash=bcrypt.generate_password_hash(password).decode('utf-8'),
            full_name=full_name,
            role='User'
        )
        
        db.session.add(user)
        db.session.commit()
        
        flash('Registration successful! Please log in.', 'success')
        return redirect(url_for('login'))
    
    return render_template_string(REGISTER_TEMPLATE)

@app.route('/logout')
@login_required
def logout():
    logout_user()
    flash('You have been logged out.', 'info')
    return redirect(url_for('login'))

# ==================== DASHBOARD ROUTES ====================

@app.route('/dashboard')
@login_required
def dashboard():
    # Get dashboard statistics
    total_vehicles = Vehicle.query.count()
    active_routes = LogisticsData.query.filter_by(delivery_status='In Progress').count()
    
    pending_payments = PaymentRequest.query.filter_by(status='Pending').count()
    authorized_payments = PaymentRequest.query.filter_by(status='Authorized').count()
    committed_payments = PaymentRequest.query.filter_by(status='Committed').count()
    
    # Revenue and cost analysis
    total_revenue = db.session.query(db.func.sum(LogisticsData.revenue)).scalar() or 0
    total_cost = db.session.query(db.func.sum(LogisticsData.cost)).scalar() or 0
    profit = total_revenue - total_cost
    
    # Recent logistics data
    recent_logistics = LogisticsData.query.order_by(LogisticsData.created_at.desc()).limit(10).all()
    
    # Vehicle status distribution
    vehicle_status = db.session.query(
        Vehicle.status, db.func.count(Vehicle.id)
    ).group_by(Vehicle.status).all()
    
    # Monthly revenue trend (last 6 months)
    monthly_revenue = db.session.query(
        db.func.strftime('%Y-%m', LogisticsData.created_at),
        db.func.sum(LogisticsData.revenue)
    ).group_by(db.func.strftime('%Y-%m', LogisticsData.created_at)).limit(6).all()
    
    # Payment status distribution for chart
    payment_status = {
        'Pending': pending_payments,
        'Authorized': authorized_payments,
        'Committed': committed_payments,
        'Processed': PaymentRequest.query.filter_by(status='Processed').count()
    }
    
    return render_template_string(DASHBOARD_TEMPLATE,
                                 total_vehicles=total_vehicles,
                                 active_routes=active_routes,
                                 pending_payments=pending_payments,
                                 authorized_payments=authorized_payments,
                                 total_revenue=float(total_revenue),
                                 total_cost=float(total_cost),
                                 profit=float(profit),
                                 recent_logistics=recent_logistics,
                                 vehicle_status=vehicle_status,
                                 monthly_revenue=monthly_revenue,
                                 payment_status=payment_status)

# ==================== PAYMENT MANAGEMENT ====================

@app.route('/payments')
@login_required
def payments():
    payment_requests = PaymentRequest.query.order_by(PaymentRequest.created_at.desc()).all()
    return render_template_string(PAYMENTS_TEMPLATE, payments=payment_requests)

@app.route('/create_payment', methods=['GET', 'POST'])
@login_required
def create_payment():
    if request.method == 'POST':
        request_number = f"PAY-{datetime.now().strftime('%Y%m%d%H%M%S')}-{secrets.token_hex(4).upper()}"
        
        payment = PaymentRequest(
            request_number=request_number,
            user_id=current_user.id,
            vehicle_id=int(request.form.get('vehicle_id')) if request.form.get('vehicle_id') else None,
            amount=float(request.form.get('amount')),
            payment_type=request.form.get('payment_type'),
            description=request.form.get('description'),
            status='Pending'
        )
        
        db.session.add(payment)
        db.session.commit()
        
        flash('Payment request created successfully!', 'success')
        return redirect(url_for('payments'))
    
    vehicles = Vehicle.query.all()
    return render_template_string(CREATE_PAYMENT_TEMPLATE, vehicles=vehicles)

@app.route('/authorize_payment/<int:payment_id>', methods=['POST'])
@login_required
@role_required('Admin', 'Manager')
def authorize_payment(payment_id):
    payment = PaymentRequest.query.get_or_404(payment_id)
    
    if payment.status != 'Pending':
        flash('Payment request cannot be authorized at this stage.', 'error')
        return redirect(url_for('payments'))
    
    authorization = Authorization(
        payment_request_id=payment_id,
        authorized_by=current_user.id,
        status='Approved',
        comments=request.form.get('comments', '')
    )
    
    payment.status = 'Authorized'
    payment.updated_at = datetime.utcnow()
    
    db.session.add(authorization)
    db.session.commit()
    
    flash('Payment request authorized successfully!', 'success')
    return redirect(url_for('payments'))

@app.route('/commit_payment/<int:payment_id>', methods=['POST'])
@login_required
@role_required('Admin', 'Manager')
def commit_payment(payment_id):
    payment = PaymentRequest.query.get_or_404(payment_id)
    
    if payment.status != 'Authorized':
        flash('Payment must be authorized before committal.', 'error')
        return redirect(url_for('payments'))
    
    committal = Committal(
        payment_request_id=payment_id,
        committed_by=current_user.id,
        amount=payment.amount,
        scheduled_date=datetime.strptime(request.form.get('scheduled_date'), '%Y-%m-%d').date(),
        status='Committed'
    )
    
    payment.status = 'Committed'
    payment.updated_at = datetime.utcnow()
    
    db.session.add(committal)
    db.session.commit()
    
    flash('Payment committed successfully!', 'success')
    return redirect(url_for('payments'))

@app.route('/process_payment/<int:payment_id>', methods=['POST'])
@login_required
@role_required('Admin')
def process_payment(payment_id):
    payment = PaymentRequest.query.get_or_404(payment_id)
    
    if payment.status != 'Committed':
        flash('Payment must be committed before processing.', 'error')
        return redirect(url_for('payments'))
    
    transaction = Transaction(
        payment_request_id=payment_id,
        transaction_reference=f"TXN-{datetime.now().strftime('%Y%m%d%H%M%S')}-{payment_id}",
        amount=payment.amount,
        status='Success',
        payment_method=request.form.get('payment_method', 'Bank Transfer'),
        notes=request.form.get('notes', '')
    )
    
    payment.status = 'Processed'
    payment.updated_at = datetime.utcnow()
    
    db.session.add(transaction)
    db.session.commit()
    
    flash('Payment processed successfully!', 'success')
    return redirect(url_for('payments'))

@app.route('/reject_payment/<int:payment_id>', methods=['POST'])
@login_required
@role_required('Admin', 'Manager')
def reject_payment(payment_id):
    payment = PaymentRequest.query.get_or_404(payment_id)
    payment.status = 'Rejected'
    payment.updated_at = datetime.utcnow()
    db.session.commit()
    
    flash('Payment request rejected.', 'warning')
    return redirect(url_for('payments'))

# ==================== VEHICLE MANAGEMENT ====================

@app.route('/vehicles')
@login_required
def vehicles():
    all_vehicles = Vehicle.query.all()
    maintenance_records = MaintenanceRecord.query.order_by(MaintenanceRecord.date.desc()).limit(20).all()
    return render_template_string(VEHICLES_TEMPLATE, vehicles=all_vehicles, maintenance_records=maintenance_records)

@app.route('/register_vehicle', methods=['GET', 'POST'])
@login_required
@role_required('Admin', 'Manager')
def register_vehicle():
    if request.method == 'POST':
        vehicle = Vehicle(
            registration_number=request.form.get('registration_number'),
            model=request.form.get('model'),
            battery_capacity_kwh=float(request.form.get('battery_capacity')),
            status=request.form.get('status'),
            purchase_date=datetime.strptime(request.form.get('purchase_date'), '%Y-%m-%d').date() if request.form.get('purchase_date') else None
        )
        
        db.session.add(vehicle)
        db.session.commit()
        
        flash('Vehicle registered successfully!', 'success')
        return redirect(url_for('vehicles'))
    
    return render_template_string(REGISTER_VEHICLE_TEMPLATE)

@app.route('/update_vehicle/<int:vehicle_id>', methods=['POST'])
@login_required
@role_required('Admin', 'Manager')
def update_vehicle(vehicle_id):
    vehicle = Vehicle.query.get_or_404(vehicle_id)
    
    vehicle.current_battery_level = float(request.form.get('battery_level', vehicle.current_battery_level))
    vehicle.status = request.form.get('status', vehicle.status)
    
    db.session.commit()
    flash('Vehicle updated successfully!', 'success')
    return redirect(url_for('vehicles'))

@app.route('/add_maintenance/<int:vehicle_id>', methods=['POST'])
@login_required
@role_required('Admin', 'Manager')
def add_maintenance(vehicle_id):
    maintenance = MaintenanceRecord(
        vehicle_id=vehicle_id,
        maintenance_type=request.form.get('maintenance_type'),
        cost=float(request.form.get('cost')),
        date=datetime.strptime(request.form.get('date'), '%Y-%m-%d').date(),
        description=request.form.get('description'),
        next_maintenance_date=datetime.strptime(request.form.get('next_maintenance_date'), '%Y-%m-%d').date() if request.form.get('next_maintenance_date') else None
    )
    
    vehicle = Vehicle.query.get(vehicle_id)
    vehicle.last_maintenance_date = maintenance.date
    vehicle.next_maintenance_date = maintenance.next_maintenance_date
    vehicle.status = 'Maintenance'
    
    db.session.add(maintenance)
    db.session.commit()
    
    flash('Maintenance record added!', 'success')
    return redirect(url_for('vehicles'))

# ==================== ROUTE MANAGEMENT ====================

@app.route('/routes')
@login_required
def routes():
    all_routes = Route.query.all()
    vehicles = Vehicle.query.filter_by(status='Available').all()
    active_deliveries = LogisticsData.query.filter_by(delivery_status='In Progress').all()
    return render_template_string(ROUTES_TEMPLATE, routes=all_routes, vehicles=vehicles, active_deliveries=active_deliveries)

@app.route('/create_route', methods=['GET', 'POST'])
@login_required
@role_required('Admin', 'Manager')
def create_route():
    if request.method == 'POST':
        route = Route(
            route_name=request.form.get('route_name'),
            origin=request.form.get('origin'),
            destination=request.form.get('destination'),
            distance_km=float(request.form.get('distance_km')),
            estimated_time_minutes=int(request.form.get('estimated_time_minutes')),
            energy_required_kwh=float(request.form.get('energy_required_kwh')),
            cost_per_km=float(request.form.get('cost_per_km', 20))
        )
        
        db.session.add(route)
        db.session.commit()
        
        flash('Route created successfully!', 'success')
        return redirect(url_for('routes'))
    
    return render_template_string(CREATE_ROUTE_TEMPLATE)

@app.route('/assign_route', methods=['POST'])
@login_required
def assign_route():
    vehicle_id = request.form.get('vehicle_id')
    route_id = request.form.get('route_id')
    
    vehicle = Vehicle.query.get(vehicle_id)
    route = Route.query.get(route_id)
    
    # Create logistics entry
    logistics = LogisticsData(
        vehicle_id=vehicle_id,
        route_id=route_id,
        driver_name=request.form.get('driver_name'),
        start_time=datetime.now(),
        delivery_status='In Progress',
        revenue=route.distance_km * 30,  # Assuming rate per km
        cost=route.distance_km * route.cost_per_km
    )
    
    vehicle.status = 'In Route'
    
    db.session.add(logistics)
    db.session.commit()
    
    flash(f'Route assigned to {vehicle.registration_number}!', 'success')
    return redirect(url_for('routes'))

@app.route('/complete_delivery/<int:logistics_id>', methods=['POST'])
@login_required
def complete_delivery(logistics_id):
    logistics = LogisticsData.query.get_or_404(logistics_id)
    logistics.end_time = datetime.now()
    logistics.delivery_status = 'Completed'
    logistics.actual_distance_km = float(request.form.get('actual_distance', logistics.route.distance_km))
    logistics.energy_consumed_kwh = float(request.form.get('energy_consumed', logistics.route.energy_required_kwh))
    
    # Update vehicle stats
    vehicle = Vehicle.query.get(logistics.vehicle_id)
    vehicle.status = 'Available'
    vehicle.total_distance_km = (vehicle.total_distance_km or 0) + (logistics.actual_distance_km or 0)
    vehicle.total_energy_consumed_kwh = (vehicle.total_energy_consumed_kwh or 0) + (logistics.energy_consumed_kwh or 0)
    
    db.session.commit()
    
    flash('Delivery completed successfully!', 'success')
    return redirect(url_for('routes'))

# ==================== BUSINESS INTELLIGENCE ====================

@app.route('/bi_analytics')
@login_required
def bi_analytics():
    # KPI Calculations
    total_revenue = db.session.query(db.func.sum(LogisticsData.revenue)).scalar() or 0
    total_cost = db.session.query(db.func.sum(LogisticsData.cost)).scalar() or 0
    total_deliveries = LogisticsData.query.count()
    completed_deliveries = LogisticsData.query.filter_by(delivery_status='Completed').count()
    completion_rate = (completed_deliveries/total_deliveries*100 if total_deliveries>0 else 0)
    
    # Energy consumption analysis
    energy_data = db.session.query(
        Vehicle.model,
        db.func.avg(LogisticsData.energy_consumed_kwh).label('avg_energy'),
        db.func.avg(LogisticsData.actual_distance_km).label('avg_distance')
    ).join(Vehicle, LogisticsData.vehicle_id == Vehicle.id)\
     .group_by(Vehicle.model).all()
    
    # Cost analysis by route
    cost_analysis = db.session.query(
        Route.route_name,
        db.func.avg(LogisticsData.cost).label('avg_cost'),
        db.func.avg(LogisticsData.revenue).label('avg_revenue'),
        db.func.count(LogisticsData.id).label('trip_count')
    ).join(Route, LogisticsData.route_id == Route.id)\
     .group_by(Route.route_name).all()
    
    # Monthly trend
    monthly_trend = db.session.query(
        db.func.strftime('%Y-%m', LogisticsData.created_at).label('month'),
        db.func.sum(LogisticsData.revenue).label('revenue'),
        db.func.sum(LogisticsData.cost).label('cost'),
        db.func.count(LogisticsData.id).label('trips')
    ).group_by('month').order_by('month').limit(12).all()
    
    # Vehicle performance
    vehicle_performance = db.session.query(
        Vehicle.registration_number,
        Vehicle.model,
        db.func.sum(LogisticsData.revenue).label('total_revenue'),
        db.func.sum(LogisticsData.cost).label('total_cost'),
        db.func.sum(LogisticsData.actual_distance_km).label('total_distance'),
        db.func.count(LogisticsData.id).label('trips_completed')
    ).join(LogisticsData, Vehicle.id == LogisticsData.vehicle_id)\
     .group_by(Vehicle.id).all()
    
    # Profit trend
    profit_trend = [(month[0], float(month[1] or 0) - float(month[2] or 0)) for month in monthly_trend]
    
    return render_template_string(BI_ANALYTICS_TEMPLATE,
                                 total_revenue=float(total_revenue),
                                 total_cost=float(total_cost),
                                 profit=float(total_revenue - total_cost),
                                 total_deliveries=total_deliveries,
                                 completion_rate=completion_rate,
                                 energy_data=energy_data,
                                 cost_analysis=cost_analysis,
                                 monthly_trend=monthly_trend,
                                 profit_trend=profit_trend,
                                 vehicle_performance=vehicle_performance)

# ==================== REPORTING ====================

@app.route('/reports')
@login_required
def reports():
    generated_reports = BIReport.query.order_by(BIReport.generated_at.desc()).limit(20).all()
    return render_template_string(REPORTS_TEMPLATE, reports=generated_reports)

@app.route('/generate_report', methods=['POST'])
@login_required
def generate_report():
    report_type = request.form.get('report_type')
    date_from = request.form.get('date_from')
    date_to = request.form.get('date_to')
    
    # Build query based on report type
    query = LogisticsData.query
    
    if date_from:
        query = query.filter(LogisticsData.created_at >= datetime.strptime(date_from, '%Y-%m-%d'))
    if date_to:
        query = query.filter(LogisticsData.created_at <= datetime.strptime(date_to, '%Y-%m-%d'))
    
    data = query.all()
    
    # Convert to list of dicts for JSON storage
    report_data = []
    for log in data:
        report_data.append({
            'Date': log.created_at.strftime('%Y-%m-%d'),
            'Driver': log.driver_name,
            'Vehicle': log.vehicle.registration_number if log.vehicle else 'N/A',
            'Route': log.route.route_name if log.route else 'N/A',
            'Distance (km)': float(log.actual_distance_km or 0),
            'Energy (kWh)': float(log.energy_consumed_kwh or 0),
            'Revenue (KES)': float(log.revenue or 0),
            'Cost (KES)': float(log.cost or 0),
            'Profit (KES)': float((log.revenue or 0) - (log.cost or 0)),
            'Status': log.delivery_status
        })
    
    # Save report to database
    report = BIReport(
        report_name=f"{report_type}_Report_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
        report_type=report_type,
        report_data=json.dumps(report_data),
        generated_by=current_user.id
    )
    
    db.session.add(report)
    db.session.commit()
    
    flash(f'{report_type} report generated successfully!', 'success')
    return redirect(url_for('reports'))

@app.route('/view_report/<int:report_id>')
@login_required
def view_report(report_id):
    report = BIReport.query.get_or_404(report_id)
    report_data = json.loads(report.report_data)
    return render_template_string(VIEW_REPORT_TEMPLATE, report=report, data=report_data)

@app.route('/api/dashboard_data')
@login_required
def dashboard_api():
    """API endpoint for real-time dashboard data"""
    data = {
        'total_vehicles': Vehicle.query.count(),
        'active_routes': LogisticsData.query.filter_by(delivery_status='In Progress').count(),
        'pending_payments': PaymentRequest.query.filter_by(status='Pending').count(),
        'total_revenue': float(db.session.query(db.func.sum(LogisticsData.revenue)).scalar() or 0),
        'recent_activities': []
    }
    return jsonify(data)

# ==================== HTML TEMPLATES ====================

# Base Template
BASE_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}E-Mobility BI System - Kenya{% endblock %}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
        }
        
        .navbar {
            background: linear-gradient(135deg, #2d3748 0%, #1a202c 100%);
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }
        
        .navbar-brand {
            font-size: 1.5rem;
            font-weight: bold;
        }
        
        .sidebar {
            position: fixed;
            top: 56px;
            left: 0;
            height: calc(100% - 56px);
            width: 260px;
            background: linear-gradient(180deg, #2d3748 0%, #1a202c 100%);
            color: white;
            transition: all 0.3s;
            z-index: 1000;
            overflow-y: auto;
        }
        
        .sidebar .nav-link {
            color: #cbd5e0;
            padding: 12px 20px;
            transition: all 0.3s;
            border-radius: 8px;
            margin: 4px 10px;
        }
        
        .sidebar .nav-link:hover {
            background: rgba(255,255,255,0.1);
            color: white;
            transform: translateX(5px);
        }
        
        .sidebar .nav-link.active {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        
        .main-content {
            margin-left: 260px;
            padding: 20px;
            margin-top: 56px;
        }
        
        .card {
            border-radius: 15px;
            box-shadow: 0 10px 40px rgba(0,0,0,0.1);
            border: none;
            transition: transform 0.3s;
        }
        
        .card:hover {
            transform: translateY(-5px);
        }
        
        .stat-card {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        
        .btn-primary {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            border: none;
        }
        
        .btn-primary:hover {
            transform: translateY(-2px);
            box-shadow: 0 5px 15px rgba(102,126,234,0.4);
        }
        
        @media (max-width: 768px) {
            .sidebar {
                margin-left: -260px;
            }
            .sidebar.active {
                margin-left: 0;
            }
            .main-content {
                margin-left: 0;
            }
        }
        
        .table {
            background: white;
            border-radius: 10px;
            overflow: hidden;
        }
        
        .badge {
            padding: 5px 12px;
            border-radius: 20px;
        }
        
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(20px); }
            to { opacity: 1; transform: translateY(0); }
        }
        
        .fade-in {
            animation: fadeIn 0.5s ease-out;
        }
    </style>
    {% block extra_css %}{% endblock %}
</head>
<body>
    {% if current_user.is_authenticated %}
    <nav class="navbar navbar-dark fixed-top">
        <div class="container-fluid">
            <button class="btn btn-dark d-md-none" type="button" onclick="toggleSidebar()">
                <i class="fas fa-bars"></i>
            </button>
            <a class="navbar-brand" href="{{ url_for('dashboard') }}">
                <i class="fas fa-charging-station"></i> GreenWheels BI
            </a>
            <div class="dropdown">
                <button class="btn btn-dark dropdown-toggle" type="button" data-bs-toggle="dropdown">
                    <i class="fas fa-user"></i> {{ current_user.full_name }}
                </button>
                <ul class="dropdown-menu dropdown-menu-end">
                    <li><span class="dropdown-item-text"><i class="fas fa-user-tag"></i> Role: {{ current_user.role }}</span></li>
                    <li><hr class="dropdown-divider"></li>
                    <li><a class="dropdown-item" href="{{ url_for('logout') }}"><i class="fas fa-sign-out-alt"></i> Logout</a></li>
                </ul>
            </div>
        </div>
    </nav>

    <div class="sidebar">
        <div class="p-3">
            <h5 class="text-center mb-4">E-Mobility BI</h5>
        </div>
        <nav class="nav flex-column">
            <a class="nav-link" href="{{ url_for('dashboard') }}">
                <i class="fas fa-tachometer-alt"></i> Dashboard
            </a>
            <a class="nav-link" href="{{ url_for('payments') }}">
                <i class="fas fa-money-bill-wave"></i> Payments
            </a>
            <a class="nav-link" href="{{ url_for('vehicles') }}">
                <i class="fas fa-truck"></i> Vehicles
            </a>
            <a class="nav-link" href="{{ url_for('routes') }}">
                <i class="fas fa-map-marked-alt"></i> Routes
            </a>
            <a class="nav-link" href="{{ url_for('bi_analytics') }}">
                <i class="fas fa-chart-line"></i> BI Analytics
            </a>
            <a class="nav-link" href="{{ url_for('reports') }}">
                <i class="fas fa-file-alt"></i> Reports
            </a>
        </nav>
    </div>
    {% endif %}

    <div class="main-content">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category if category != 'error' else 'danger' }} alert-dismissible fade show fade-in">
                        {{ message }}
                        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        
        {% block content %}{% endblock %}
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
    <script>
        function toggleSidebar() {
            document.querySelector('.sidebar').classList.toggle('active');
        }
        
        // Auto-hide alerts after 5 seconds
        setTimeout(function() {
            $('.alert').fadeOut('slow');
        }, 5000);
    </script>
    {% block scripts %}{% endblock %}
</body>
</html>
'''

# Login Template
LOGIN_TEMPLATE = '''
{% extends base_template %}
{% block title %}Login - E-Mobility BI System{% endblock %}
{% block content %}
<div class="row justify-content-center min-vh-100 align-items-center">
    <div class="col-md-4">
        <div class="card fade-in">
            <div class="card-header bg-primary text-white text-center">
                <h4><i class="fas fa-charging-station"></i> GreenWheels Kenya</h4>
                <h6>E-Mobility Logistics BI System</h6>
            </div>
            <div class="card-body">
                <form method="POST">
                    <div class="mb-3">
                        <label class="form-label"><i class="fas fa-user"></i> Username</label>
                        <input type="text" name="username" class="form-control" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label"><i class="fas fa-lock"></i> Password</label>
                        <input type="password" name="password" class="form-control" required>
                    </div>
                    <button type="submit" class="btn btn-primary w-100">
                        <i class="fas fa-sign-in-alt"></i> Login
                    </button>
                </form>
                <hr>
                <p class="text-center mb-0">
                    <a href="{{ url_for('register') }}">Create Account</a>
                </p>
                <div class="alert alert-info mt-3 small">
                    <strong><i class="fas fa-info-circle"></i> Demo Credentials:</strong><br>
                    Admin: admin / Admin123!<br>
                    Manager: manager1 / Admin123!<br>
                    User: user1 / User123!
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
'''

# Register Template
REGISTER_TEMPLATE = '''
{% extends base_template %}
{% block title %}Register - E-Mobility BI System{% endblock %}
{% block content %}
<div class="row justify-content-center min-vh-100 align-items-center">
    <div class="col-md-6">
        <div class="card fade-in">
            <div class="card-header bg-success text-white text-center">
                <h4><i class="fas fa-user-plus"></i> Create Account</h4>
            </div>
            <div class="card-body">
                <form method="POST">
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label><i class="fas fa-user"></i> Username *</label>
                            <input type="text" name="username" class="form-control" required>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label><i class="fas fa-envelope"></i> Email *</label>
                            <input type="email" name="email" class="form-control" required>
                        </div>
                    </div>
                    <div class="mb-3">
                        <label><i class="fas fa-id-card"></i> Full Name *</label>
                        <input type="text" name="full_name" class="form-control" required>
                    </div>
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label><i class="fas fa-lock"></i> Password *</label>
                            <input type="password" name="password" class="form-control" required>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label><i class="fas fa-lock"></i> Confirm Password *</label>
                            <input type="password" name="confirm_password" class="form-control" required>
                        </div>
                    </div>
                    <button type="submit" class="btn btn-success w-100">
                        <i class="fas fa-check"></i> Register
                    </button>
                </form>
                <hr>
                <p class="text-center mb-0">
                    Already have an account? <a href="{{ url_for('login') }}">Login</a>
                </p>
            </div>
        </div>
    </div>
</div>
{% endblock %}
'''

# Dashboard Template
DASHBOARD_TEMPLATE = '''
{% extends base_template %}
{% block title %}Dashboard{% endblock %}
{% block content %}
<div class="fade-in">
    <div class="row mb-4">
        <div class="col-12">
            <h2><i class="fas fa-tachometer-alt"></i> Dashboard</h2>
            <p class="text-muted">Welcome back, {{ current_user.full_name }}!</p>
        </div>
    </div>

    <div class="row mb-4">
        <div class="col-md-3">
            <div class="card stat-card">
                <div class="card-body">
                    <h6 class="card-title">Total Vehicles</h6>
                    <h2 class="mb-0">{{ total_vehicles }}</h2>
                    <i class="fas fa-truck fa-2x float-end"></i>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="card bg-warning text-white">
                <div class="card-body">
                    <h6 class="card-title">Active Routes</h6>
                    <h2 class="mb-0">{{ active_routes }}</h2>
                    <i class="fas fa-route fa-2x float-end"></i>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="card bg-info text-white">
                <div class="card-body">
                    <h6 class="card-title">Pending Payments</h6>
                    <h2 class="mb-0">{{ pending_payments }}</h2>
                    <i class="fas fa-clock fa-2x float-end"></i>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="card bg-success text-white">
                <div class="card-body">
                    <h6 class="card-title">Total Revenue</h6>
                    <h4 class="mb-0">KES {{ "{:,.0f}".format(total_revenue) }}</h4>
                    <i class="fas fa-chart-line fa-2x float-end"></i>
                </div>
            </div>
        </div>
    </div>

    <div class="row mb-4">
        <div class="col-md-6">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-chart-line"></i> Revenue & Cost Analysis</h5>
                </div>
                <div class="card-body">
                    <canvas id="revenueChart"></canvas>
                </div>
            </div>
        </div>
        <div class="col-md-6">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-chart-pie"></i> Payment Status</h5>
                </div>
                <div class="card-body">
                    <canvas id="paymentChart"></canvas>
                </div>
            </div>
        </div>
    </div>

    <div class="row">
        <div class="col-md-12">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-history"></i> Recent Logistics Activities</h5>
                </div>
                <div class="card-body">
                    <div class="table-responsive">
                        <table class="table table-hover">
                            <thead>
                                <tr>
                                    <th>Date</th>
                                    <th>Driver</th>
                                    <th>Vehicle</th>
                                    <th>Route</th>
                                    <th>Revenue (KES)</th>
                                    <th>Cost (KES)</th>
                                    <th>Status</th>
                                </tr>
                            </thead>
                            <tbody>
                                {% for log in recent_logistics %}
                                <tr>
                                    <td>{{ log.created_at.strftime('%Y-%m-%d') }}</td>
                                    <td>{{ log.driver_name }}</td>
                                    <td>{{ log.vehicle.registration_number if log.vehicle else 'N/A' }}</td>
                                    <td>{{ log.route.route_name if log.route else 'N/A' }}</td>
                                    <td>{{ "{:,.2f}".format(log.revenue or 0) }}</td>
                                    <td>{{ "{:,.2f}".format(log.cost or 0) }}</td>
                                    <td>
                                        <span class="badge bg-{{ 'success' if log.delivery_status == 'Completed' else 'warning' if log.delivery_status == 'In Progress' else 'secondary' }}">
                                            {{ log.delivery_status }}
                                        </span>
                                    </td>
                                </tr>
                                {% endfor %}
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>

<script>
// Revenue Chart
const revenueCtx = document.getElementById('revenueChart').getContext('2d');
new Chart(revenueCtx, {
    type: 'line',
    data: {
        labels: [{% for month in monthly_revenue %}'{{ month[0] }}',{% endfor %}],
        datasets: [{
            label: 'Revenue (KES)',
            data: [{% for month in monthly_revenue %}{{ month[1] or 0 }},{% endfor %}],
            borderColor: '#667eea',
            backgroundColor: 'rgba(102,126,234,0.1)',
            tension: 0.4,
            fill: true
        }]
    },
    options: {
        responsive: true,
        plugins: {
            legend: { position: 'top' }
        }
    }
});

// Payment Chart
const paymentCtx = document.getElementById('paymentChart').getContext('2d');
new Chart(paymentCtx, {
    type: 'doughnut',
    data: {
        labels: ['Pending', 'Authorized', 'Committed', 'Processed'],
        datasets: [{
            data: [{{ payment_status.Pending }}, {{ payment_status.Authorized }}, {{ payment_status.Committed }}, {{ payment_status.Processed or 0 }}],
            backgroundColor: ['#ffc107', '#17a2b8', '#28a745', '#6f42c1']
        }]
    }
});
</script>
{% endblock %}
'''

# Payments Template
PAYMENTS_TEMPLATE = '''
{% extends base_template %}
{% block title %}Payment Management{% endblock %}
{% block content %}
<div class="fade-in">
    <div class="row mb-4">
        <div class="col-12">
            <h2><i class="fas fa-money-bill-wave"></i> Payment Management</h2>
            <a href="{{ url_for('create_payment') }}" class="btn btn-primary">
                <i class="fas fa-plus"></i> New Payment Request
            </a>
        </div>
    </div>

    <div class="card">
        <div class="card-header">
            <h5><i class="fas fa-list"></i> Payment Requests</h5>
        </div>
        <div class="card-body">
            <div class="table-responsive">
                <table class="table table-hover">
                    <thead>
                        <tr>
                            <th>Request #</th>
                            <th>Date</th>
                            <th>Requester</th>
                            <th>Amount (KES)</th>
                            <th>Type</th>
                            <th>Status</th>
                            <th>Actions</th>
                        </tr>
                    </thead>
                    <tbody>
                        {% for payment in payments %}
                        <tr>
                            <td>{{ payment.request_number }}</td>
                            <td>{{ payment.created_at.strftime('%Y-%m-%d') }}</td>
                            <td>{{ payment.requester.full_name }}</td>
                            <td>{{ "{:,.2f}".format(payment.amount) }}</td>
                            <td>{{ payment.payment_type }}</td>
                            <td>
                                <span class="badge bg-{{ 'warning' if payment.status == 'Pending' else 'info' if payment.status == 'Authorized' else 'primary' if payment.status == 'Committed' else 'success' if payment.status == 'Processed' else 'danger' }}">
                                    {{ payment.status }}
                                </span>
                            </td>
                            <td>
                                {% if payment.status == 'Pending' and current_user.role in ['Admin', 'Manager'] %}
                                <form method="POST" action="{{ url_for('authorize_payment', payment_id=payment.id) }}" style="display:inline">
                                    <input type="text" name="comments" placeholder="Comments" class="form-control form-control-sm d-inline-block w-auto">
                                    <button type="submit" class="btn btn-sm btn-success">Authorize</button>
                                </form>
                                {% endif %}
                                {% if payment.status == 'Authorized' and current_user.role in ['Admin', 'Manager'] %}
                                <form method="POST" action="{{ url_for('commit_payment', payment_id=payment.id) }}" style="display:inline">
                                    <input type="date" name="scheduled_date" class="form-control form-control-sm d-inline-block w-auto" required>
                                    <button type="submit" class="btn btn-sm btn-primary">Commit</button>
                                </form>
                                {% endif %}
                                {% if payment.status == 'Committed' and current_user.role == 'Admin' %}
                                <form method="POST" action="{{ url_for('process_payment', payment_id=payment.id) }}" style="display:inline">
                                    <select name="payment_method" class="form-control form-control-sm d-inline-block w-auto">
                                        <option>Bank Transfer</option>
                                        <option>M-Pesa</option>
                                        <option>Cheque</option>
                                    </select>
                                    <button type="submit" class="btn btn-sm btn-info">Process</button>
                                </form>
                                {% endif %}
                                {% if payment.status in ['Pending', 'Authorized'] and current_user.role in ['Admin', 'Manager'] %}
                                <form method="POST" action="{{ url_for('reject_payment', payment_id=payment.id) }}" style="display:inline">
                                    <button type="submit" class="btn btn-sm btn-danger" onclick="return confirm('Reject this payment?')">Reject</button>
                                </form>
                                {% endif %}
                            </td>
                        </tr>
                        {% endfor %}
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</div>
{% endblock %}
'''

# Create Payment Template
CREATE_PAYMENT_TEMPLATE = '''
{% extends base_template %}
{% block title %}Create Payment Request{% endblock %}
{% block content %}
<div class="row justify-content-center fade-in">
    <div class="col-md-6">
        <div class="card">
            <div class="card-header">
                <h5><i class="fas fa-plus-circle"></i> Create Payment Request</h5>
            </div>
            <div class="card-body">
                <form method="POST">
                    <div class="mb-3">
                        <label>Amount (KES) *</label>
                        <input type="number" name="amount" class="form-control" step="0.01" required>
                    </div>
                    <div class="mb-3">
                        <label>Payment Type *</label>
                        <select name="payment_type" class="form-control" required>
                            <option value="Charging">Charging</option>
                            <option value="Maintenance">Maintenance</option>
                            <option value="Salary">Salary</option>
                            <option value="Other">Other</option>
                        </select>
                    </div>
                    <div class="mb-3">
                        <label>Vehicle (Optional)</label>
                        <select name="vehicle_id" class="form-control">
                            <option value="">Select Vehicle</option>
                            {% for vehicle in vehicles %}
                            <option value="{{ vehicle.id }}">{{ vehicle.registration_number }} - {{ vehicle.model }}</option>
                            {% endfor %}
                        </select>
                    </div>
                    <div class="mb-3">
                        <label>Description</label>
                        <textarea name="description" class="form-control" rows="3"></textarea>
                    </div>
                    <button type="submit" class="btn btn-primary">Submit Request</button>
                    <a href="{{ url_for('payments') }}" class="btn btn-secondary">Cancel</a>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}
'''

# Vehicles Template
VEHICLES_TEMPLATE = '''
{% extends base_template %}
{% block title %}Vehicle Management{% endblock %}
{% block content %}
<div class="fade-in">
    <div class="row mb-4">
        <div class="col-12">
            <h2><i class="fas fa-truck"></i> Vehicle Management</h2>
            <a href="{{ url_for('register_vehicle') }}" class="btn btn-primary">
                <i class="fas fa-plus"></i> Register Vehicle
            </a>
        </div>
    </div>

    <div class="row">
        <div class="col-md-8">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-list"></i> Fleet Inventory</h5>
                </div>
                <div class="card-body">
                    <div class="table-responsive">
                        <table class="table table-hover">
                            <thead>
                                <tr>
                                    <th>Reg #</th>
                                    <th>Model</th>
                                    <th>Battery (kWh)</th>
                                    <th>Current Level</th>
                                    <th>Status</th>
                                    <th>Total Distance</th>
                                    <th>Actions</th>
                                </tr>
                            </thead>
                            <tbody>
                                {% for vehicle in vehicles %}
                                <tr>
                                    <td><strong>{{ vehicle.registration_number }}</strong></td>
                                    <td>{{ vehicle.model }}</td>
                                    <td>{{ vehicle.battery_capacity_kwh }}</td>
                                    <td>
                                        <div class="progress" style="height: 20px;">
                                            <div class="progress-bar bg-{{ 'success' if vehicle.current_battery_level > 50 else 'warning' if vehicle.current_battery_level > 20 else 'danger' }}" 
                                                 style="width: {{ vehicle.current_battery_level }}%">
                                                {{ vehicle.current_battery_level }}%
                                            </div>
                                        </div>
                                    </td>
                                    <td>
                                        <span class="badge bg-{{ 'success' if vehicle.status == 'Available' else 'warning' if vehicle.status == 'In Route' else 'info' if vehicle.status == 'Charging' else 'danger' }}">
                                            {{ vehicle.status }}
                                        </span>
                                    </td>
                                    <td>{{ "{:,.0f}".format(vehicle.total_distance_km) }} km</td>
                                    <td>
                                        <button class="btn btn-sm btn-info" data-bs-toggle="modal" data-bs-target="#updateModal{{ vehicle.id }}">
                                            <i class="fas fa-edit"></i>
                                        </button>
                                        <button class="btn btn-sm btn-warning" data-bs-toggle="modal" data-bs-target="#maintenanceModal{{ vehicle.id }}">
                                            <i class="fas fa-tools"></i>
                                        </button>
                                    </td>
                                </tr>
                                
                                <!-- Update Modal -->
                                <div class="modal fade" id="updateModal{{ vehicle.id }}" tabindex="-1">
                                    <div class="modal-dialog">
                                        <div class="modal-content">
                                            <form method="POST" action="{{ url_for('update_vehicle', vehicle_id=vehicle.id) }}">
                                                <div class="modal-header">
                                                    <h5 class="modal-title">Update Vehicle</h5>
                                                    <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
                                                </div>
                                                <div class="modal-body">
                                                    <div class="mb-3">
                                                        <label>Battery Level (%)</label>
                                                        <input type="number" name="battery_level" class="form-control" step="0.1" value="{{ vehicle.current_battery_level }}">
                                                    </div>
                                                    <div class="mb-3">
                                                        <label>Status</label>
                                                        <select name="status" class="form-control">
                                                            <option value="Available" {% if vehicle.status == 'Available' %}selected{% endif %}>Available</option>
                                                            <option value="In Route" {% if vehicle.status == 'In Route' %}selected{% endif %}>In Route</option>
                                                            <option value="Charging" {% if vehicle.status == 'Charging' %}selected{% endif %}>Charging</option>
                                                            <option value="Maintenance" {% if vehicle.status == 'Maintenance' %}selected{% endif %}>Maintenance</option>
                                                        </select>
                                                    </div>
                                                </div>
                                                <div class="modal-footer">
                                                    <button type="submit" class="btn btn-primary">Update</button>
                                                </div>
                                            </form>
                                        </div>
                                    </div>
                                </div>
                                
                                <!-- Maintenance Modal -->
                                <div class="modal fade" id="maintenanceModal{{ vehicle.id }}" tabindex="-1">
                                    <div class="modal-dialog">
                                        <div class="modal-content">
                                            <form method="POST" action="{{ url_for('add_maintenance', vehicle_id=vehicle.id) }}">
                                                <div class="modal-header">
                                                    <h5 class="modal-title">Add Maintenance Record</h5>
                                                    <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
                                                </div>
                                                <div class="modal-body">
                                                    <div class="mb-3">
                                                        <label>Maintenance Type</label>
                                                        <input type="text" name="maintenance_type" class="form-control" required>
                                                    </div>
                                                    <div class="mb-3">
                                                        <label>Cost (KES)</label>
                                                        <input type="number" name="cost" class="form-control" step="0.01" required>
                                                    </div>
                                                    <div class="mb-3">
                                                        <label>Date</label>
                                                        <input type="date" name="date" class="form-control" required>
                                                    </div>
                                                    <div class="mb-3">
                                                        <label>Description</label>
                                                        <textarea name="description" class="form-control"></textarea>
                                                    </div>
                                                    <div class="mb-3">
                                                        <label>Next Maintenance Date</label>
                                                        <input type="date" name="next_maintenance_date" class="form-control">
                                                    </div>
                                                </div>
                                                <div class="modal-footer">
                                                    <button type="submit" class="btn btn-primary">Add Record</button>
                                                </div>
                                            </form>
                                        </div>
                                    </div>
                                </div>
                                {% endfor %}
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>
        
        <div class="col-md-4">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-clipboard-list"></i> Recent Maintenance</h5>
                </div>
                <div class="card-body">
                    <div class="list-group">
                        {% for record in maintenance_records %}
                        <div class="list-group-item">
                            <div class="d-flex justify-content-between">
                                <strong>{{ record.maintenance_type }}</strong>
                                <small>{{ record.date.strftime('%Y-%m-%d') }}</small>
                            </div>
                            <div>Vehicle: {{ record.vehicle.registration_number }}</div>
                            <div>Cost: KES {{ "{:,.2f}".format(record.cost) }}</div>
                        </div>
                        {% endfor %}
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
'''

# Routes Template
ROUTES_TEMPLATE = '''
{% extends base_template %}
{% block title %}Route Management{% endblock %}
{% block content %}
<div class="fade-in">
    <div class="row mb-4">
        <div class="col-12">
            <h2><i class="fas fa-map-marked-alt"></i> Route Management</h2>
            <a href="{{ url_for('create_route') }}" class="btn btn-primary">
                <i class="fas fa-plus"></i> Create Route
            </a>
        </div>
    </div>

    <div class="row">
        <div class="col-md-7">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-road"></i> Available Routes</h5>
                </div>
                <div class="card-body">
                    <div class="table-responsive">
                        <table class="table table-hover">
                            <thead>
                                <tr>
                                    <th>Route Name</th>
                                    <th>Origin → Destination</th>
                                    <th>Distance (km)</th>
                                    <th>Est. Time</th>
                                    <th>Energy (kWh)</th>
                                </tr>
                            </thead>
                            <tbody>
                                {% for route in routes %}
                                <tr>
                                    <td><strong>{{ route.route_name }}</strong></td>
                                    <td>{{ route.origin }} → {{ route.destination }}</td>
                                    <td>{{ route.distance_km }}</td>
                                    <td>{{ route.estimated_time_minutes }} min</td>
                                    <td>{{ route.energy_required_kwh }}</td>
                                </tr>
                                {% endfor %}
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>
        
        <div class="col-md-5">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-tasks"></i> Assign Route</h5>
                </div>
                <div class="card-body">
                    <form method="POST" action="{{ url_for('assign_route') }}">
                        <div class="mb-3">
                            <label>Select Vehicle</label>
                            <select name="vehicle_id" class="form-control" required>
                                <option value="">Choose Vehicle</option>
                                {% for vehicle in vehicles %}
                                <option value="{{ vehicle.id }}">{{ vehicle.registration_number }} - {{ vehicle.model }}</option>
                                {% endfor %}
                            </select>
                        </div>
                        <div class="mb-3">
                            <label>Select Route</label>
                            <select name="route_id" class="form-control" required>
                                <option value="">Choose Route</option>
                                {% for route in routes %}
                                <option value="{{ route.id }}">{{ route.route_name }} ({{ route.distance_km }} km)</option>
                                {% endfor %}
                            </select>
                        </div>
                        <div class="mb-3">
                            <label>Driver Name</label>
                            <input type="text" name="driver_name" class="form-control" required>
                        </div>
                        <button type="submit" class="btn btn-success w-100">
                            <i class="fas fa-play"></i> Start Delivery
                        </button>
                    </form>
                </div>
            </div>
            
            <div class="card mt-3">
                <div class="card-header">
                    <h5><i class="fas fa-charging-station"></i> Active Deliveries</h5>
                </div>
                <div class="card-body">
                    {% for delivery in active_deliveries %}
                    <div class="border-bottom mb-2 pb-2">
                        <div><strong>{{ delivery.driver_name }}</strong></div>
                        <div>Vehicle: {{ delivery.vehicle.registration_number }}</div>
                        <div>Route: {{ delivery.route.route_name }}</div>
                        <div>Started: {{ delivery.start_time.strftime('%Y-%m-%d %H:%M') }}</div>
                        <form method="POST" action="{{ url_for('complete_delivery', logistics_id=delivery.id) }}" class="mt-2">
                            <div class="row">
                                <div class="col-6">
                                    <input type="number" name="actual_distance" placeholder="Actual km" class="form-control form-control-sm">
                                </div>
                                <div class="col-6">
                                    <input type="number" name="energy_consumed" placeholder="Energy kWh" class="form-control form-control-sm">
                                </div>
                            </div>
                            <button type="submit" class="btn btn-sm btn-success mt-2 w-100">Complete Delivery</button>
                        </form>
                    </div>
                    {% else %}
                    <p class="text-muted">No active deliveries</p>
                    {% endfor %}
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
'''

# Create Route Template
CREATE_ROUTE_TEMPLATE = '''
{% extends base_template %}
{% block title %}Create Route{% endblock %}
{% block content %}
<div class="row justify-content-center fade-in">
    <div class="col-md-6">
        <div class="card">
            <div class="card-header">
                <h5><i class="fas fa-plus-circle"></i> Create New Route</h5>
            </div>
            <div class="card-body">
                <form method="POST">
                    <div class="mb-3">
                        <label>Route Name *</label>
                        <input type="text" name="route_name" class="form-control" required>
                    </div>
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label>Origin *</label>
                            <input type="text" name="origin" class="form-control" required>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label>Destination *</label>
                            <input type="text" name="destination" class="form-control" required>
                        </div>
                    </div>
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label>Distance (km) *</label>
                            <input type="number" name="distance_km" class="form-control" step="0.1" required>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label>Est. Time (minutes)</label>
                            <input type="number" name="estimated_time_minutes" class="form-control">
                        </div>
                    </div>
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label>Energy Required (kWh)</label>
                            <input type="number" name="energy_required_kwh" class="form-control" step="0.1">
                        </div>
                        <div class="col-md-6 mb-3">
                            <label>Cost per km (KES)</label>
                            <input type="number" name="cost_per_km" class="form-control" step="0.01" value="20">
                        </div>
                    </div>
                    <button type="submit" class="btn btn-primary">Create Route</button>
                    <a href="{{ url_for('routes') }}" class="btn btn-secondary">Cancel</a>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}
'''

# Register Vehicle Template
REGISTER_VEHICLE_TEMPLATE = '''
{% extends base_template %}
{% block title %}Register Vehicle{% endblock %}
{% block content %}
<div class="row justify-content-center fade-in">
    <div class="col-md-6">
        <div class="card">
            <div class="card-header">
                <h5><i class="fas fa-plus-circle"></i> Register New Vehicle</h5>
            </div>
            <div class="card-body">
                <form method="POST">
                    <div class="mb-3">
                        <label>Registration Number *</label>
                        <input type="text" name="registration_number" class="form-control" required>
                    </div>
                    <div class="mb-3">
                        <label>Model *</label>
                        <input type="text" name="model" class="form-control" required>
                    </div>
                    <div class="mb-3">
                        <label>Battery Capacity (kWh)</label>
                        <input type="number" name="battery_capacity" class="form-control" step="0.1">
                    </div>
                    <div class="mb-3">
                        <label>Purchase Date</label>
                        <input type="date" name="purchase_date" class="form-control">
                    </div>
                    <div class="mb-3">
                        <label>Status</label>
                        <select name="status" class="form-control">
                            <option value="Available">Available</option>
                            <option value="Maintenance">Maintenance</option>
                            <option value="Charging">Charging</option>
                        </select>
                    </div>
                    <button type="submit" class="btn btn-primary">Register Vehicle</button>
                    <a href="{{ url_for('vehicles') }}" class="btn btn-secondary">Cancel</a>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}
'''

# BI Analytics Template
BI_ANALYTICS_TEMPLATE = '''
{% extends base_template %}
{% block title %}BI Analytics{% endblock %}
{% block content %}
<div class="fade-in">
    <div class="row mb-4">
        <div class="col-12">
            <h2><i class="fas fa-chart-line"></i> Business Intelligence Analytics</h2>
            <p class="text-muted">Key Performance Indicators and Analytics Dashboard</p>
        </div>
    </div>

    <!-- KPI Cards -->
    <div class="row mb-4">
        <div class="col-md-3">
            <div class="card bg-primary text-white">
                <div class="card-body">
                    <h6>Total Revenue</h6>
                    <h3>KES {{ "{:,.0f}".format(total_revenue) }}</h3>
                    <i class="fas fa-dollar-sign fa-2x float-end"></i>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="card bg-danger text-white">
                <div class="card-body">
                    <h6>Total Cost</h6>
                    <h3>KES {{ "{:,.0f}".format(total_cost) }}</h3>
                    <i class="fas fa-chart-line fa-2x float-end"></i>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="card bg-success text-white">
                <div class="card-body">
                    <h6>Net Profit</h6>
                    <h3>KES {{ "{:,.0f}".format(profit) }}</h3>
                    <i class="fas fa-chart-line fa-2x float-end"></i>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="card bg-info text-white">
                <div class="card-body">
                    <h6>Completion Rate</h6>
                    <h3>{{ "{:.1f}".format(completion_rate) }}%</h3>
                    <i class="fas fa-check-circle fa-2x float-end"></i>
                </div>
            </div>
        </div>
    </div>

    <!-- Charts Row -->
    <div class="row mb-4">
        <div class="col-md-6">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-chart-line"></i> Monthly Profit Trend</h5>
                </div>
                <div class="card-body">
                    <canvas id="profitChart"></canvas>
                </div>
            </div>
        </div>
        <div class="col-md-6">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-chart-bar"></i> Vehicle Performance</h5>
                </div>
                <div class="card-body">
                    <canvas id="performanceChart"></canvas>
                </div>
            </div>
        </div>
    </div>

    <!-- Energy Analysis -->
    <div class="row mb-4">
        <div class="col-md-6">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-bolt"></i> Energy Consumption Analysis</h5>
                </div>
                <div class="card-body">
                    <div class="table-responsive">
                        <table class="table table-sm">
                            <thead>
                                <tr><th>Vehicle Model</th><th>Avg Energy (kWh)</th><th>Avg Distance (km)</th><th>Efficiency</th></tr>
                            </thead>
                            <tbody>
                                {% for data in energy_data %}
                                <tr>
                                    <td>{{ data.model }}</td>
                                    <td>{{ "{:.1f}".format(data.avg_energy or 0) }}</td>
                                    <td>{{ "{:.1f}".format(data.avg_distance or 0) }}</td>
                                    <td>{{ "{:.1f}".format((data.avg_distance or 0) / (data.avg_energy or 1) ) }} km/kWh</td>
                                </tr>
                                {% endfor %}
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>
        <div class="col-md-6">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-chart-pie"></i> Route Profitability</h5>
                </div>
                <div class="card-body">
                    <canvas id="routeChart"></canvas>
                </div>
            </div>
        </div>
    </div>

    <!-- Vehicle Performance Table -->
    <div class="card">
        <div class="card-header">
            <h5><i class="fas fa-trophy"></i> Vehicle Performance Rankings</h5>
        </div>
        <div class="card-body">
            <div class="table-responsive">
                <table class="table table-hover">
                    <thead>
                        <tr>
                            <th>Rank</th>
                            <th>Vehicle</th>
                            <th>Model</th>
                            <th>Trips</th>
                            <th>Revenue (KES)</th>
                            <th>Cost (KES)</th>
                            <th>Profit (KES)</th>
                            <th>Distance (km)</th>
                        </tr>
                    </thead>
                    <tbody>
                        {% for vehicle in vehicle_performance %}
                        <tr>
                            <td>{{ loop.index }}</td>
                            <td><strong>{{ vehicle.registration_number }}</strong></td>
                            <td>{{ vehicle.model }}</td>
                            <td>{{ vehicle.trips_completed }}</td>
                            <td>{{ "{:,.0f}".format(vehicle.total_revenue or 0) }}</td>
                            <td>{{ "{:,.0f}".format(vehicle.total_cost or 0) }}</td>
                            <td class="text-success">{{ "{:,.0f}".format((vehicle.total_revenue or 0) - (vehicle.total_cost or 0)) }}</td>
                            <td>{{ "{:,.0f}".format(vehicle.total_distance or 0) }}</td>
                        </tr>
                        {% endfor %}
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</div>

<script>
// Profit Chart
const profitCtx = document.getElementById('profitChart').getContext('2d');
new Chart(profitCtx, {
    type: 'line',
    data: {
        labels: [{% for month in monthly_trend %}'{{ month.month }}',{% endfor %}],
        datasets: [{
            label: 'Profit (KES)',
            data: [{% for p in profit_trend %}{{ p[1] }},{% endfor %}],
            borderColor: '#28a745',
            backgroundColor: 'rgba(40,167,69,0.1)',
            tension: 0.4,
            fill: true
        }]
    }
});

// Performance Chart
const perfCtx = document.getElementById('performanceChart').getContext('2d');
new Chart(perfCtx, {
    type: 'bar',
    data: {
        labels: [{% for vehicle in vehicle_performance[:5] %}'{{ vehicle.registration_number }}',{% endfor %}],
        datasets: [{
            label: 'Revenue (KES)',
            data: [{% for vehicle in vehicle_performance[:5] %}{{ vehicle.total_revenue or 0 }},{% endfor %}],
            backgroundColor: '#667eea'
        }]
    }
});

// Route Chart
const routeCtx = document.getElementById('routeChart').getContext('2d');
new Chart(routeCtx, {
    type: 'doughnut',
    data: {
        labels: [{% for analysis in cost_analysis %}'{{ analysis.route_name }}',{% endfor %}],
        datasets: [{
            data: [{% for analysis in cost_analysis %}{{ (analysis.avg_revenue or 0) - (analysis.avg_cost or 0) }},{% endfor %}],
            backgroundColor: ['#667eea', '#764ba2', '#f093fb', '#4facfe', '#43e97b']
        }]
    }
});
</script>
{% endblock %}
'''

# Reports Template
REPORTS_TEMPLATE = '''
{% extends base_template %}
{% block title %}Reports{% endblock %}
{% block content %}
<div class="fade-in">
    <div class="row mb-4">
        <div class="col-12">
            <h2><i class="fas fa-file-alt"></i> Report Generation</h2>
        </div>
    </div>

    <div class="row">
        <div class="col-md-4">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-cog"></i> Generate Report</h5>
                </div>
                <div class="card-body">
                    <form method="POST" action="{{ url_for('generate_report') }}">
                        <div class="mb-3">
                            <label>Report Type *</label>
                            <select name="report_type" class="form-control" required>
                                <option value="Daily">Daily Report</option>
                                <option value="Weekly">Weekly Report</option>
                                <option value="Monthly">Monthly Report</option>
                                <option value="Custom">Custom Range</option>
                            </select>
                        </div>
                        <div class="mb-3">
                            <label>From Date</label>
                            <input type="date" name="date_from" class="form-control">
                        </div>
                        <div class="mb-3">
                            <label>To Date</label>
                            <input type="date" name="date_to" class="form-control">
                        </div>
                        <button type="submit" class="btn btn-primary w-100">
                            <i class="fas fa-download"></i> Generate Report
                        </button>
                    </form>
                </div>
            </div>
        </div>
        
        <div class="col-md-8">
            <div class="card">
                <div class="card-header">
                    <h5><i class="fas fa-history"></i> Generated Reports</h5>
                </div>
                <div class="card-body">
                    <div class="list-group">
                        {% for report in reports %}
                        <a href="{{ url_for('view_report', report_id=report.id) }}" class="list-group-item list-group-item-action">
                            <div class="d-flex justify-content-between align-items-center">
                                <div>
                                    <i class="fas fa-file-pdf"></i>
                                    <strong>{{ report.report_name }}</strong>
                                    <br>
                                    <small class="text-muted">Type: {{ report.report_type }} | Generated: {{ report.generated_at.strftime('%Y-%m-%d %H:%M') }}</small>
                                </div>
                                <span class="badge bg-primary">View</span>
                            </div>
                        </a>
                        {% else %}
                        <p class="text-muted">No reports generated yet.</p>
                        {% endfor %}
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
'''

# View Report Template
VIEW_REPORT_TEMPLATE = '''
{% extends base_template %}
{% block title %}View Report{% endblock %}
{% block content %}
<div class="fade-in">
    <div class="row mb-4">
        <div class="col-12">
            <h2><i class="fas fa-file-alt"></i> {{ report.report_name }}</h2>
            <p>Generated on: {{ report.generated_at.strftime('%Y-%m-%d %H:%M:%S') }}</p>
        </div>
    </div>

    <div class="card">
        <div class="card-body">
            <div class="table-responsive">
                <table class="table table-striped">
                    <thead>
                        <tr>
                            <th>Date</th>
                            <th>Driver</th>
                            <th>Vehicle</th>
                            <th>Route</th>
                            <th>Distance (km)</th>
                            <th>Energy (kWh)</th>
                            <th>Revenue (KES)</th>
                            <th>Cost (KES)</th>
                            <th>Profit (KES)</th>
                            <th>Status</th>
                        </tr>
                    </thead>
                    <tbody>
                        {% for row in data %}
                        <tr>
                            <td>{{ row.Date }}</td>
                            <td>{{ row.Driver }}</td>
                            <td>{{ row.Vehicle }}</td>
                            <td>{{ row.Route }}</td>
                            <td>{{ row['Distance (km)'] }}</td>
                            <td>{{ row['Energy (kWh)'] }}</td>
                            <td>{{ row['Revenue (KES)'] }}</td>
                            <td>{{ row['Cost (KES)'] }}</td>
                            <td class="text-success">{{ row['Profit (KES)'] }}</td>
                            <td><span class="badge bg-success">{{ row.Status }}</span></td>
                        </tr>
                        {% endfor %}
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</div>
{% endblock %}
'''

# ==================== MAIN EXECUTION ====================

if __name__ == '__main__':
    with app.app_context():
        init_database()
    
    print("\n" + "="*60)
    print("E-MOBILITY LOGISTICS BI SYSTEM - KENYA")
    print("="*60)
    print("Server running at: http://localhost:5000")
    print("\nDemo Credentials:")
    print("  Admin    - username: admin    | password: Admin123!")
    print("  Manager  - username: manager1 | password: Admin123!")
    print("  User     - username: user1    | password: User123!")
    print("\nSystem Features:")
    print("  ✅ Authentication & Role Management")
    print("  ✅ Payment Workflow (Create → Authorize → Commit → Process)")
    print("  ✅ Vehicle & Fleet Management")
    print("  ✅ Route Planning & Delivery Tracking")
    print("  ✅ BI Analytics & KPIs")
    print("  ✅ Report Generation")
    print("="*60)
    print("\nPress CTRL+C to stop the server\n")
    
    app.run(debug=True, host='0.0.0.0', port=5000)

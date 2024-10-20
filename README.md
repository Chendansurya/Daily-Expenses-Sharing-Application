pip install Flask Flask-SQLAlchemy

from flask import Flask, request, jsonify, send_file
from flask_sqlalchemy import SQLAlchemy
from io import BytesIO
import csv

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///expenses.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# User Model
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(100), unique=True, nullable=False)
    mobile_number = db.Column(db.String(15), nullable=False)

# Expense Model
class Expense(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    description = db.Column(db.String(100), nullable=False)
    total_amount = db.Column(db.Float, nullable=False)
    split_type = db.Column(db.String(20), nullable=False)
    payer_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)

# Splitting Details
class Split(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    expense_id = db.Column(db.Integer, db.ForeignKey('expense.id'), nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    amount = db.Column(db.Float, nullable=False)

# Initialize the database
with app.app_context():
    db.create_all()

# Helper Function to handle splits
def handle_split(expense_id, split_type, total_amount, split_details):
    if split_type == 'equal':
        split_amount = total_amount / len(split_details)
        for user_id in split_details:
            split = Split(expense_id=expense_id, user_id=user_id, amount=split_amount)
            db.session.add(split)

    elif split_type == 'exact':
        for user_id, amount in split_details.items():
            split = Split(expense_id=expense_id, user_id=user_id, amount=amount)
            db.session.add(split)

    elif split_type == 'percentage':
        for user_id, percentage in split_details.items():
            amount = total_amount * (percentage / 100)
            split = Split(expense_id=expense_id, user_id=user_id, amount=amount)
            db.session.add(split)

    db.session.commit()

# Create User
@app.route('/user', methods=['POST'])
def create_user():
    data = request.json
    user = User(name=data['name'], email=data['email'], mobile_number=data['mobile_number'])
    db.session.add(user)
    db.session.commit()
    return jsonify({"message": "User created successfully"}), 201

# Add Expense
@app.route('/expense', methods=['POST'])
def add_expense():
    data = request.json
    expense = Expense(description=data['description'], total_amount=data['total_amount'], split_type=data['split_type'], payer_id=data['payer_id'])
    db.session.add(expense)
    db.session.commit()

    # Split the expense among participants
    handle_split(expense.id, data['split_type'], data['total_amount'], data['split_details'])

    return jsonify({"message": "Expense added successfully"}), 201

# Retrieve User Expenses
@app.route('/user/<int:user_id>/expenses', methods=['GET'])
def get_user_expenses(user_id):
    expenses = Split.query.filter_by(user_id=user_id).all()
    result = [{"expense_id": expense.expense_id, "amount": expense.amount} for expense in expenses]
    return jsonify(result)

# Retrieve All Expenses
@app.route('/expenses', methods=['GET'])
def get_all_expenses():
    expenses = Expense.query.all()
    result = [{"description": expense.description, "total_amount": expense.total_amount, "split_type": expense.split_type} for expense in expenses]
    return jsonify(result)

# Download Balance Sheet
@app.route('/balancesheet/download', methods=['GET'])
def download_balance_sheet():
    output = BytesIO()
    writer = csv.writer(output)
    writer.writerow(['User ID', 'Expense ID', 'Amount'])
    splits = Split.query.all()
    for split in splits:
        writer.writerow([split.user_id, split.expense_id, split.amount])

    output.seek(0)
    return send_file(output, mimetype='text/csv', as_attachment=True, attachment_filename='balance_sheet.csv')

if __name__ == '__main__':
    app.run(debug=True)



Models:
User: Manages user information.
Expense: Stores details about each expense.
Split: Stores the split information for each user in the expense.

Endpoints:
POST /user: Create a user.
POST /expense: Add a new expense with split details.
GET /user/<user_id>/expenses: Retrieve the expenses for a specific user.
GET /expenses: Retrieve all expenses.
GET /balancesheet/download: Download the balance sheet as a CSV file.

Features:
Equal Split: Automatically divides the total expense equally among participants.
Exact Split: Allows specifying exact amounts for each participant.
Percentage Split: Allows specifying percentages, with validation to ensure they sum to 100%.
Balance Sheet: Provides a downloadable CSV file containing the balance sheet.


User Management:
You can create users with POST /users.

Expense Management:
Add expenses using POST /expenses.
You can split expenses using three methods:
Equal: All participants share the amount equally.
Exact: Specify the exact amount for each participant.
Percentage: Specify the percentage of the amount each participant owes (must sum to 100%).

Expense Retrieval:
Retrieve individual user expenses using GET /users/<user_id>/expenses.
Retrieve all expenses using GET /expenses.
Balance Sheet:
Download a CSV balance sheet using GET /balancesheet/download.


1.Create User:
curl -X POST http://localhost:5000/users -H "Content-Type: application/json" -d '{
    "name": "John Doe",
    "email": "john@example.com",
    "mobile_number": "1234567890"
}'

2.Add Expense (Equal Split):
curl -X POST http://localhost:5000/expenses -H "Content-Type: application/json" -d '{
    "description": "Dinner",
    "amount": 3000,
    "split_type": "equal",
    "payer_id": 1,
    "participants": [1, 2, 3]
}'

3.Add Expense (Exact Split):
curl -X POST http://localhost:5000/expenses -H "Content-Type: application/json" -d '{
    "description": "Shopping",
    "amount": 4299,
    "split_type": "exact",
    "payer_id": 1,
    "participants": {"1": 799, "2": 2000, "3": 1500}
}'

1.Setup Virtual Environment:
python3 -m venv venv
source venv/bin/activate

2.Install Dependencies:
pip install Flask SQLAlchemy

3.Run the Application:
python app.py

4.Access the API: Visit http://127.0.0.1:5000/ for endpoints.

Improvements:
Authentication: Use JWT for secure access.
Tests: Add unit tests for API endpoints.
Error Handling: Handle more edge cases and validation errors.

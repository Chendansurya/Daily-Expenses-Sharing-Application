from flask import Flask, request, jsonify
from collections import defaultdict
import pandas as pd

app = Flask(__name__)

# Sample database structures (can use SQLAlchemy for actual database)
users = {}
expenses = defaultdict(list)

# Create a user
@app.route('/user/create', methods=['POST'])
def create_user():
    data = request.json
    user_id = len(users) + 1
    users[user_id] = {
        "name": data['name'],
        "email": data['email'],
        "mobile": data['mobile']
    }
    return jsonify({"message": "User created", "user_id": user_id}), 201

# Add an expense
@app.route('/expense/add', methods=['POST'])
def add_expense():
    data = request.json
    user_id = data['user_id']
    method = data['split_method']
    total_amount = data['amount']
    participants = data['participants']
    
    if method == 'equal':
        per_person = total_amount / len(participants)
        for p in participants:
            expenses[p].append({'amount': per_person, 'description': data['description']})
    elif method == 'exact':
        amounts = data['amounts']
        for p, amt in zip(participants, amounts):
            expenses[p].append({'amount': amt, 'description': data['description']})
    elif method == 'percentage':
        percentages = data['percentages']
        if sum(percentages) != 100:
            return jsonify({"error": "Percentages do not add up to 100%"}), 400
        for p, percent in zip(participants, percentages):
            amt = total_amount * (percent / 100)
            expenses[p].append({'amount': amt, 'description': data['description']})

    return jsonify({"message": "Expense added"}), 201

# Retrieve individual expenses
@app.route('/expense/user/<int:user_id>', methods=['GET'])
def get_user_expenses(user_id):
    user_expenses = expenses[user_id]
    return jsonify(user_expenses)

# Download balance sheet
@app.route('/balance/download', methods=['GET'])
def download_balance():
    df = pd.DataFrame(expenses)
    df.to_csv('balance_sheet.csv')
    return jsonify({"message": "Balance sheet downloaded"})

if __name__ == '__main__':
    app.run(debug=True)

# Heet-s-AI
this is a AI to use all work and give to all answer the question.
pip install flask openai email-validator flask_sqlalchemy

from flask import Flask, request, jsonify
from email_validator import validate_email, EmailNotValidError
from flask_sqlalchemy import SQLAlchemy
import openai

app = Flask(__name__)

# Database configuration for users
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///users.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# Set OpenAI API Key
openai.api_key = "your_openai_api_key_here"

# User model
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(120), unique=True, nullable=False)
    is_premium = db.Column(db.Boolean, default=False)

# Initialize the database
with app.app_context():
    db.create_all()

# Route for email validation and user registration
@app.route('/register', methods=['POST'])
def register():
    data = request.json
    email = data.get('email')
    is_premium = data.get('is_premium', False)

    # Validate email
    try:
        valid = validate_email(email)
        email = valid.email
    except EmailNotValidError as e:
        return jsonify({'error': str(e)}), 400

    # Check if the user is already registered
    existing_user = User.query.filter_by(email=email).first()
    if existing_user:
        return jsonify({'message': 'User already registered.'}), 200

    # Add new user to the database
    new_user = User(email=email, is_premium=is_premium)
    db.session.add(new_user)
    db.session.commit()

    return jsonify({'message': 'User registered successfully.'}), 201

# Route to handle question-answering
@app.route('/ask', methods=['POST'])
def ask():
    data = request.json
    email = data.get('email')
    question = data.get('question')

    # Check if the email is registered
    user = User.query.filter_by(email=email).first()
    if not user:
        return jsonify({'error': 'User not found. Please register first.'}), 404

    # Premium user check
    if not user.is_premium:
        return jsonify({'error': 'Access restricted to premium users.'}), 403

    # Use OpenAI API to get the answer
    try:
        response = openai.Completion.create(
            engine="text-davinci-003",
            prompt=question,
            max_tokens=200
        )
        answer = response['choices'][0]['text'].strip()
        return jsonify({'question': question, 'answer': answer}), 200
    except Exception as e:
        return jsonify({'error': str(e)}), 500

# Run the app
if __name__ == '__main__':
    app.run(debug=True)

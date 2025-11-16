Agent.edu
TO help to learn and educate poor children
### Problem Statement-- the problem you're trying to solve, and why you think it's an important or interesting problem to solve
### Why agents? -- Why are agents the right solution to this problem
### What you created -- What's the overall architecture?
### Demo -- Show your Solution
### The Build -- How you created it, what tools ot technologies you used.
### If i had more time, this is what i'd do

# app.py
"""
Education AI prototype for underprivileged children.
Single-file Flask app with an adaptive quiz engine and progress tracking.

Run:
    pip install flask flask_sqlalchemy werkzeug
    python app.py

Open http://127.0.0.1:5000 in your browser.

This is a simple prototype for demonstration and teaching purposes.
"""

from flask import Flask, request, redirect, url_for, render_template_string, session, flash
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash
import random
import datetime

app = Flask(_name_)
app.config['SECRET_KEY'] = 'change-this-secret-in-production'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///eduai.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

### Database models

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100))
    username = db.Column(db.String(80), unique=True, nullable=False)
    password_hash = db.Column(db.String(200), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.datetime.utcnow)

    def check_password(self, pw):
        return check_password_hash(self.password_hash, pw)

class Progress(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    topic = db.Column(db.String(80), nullable=False)
    correct = db.Column(db.Integer, default=0)
    total = db.Column(db.Integer, default=0)
    last_attempt = db.Column(db.DateTime, default=datetime.datetime.utcnow)

    _table_args_ = (db.UniqueConstraint('user_id', 'topic', name='_user_topic_uc'),)

class Attempt(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    question_id = db.Column(db.String(80), nullable=False)  # id from question bank
    chosen = db.Column(db.String(200))
    correct = db.Column(db.Boolean)
    timestamp = db.Column(db.DateTime, default=datetime.datetime.utcnow)


### A small in-memory question bank (could be moved to files or DB)
# Each question has id, topic, difficulty ('easy','medium','hard'), text, options, answer (index)
QUESTION_BANK = [
    # Reading - easy
    {"id": "r1", "topic": "reading", "difficulty": "easy",
     "text": "Which word is a noun? 'cat', 'run', or 'quickly'?", "options": ["cat", "run", "quickly"], "answer": 0},
    {"id": "r2", "topic": "reading", "difficulty": "easy",
     "text": "Which word means the opposite of 'big'?", "options": ["huge", "small", "tall"], "answer": 1},

    # Reading - medium
    {"id": "r3", "topic": "reading", "difficulty": "medium",
     "text": "Choose the best word to complete: 'She ___ to school every day.'", "options": ["go", "goes", "going"], "answer": 1},
    {"id": "r4", "topic": "reading", "difficulty": "medium",
     "text": "Find the adjective: 'The red ball bounced high.'", "options": ["red", "ball", "bounced"], "answer": 0},

    # Reading - hard
    {"id": "r5", "topic": "reading", "difficulty": "hard",
     "text": "Identify the subject and predicate in: 'The quick brown fox jumps over the fence.'",
     "options": ["subject: fox; predicate: jumps over the fence", "subject: jumps; predicate: the quick brown fox", "subject: fence; predicate: over the fox"], "answer": 0},

    # Math - easy
    {"id": "m1", "topic": "math", "difficulty": "easy",
     "text": "What is 2 + 3?", "options": ["4", "5", "6"], "answer": 1},
    {"id": "m2", "topic": "math", "difficulty": "easy",
     "text": "What number comes after 9?", "options": ["8", "10", "11"], "answer": 1},

    # Math - medium
    {"id": "m3", "topic": "math", "difficulty": "medium",
     "text": "What is 7 x 6?", "options": ["42", "36", "48"], "answer": 0},
    {"id": "m4", "topic": "math", "difficulty": "medium",
     "text": "If you have 10 apples and give 3 away, how many left?", "options": ["6", "7", "8"], "answer": 1},

    # Math - hard
    {"id": "m5", "topic": "math", "difficulty": "hard",
     "text": "What is (12 ÷ 3) + (4 x 2)?", "options": ["12", "14", "10"], "answer": 1},

    # General knowledge / science - easy
    {"id": "g1", "topic": "science", "difficulty": "easy",
     "text": "What do plants need to make food?", "options": ["sunlight", "music", "carrots"], "answer": 0},
    {"id": "g2", "topic": "science", "difficulty": "easy",
     "text": "Which is a flying animal?", "options": ["fish", "bird", "elephant"], "answer": 1},

    # Science - medium
    {"id": "g3", "topic": "science", "difficulty": "medium",
     "text": "Water turns to steam when it is ___", "options": ["frozen", "boiled", "cut"], "answer": 1},

    # Add more questions as needed...
]

TOPICS = sorted(list({q['topic'] for q in QUESTION_BANK}))


### Helper functions

def seed_db():
    """Create tables and a demo user (if not exists)."""
    db.create_all()
    if not User.query.filter_by(username='demo').first():
        demo = User(name='Demo Kid', username='demo',
                    password_hash=generate_password_hash('demo'))
        db.session.add(demo)
        db.session.commit()
    # seed progress rows for demo user
    demo = User.query.filter_by(username='demo').first()
    for t in TOPICS:
        p = Progress.query.filter_by(user_id=demo.id, topic=t).first()
        if not p:
            p = Progress(user_id=demo.id, topic=t, correct=0, total=0)
            db.session.add(p)
    db.session.commit()

def get_user():
    uid = session.get('user_id')
    if not uid:
        return None
    return User.query.get(uid)

def get_mastery(user_id, topic):
    """Return mastery score in [0,1] for a given user and topic.
    If no attempts, return 0.5 (neutral)."""
    p = Progress.query.filter_by(user_id=user_id, topic=topic).first()
    if not p or p.total == 0:
        return 0.5
    return p.correct / p.total

def choose_question_for(user_id, topic):
    """Adaptive selection:
       - if mastery < 0.4 -> choose easy
       - 0.4 <= mastery < 0.75 -> choose medium
       - >=0.75 -> choose hard
       Randomly pick a question not seen recently.
    """
    mastery = get_mastery(user_id, topic)
    if mastery < 0.4:
        desired = 'easy'
    elif mastery < 0.75:
        desired = 'medium'
    else:
        desired = 'hard'
    # filter bank
    candidates = [q for q in QUESTION_BANK if q['topic'] == topic and q['difficulty'] == desired]
    if not candidates:
        # fallback across difficulties
        candidates = [q for q in QUESTION_BANK if q['topic'] == topic]
    if not candidates:
        return None
    # avoid repeating the same question seen in last 5 attempts
    recent_qids = [a.question_id for a in Attempt.query.filter_by(user_id=user_id).order_by(Attempt.timestamp.desc()).limit(10).all()]
    filtered = [q for q in candidates if q['id'] not in recent_qids]
    if filtered:
        candidates = filtered
    return random.choice(candidates)

def record_attempt(user_id, question_id, chosen_index, correct):
    a = Attempt(user_id=user_id, question_id=question_id, chosen=str(chosen_index), correct=bool(correct))
    db.session.add(a)
    # update progress by topic
    q = next((item for item in QUESTION_BANK if item["id"] == question_id), None)
    if q:
        p = Progress.query.filter_by(user_id=user_id, topic=q['topic']).first()
        if not p:
            p = Progress(user_id=user_id, topic=q['topic'], correct=0, total=0)
            db.session.add(p)
        if correct:
            p.correct += 1
        p.total += 1
        p.last_attempt = datetime.datetime.utcnow()
    db.session.commit()

def get_recommendation(user_id):
    """Return simple recommendations per topic."""
    recs = []
    for t in TOPICS:
        mastery = get_mastery(user_id, t)
        if mastery < 0.4:
            recs.append((t, mastery, 'Remedial: focus on basics and simple practice.'))
        elif mastery < 0.75:
            recs.append((t, mastery, 'Practice: keep practicing to build confidence.'))
        else:
            recs.append((t, mastery, 'Advance: try harder problems and challenge tasks.'))
    return recs

### Routes and templates

NAV = """
<nav style="background:#f0f8ff;padding:8px;border-radius:6px;margin-bottom:12px;">
  <a href="/">Home</a> |
  {% if user %}<a href="{{ url_for('dashboard') }}">Dashboard</a> | <a href="{{ url_for('logout') }}">Logout</a>{% else %}<a href="{{ url_for('login') }}">Login</a> | <a href="{{ url_for('register') }}">Sign up</a>{% endif %}
</nav>
"""

INDEX_TPL = NAV + """
<h1>Education AI — Learning for Every Child</h1>
<p>Welcome! This prototype adapts question difficulty to the learner's level and tracks progress.</p>
{% if user %}
  <p>Hello, <strong>{{ user.name }}</strong> — pick a topic to start:</p>
  <ul>
  {% for t in topics %}
    <li><a href="{{ url_for('start_quiz', topic=t) }}">{{ t.title() }}</a></li>
  {% endfor %}
  </ul>
  <p><a href="{{ url_for('dashboard') }}">View your dashboard & progress</a></p>
{% else %}
  <p><a href="{{ url_for('login') }}">Login</a> or <a href="{{ url_for('register') }}">Sign up</a> to begin.</p>
  <p>Try demo user: <strong>demo</strong> / password <strong>demo</strong></p>
{% endif %}
"""

REGISTER_TPL = NAV + """
<h2>Sign up</h2>
<form method="post">
  Full name:<br><input name="name" required><br>
  Username:<br><input name="username" required><br>
  Password:<br><input name="password" type="password" required><br><br>
  <button type="submit">Create account</button>
</form>
"""

LOGIN_TPL = NAV + """
<h2>Login</h2>
<form method="post">
  Username:<br><input name="username" required><br>
  Password:<br><input name="password" type="password" required><br><br>
  <button type="submit">Login</button>
</form>
"""

QUIZ_TPL = NAV + """
<h2>Topic: {{ topic.title() }}</h2>
<p>Difficulty chosen: <strong>{{ q.difficulty }}</strong></p>
<form method="post" action="{{ url_for('submit_answer') }}">
  <p><strong>{{ q.text }}</strong></p>
  {% for idx,opt in enumerate(q.options) %}
    <div><label><input type="radio" name="choice" value="{{ idx }}" required> {{ opt }}</label></div>
  {% endfor %}
  <input type="hidden" name="question_id" value="{{ q.id }}">
  <br><button type="submit">Submit Answer</button>
</form>
"""

RESULT_TPL = NAV + """
<h2>Result</h2>
<p>{{ msg }}</p>
<p><a href="{{ url_for('start_quiz', topic=topic) }}">Next question in {{ topic.title() }}</a></p>
<p><a href="{{ url_for('dashboard') }}">View Dashboard</a></p>
"""

DASH_TPL = NAV + """
<h2>Your Dashboard</h2>
<p>Name: <strong>{{ user.name }}</strong></p>
<table border="1" cellpadding="6" cellspacing="0">
  <tr><th>Topic</th><th>Correct</th><th>Total</th><th>Mastery</th><th>Recommendation</th></tr>
  {% for t, correct, total, mastery, note in rows %}
    <tr>
      <td>{{ t.title() }}</td>
      <td style="text-align:center">{{ correct }}</td>
      <td style="text-align:center">{{ total }}</td>
      <td style="text-align:center">{{ '{:.0%}'.format(mastery) }}</td>
      <td>{{ note }}</td>
    </tr>
  {% endfor %}
</table>
<br><h3>Recent Attempts</h3>
<ul>
  {% for a in attempts %}
    <li>{{ a.timestamp.strftime('%Y-%m-%d %H:%M') }} — Q:{{ a.question_id }} — Chosen:{{ a.chosen }} — Correct:{{ 'Yes' if a.correct else 'No' }}</li>
  {% endfor %}
</ul>
"""

@app.route('/')
def index():
    user = get_user()
    return render_template_string(INDEX_TPL, user=user, topics=TOPICS)

@app.route('/register', methods=['GET','POST'])
def register():
    if request.method == 'POST':
        name = request.form['name'].strip()
        username = request.form['username'].strip()
        password = request.form['password']
        if User.query.filter_by(username=username).first():
            flash('Username exists — choose another.')
            return redirect(url_for('register'))
        u = User(name=name, username=username, password_hash=generate_password_hash(password))
        db.session.add(u)
        db.session.commit()
        # seed progress rows
        for t in TOPICS:
            p = Progress(user_id=u.id, topic=t, correct=0, total=0)
            db.session.add(p)
        db.session.commit()
        flash('Account created. Please log in.')
        return redirect(url_for('login'))
    return render_template_string(REGISTER_TPL)

@app.route('/login', methods=['GET','POST'])
def login():
    if request.method == 'POST':
        username = request.form['username'].strip()
        password = request.form['password']
        user = User.query.filter_by(username=username).first()
        if not user or not user.check_password(password):
            flash('Invalid credentials.')
            return redirect(url_for('login'))
        session['user_id'] = user.id
        flash('Logged in.')
        return redirect(url_for('index'))
    return render_template_string(LOGIN_TPL)

@app.route('/logout')
def logout():
    session.pop('user_id', None)
    flash('Logged out.')
    return redirect(url_for('index'))

@app.route('/start_quiz/<topic>')
def start_quiz(topic):
    user = get_user()
    if not user:
        flash('Please log in first.')
        return redirect(url_for('login'))
    if topic not in TOPICS:
        flash('Unknown topic.')
        return redirect(url_for('index'))
    q = choose_question_for(user.id, topic)
    if not q:
        flash('No questions available for this topic.')
        return redirect(url_for('index'))
    return render_template_string(QUIZ_TPL, user=user, q=q, topic=topic)

@app.route('/submit_answer', methods=['POST'])
def submit_answer():
    user = get_user()
    if not user:
        flash('Please log in first.')
        return redirect(url_for('login'))
    qid = request.form.get('question_id')
    choice = request.form.get('choice')
    if qid is None or choice is None:
        flash('Invalid submission.')
        return redirect(url_for('index'))
    try:
        chosen_index = int(choice)
    except:
        chosen_index = None
    q = next((item for item in QUESTION_BANK if item["id"] == qid), None)
    if not q:
        flash('Question not found.')
        return redirect(url_for('index'))
    correct = (chosen_index == q['answer'])
    record_attempt(user.id, qid, chosen_index, correct)
    msg = "Correct! Well done." if correct else f"Not correct. The right answer was: {q['options'][q['answer']]}."
    return render_template_string(RESULT_TPL, msg=msg, topic=q['topic'])

@app.route('/dashboard')
def dashboard():
    user = get_user()
    if not user:
        flash('Please log in first.')
        return redirect(url_for('login'))
    rows = []
    for t in TOPICS:
        p = Progress.query.filter_by(user_id=user.id, topic=t).first()
        if not p:
            correct = 0; total = 0; mastery = 0.5; note = 'No data'
        else:
            correct = p.correct; total = p.total
            mastery = (correct/total) if total>0 else 0.5
            if mastery < 0.4:
                note = 'Remedial focus'
            elif mastery < 0.75:
                note = 'Keep practicing'
            else:
                note = 'Ready for advanced'
        rows.append((t, correct, total, mastery, note))
    attempts = Attempt.query.filter_by(user_id=user.id).order_by(Attempt.timestamp.desc()).limit(10).all()
    return render_template_string(DASH_TPL, user=user, rows=rows, attempts=attempts)

# Utility route to view question bank (for teacher/admin)
@app.route('/_questions')
def show_questions():
    # very small admin view
    lines = []
    for q in QUESTION_BANK:
        lines.append(f"{q['id']}: [{q['topic']}/{q['difficulty']}] {q['text']} -- options={q['options']} answer={q['answer']}")
    return "<pre>" + "\n".join(lines) + "</pre>"

if _name_ == '_main_':
    seed_db()
    app.run(debug=True)

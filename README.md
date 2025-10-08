# student-management-system
The Student Management System is a simple app to manage student data efficiently. It allows adding, updating, viewing, and deleting records, tracking courses and grades, and maintaining organized student information with a user-friendly interface and secure database integration.

from flask import Flask, render_template_string, request, redirect, url_for, flash
from datetime import datetime

app = Flask(__name__)
app.secret_key = 'your_secret_key_here'

# In-memory storage for students
students = {}
next_id = 1

# HTML Templates
BASE_TEMPLATE = '''
<!DOCTYPE html>
<html>
<head>
    <title>Student Management System</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 1200px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f5f5f5;
        }
        h1, h2 {
            color: #333;
        }
        .container {
            background-color: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            margin-bottom: 20px;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        th, td {
            padding: 12px;
            text-align: left;
            border-bottom: 1px solid #ddd;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        tr:hover {
            background-color: #f5f5f5;
        }
        .btn {
            padding: 8px 16px;
            margin: 5px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            text-decoration: none;
            display: inline-block;
        }
        .btn-primary {
            background-color: #4CAF50;
            color: white;
        }
        .btn-edit {
            background-color: #2196F3;
            color: white;
        }
        .btn-delete {
            background-color: #f44336;
            color: white;
        }
        .btn:hover {
            opacity: 0.8;
        }
        .form-group {
            margin-bottom: 15px;
        }
        label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
        }
        input, select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
            box-sizing: border-box;
        }
        .alert {
            padding: 15px;
            margin-bottom: 20px;
            border-radius: 4px;
        }
        .alert-success {
            background-color: #d4edda;
            color: #155724;
            border: 1px solid #c3e6cb;
        }
        .alert-error {
            background-color: #f8d7da;
            color: #721c24;
            border: 1px solid #f5c6cb;
        }
    </style>
</head>
<body>
    <h1>🎓 Student Management System</h1>
    {% with messages = get_flashed_messages(with_categories=true) %}
        {% if messages %}
            {% for category, message in messages %}
                <div class="alert alert-{{ category }}">{{ message }}</div>
            {% endfor %}
        {% endif %}
    {% endwith %}
    {% block content %}{% endblock %}
</body>
</html>
'''

HOME_TEMPLATE = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', '''
    <div class="container">
        <h2>All Students</h2>
        <a href="{{ url_for('add_student') }}" class="btn btn-primary">Add New Student</a>
        
        {% if students %}
        <table>
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>Email</th>
                    <th>Course</th>
                    <th>Year</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                {% for id, student in students.items() %}
                <tr>
                    <td>{{ id }}</td>
                    <td>{{ student.name }}</td>
                    <td>{{ student.email }}</td>
                    <td>{{ student.course }}</td>
                    <td>{{ student.year }}</td>
                    <td>
                        <a href="{{ url_for('view_student', student_id=id) }}" class="btn btn-primary">View</a>
                        <a href="{{ url_for('edit_student', student_id=id) }}" class="btn btn-edit">Edit</a>
                        <a href="{{ url_for('delete_student', student_id=id) }}" class="btn btn-delete" 
                           onclick="return confirm('Are you sure you want to delete this student?')">Delete</a>
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
        {% else %}
        <p>No students found. <a href="{{ url_for('add_student') }}">Add the first student</a></p>
        {% endif %}
    </div>
''')

ADD_STUDENT_TEMPLATE = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', '''
    <div class="container">
        <h2>Add New Student</h2>
        <form method="POST">
            <div class="form-group">
                <label for="name">Name:</label>
                <input type="text" id="name" name="name" required>
            </div>
            <div class="form-group">
                <label for="email">Email:</label>
                <input type="email" id="email" name="email" required>
            </div>
            <div class="form-group">
                <label for="course">Course:</label>
                <input type="text" id="course" name="course" required>
            </div>
            <div class="form-group">
                <label for="year">Year:</label>
                <select id="year" name="year" required>
                    <option value="1">1st Year</option>
                    <option value="2">2nd Year</option>
                    <option value="3">3rd Year</option>
                    <option value="4">4th Year</option>
                </select>
            </div>
            <div class="form-group">
                <label for="phone">Phone:</label>
                <input type="tel" id="phone" name="phone">
            </div>
            <button type="submit" class="btn btn-primary">Add Student</button>
            <a href="{{ url_for('home') }}" class="btn">Cancel</a>
        </form>
    </div>
''')

EDIT_STUDENT_TEMPLATE = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', '''
    <div class="container">
        <h2>Edit Student</h2>
        <form method="POST">
            <div class="form-group">
                <label for="name">Name:</label>
                <input type="text" id="name" name="name" value="{{ student.name }}" required>
            </div>
            <div class="form-group">
                <label for="email">Email:</label>
                <input type="email" id="email" name="email" value="{{ student.email }}" required>
            </div>
            <div class="form-group">
                <label for="course">Course:</label>
                <input type="text" id="course" name="course" value="{{ student.course }}" required>
            </div>
            <div class="form-group">
                <label for="year">Year:</label>
                <select id="year" name="year" required>
                    <option value="1" {% if student.year == "1" %}selected{% endif %}>1st Year</option>
                    <option value="2" {% if student.year == "2" %}selected{% endif %}>2nd Year</option>
                    <option value="3" {% if student.year == "3" %}selected{% endif %}>3rd Year</option>
                    <option value="4" {% if student.year == "4" %}selected{% endif %}>4th Year</option>
                </select>
            </div>
            <div class="form-group">
                <label for="phone">Phone:</label>
                <input type="tel" id="phone" name="phone" value="{{ student.phone or '' }}">
            </div>
            <button type="submit" class="btn btn-primary">Update Student</button>
            <a href="{{ url_for('home') }}" class="btn">Cancel</a>
        </form>
    </div>
''')

VIEW_STUDENT_TEMPLATE = BASE_TEMPLATE.replace('{% block content %}{% endblock %}', '''
    <div class="container">
        <h2>Student Details</h2>
        <table>
            <tr>
                <th>Field</th>
                <th>Value</th>
            </tr>
            <tr>
                <td><strong>ID</strong></td>
                <td>{{ student_id }}</td>
            </tr>
            <tr>
                <td><strong>Name</strong></td>
                <td>{{ student.name }}</td>
            </tr>
            <tr>
                <td><strong>Email</strong></td>
                <td>{{ student.email }}</td>
            </tr>
            <tr>
                <td><strong>Course</strong></td>
                <td>{{ student.course }}</td>
            </tr>
            <tr>
                <td><strong>Year</strong></td>
                <td>{{ student.year }}</td>
            </tr>
            <tr>
                <td><strong>Phone</strong></td>
                <td>{{ student.phone or 'N/A' }}</td>
            </tr>
        </table>
        <br>
        <a href="{{ url_for('home') }}" class="btn btn-primary">Back to List</a>
        <a href="{{ url_for('edit_student', student_id=student_id) }}" class="btn btn-edit">Edit</a>
    </div>
''')

# Student class
class Student:
    def __init__(self, name, email, course, year, phone=None):
        self.name = name
        self.email = email
        self.course = course
        self.year = year
        self.phone = phone

# Routes
@app.route('/')
def home():
    return render_template_string(HOME_TEMPLATE, students=students)

@app.route('/add', methods=['GET', 'POST'])
def add_student():
    global next_id
    if request.method == 'POST':
        name = request.form['name']
        email = request.form['email']
        course = request.form['course']
        year = request.form['year']
        phone = request.form.get('phone')
        
        student = Student(name, email, course, year, phone)
        students[next_id] = student
        next_id += 1
        
        flash('Student added successfully!', 'success')
        return redirect(url_for('home'))
    
    return render_template_string(ADD_STUDENT_TEMPLATE)

@app.route('/view/<int:student_id>')
def view_student(student_id):
    if student_id not in students:
        flash('Student not found!', 'error')
        return redirect(url_for('home'))
    
    return render_template_string(VIEW_STUDENT_TEMPLATE, student=students[student_id], student_id=student_id)

@app.route('/edit/<int:student_id>', methods=['GET', 'POST'])
def edit_student(student_id):
    if student_id not in students:
        flash('Student not found!', 'error')
        return redirect(url_for('home'))
    
    if request.method == 'POST':
        students[student_id].name = request.form['name']
        students[student_id].email = request.form['email']
        students[student_id].course = request.form['course']
        students[student_id].year = request.form['year']
        students[student_id].phone = request.form.get('phone')
        
        flash('Student updated successfully!', 'success')
        return redirect(url_for('home'))
    
    return render_template_string(EDIT_STUDENT_TEMPLATE, student=students[student_id], student_id=student_id)

@app.route('/delete/<int:student_id>')
def delete_student(student_id):
    if student_id in students:
        del students[student_id]
        flash('Student deleted successfully!', 'success')
    else:
        flash('Student not found!', 'error')
    
    return redirect(url_for('home'))

if __name__ == '__main__':
    app.run(debug=True)

---
memory: to_finish
tags:
 - will_learn
language:
 - Python
review-date: ""
last-reviewed:
keywords:
 - celery
 - task queue
 - asynchronous
 - RabbitMQ
 - distributed tasks
scheda: done

---
# **Core Explanation:**

---

Celery is an open-source, **distributed task queue** for Python. It's used to process a large number of messages (tasks) while providing operations like scheduling, monitoring, and error handling. Essentially, it allows your application to **delegate time-consuming tasks** (like sending emails, processing images, generating reports, or complex calculations) to a separate process, preventing your main web application from blocking or becoming unresponsive.

Your Flask application acts as a "producer" or "client" that sends tasks to Celery. These tasks are then placed into a **message broker** (like RabbitMQ, which your application uses). A separate "worker" process constantly monitors the broker for new tasks, picks them up, and executes them.

In your case, you observed "Celery App created successfully!" messages multiple times. This indicates that the Celery application object was being initialized more than once, which isn't ideal but typically won't cause the server to crash itself. It just means the setup logic runs repeatedly.

# **Related Concepts:**

---
- **Asynchronous Processing:** Executing tasks outside the main flow of a program, allowing the main program to continue without waiting for the task to complete. Celery is a prime example.
- **Message Broker:** An intermediary that allows different applications (or parts of the same application) to communicate by sending and receiving messages. **RabbitMQ** is a popular choice for Celery, acting as the queue where tasks wait to be processed.
- **Worker:** A separate process that consumes tasks from the message broker and executes them.
- **Flask (Web Framework):** Your main web application that produces tasks for Celery.
- **Concurrency:** Handling multiple tasks or requests at the same time, often by interleaving their execution.

# **Examples:**

---

Let's consider a common use case: sending an email. Without Celery, if a user signs up, your Flask app might directly call an email sending function. If the email service is slow, the user has to wait for the email to send before their registration completes, leading to a poor user experience. With Celery, the email sending becomes an asynchronous task.
```python

# Backend (Flask app - Producer)

# app.py or run.py

from flask import Flask
from celery import Celery

# Assuming you have a celery app object configured

app = Flask(__name__)

# This part is crucial for Celery to connect to your broker

# Based on your logs, your Celery app is likely configured like this:

# from app.celery.celery import celery_app as celery

# Example of how it might be imported

# Or directly initialized if app.py is small:

# celery = Celery(__name__, broker='pyamqp://guest@rabbitmq//')

# Example task definition (usually in a separate tasks.py or similar file)

# @celery.task

# def send_welcome_email(user_email):

# """

# Simulates sending a welcome email.

# This task runs in a Celery worker, not the main Flask thread.

# """

# print(f"Attempting to send email to {user_email}...")

# import time

# time.sleep(5)

# Simulate a long running process

# print(f"Email sent to {user_email}!")

# return f"Email to {user_email} completed."

@app.route('/register', methods=['POST'])
def register_user:
 user_email = "newuser@example.com"

# Get this from request data in a real app

# Instead of calling send_welcome_email directly, we 'delay' it

# This sends the task to the Celery broker

# make sure 'celery' object is globally accessible or passed appropriately

# e.g., if you have it in app.celery.celery as celery_app

# from app.celery.celery import celery_app

# celery_app.send_task('your_module_name.send_welcome_email', args=[user_email])

 print(f"User {user_email} registered! Email sending initiated via Celery.")
 return "Registration successful! You'll receive a welcome email shortly.", 200

if __name__ == '__main__':

# This is where your Flask app starts.

# The 'Flask application initialized successfully' log comes from here.

# The repeated 'Celery App created successfully!' suggests this part of the code

# or a related import is being executed multiple times, possibly due to Flask's reloader.
 import os
 port = int(os.environ.get('FLASK_PORT', 8000))

# Using FLASK_PORT as discussed
 print(f"Flask is running on .0.1:{port} (Debug: True)")
 print(f"🚀 Server starting on .0.1:{port}")
 app.run(host='127.0.1', port=port, debug=True)

# The 'Address already in use' error happened when 'port' was hardcoded to 8000

# and another process (docker-proxy) was already using it.

# Setting FLASK_PORT=8001 (or similar) on the command line solved this by

# allowing the app to pick up a different port.

# To run the Celery worker (in a separate terminal)

# This process connects to RabbitMQ and listens for tasks

# celery -A app.celery.celery worker --loglevel=info

# (Assuming your Celery app instance is named 'celery' or 'celery_app' in that module)
```

# **Flashcards:**

---
What is Celery used for?;; Celery is a distributed task queue used for asynchronously processing long-running tasks, like sending emails or processing data, outside of the main application thread.

What is a message broker in Celery?;; A message broker (e.g., RabbitMQ) is an intermediary that stores tasks sent by the application, allowing Celery workers to pick them up for execution.

Why did "Celery App created successfully!" appear multiple times in my logs?;; This indicates the Celery application object or its initialization code was executed multiple times, possibly due to Flask's debug reloader or multiple imports/spawning processes.
---
memory: to_finish
tags:
  - will_learn
language:
  - Python
review-date: ""
last-reviewed:
scheda: done
---
# **Core Explanation:**

---
**Flask-CORS** is a Flask extension that handles Cross-Origin Resource Sharing (CORS) headers for you. CORS is a security mechanism implemented by web browsers to restrict web pages from making requests to a different domain than the one that served the web page. This prevents malicious scripts from making unauthorized requests to your APIs.

In your case, you encountered a CORS error because your frontend (running on `.0.1:3000`) was trying to make an API call to your backend (running on `.0.1:8001`). Since these are on different ports, the browser considers them "different origins," and without proper CORS headers from the backend, it blocks the request.

Flask-CORS simplifies adding the necessary `Access-Control-Allow-Origin`, `Access-Control-Allow-Headers`, and `Access-Control-Allow-Methods` headers to your backend's responses, allowing your frontend to communicate with it.

**Step-by-step to implement Flask-CORS:**

1. **Install Flask-CORS:** You'll need to install the library in your backend's Python environment.

 ```bash
 cd ~/InnoBee-Backend
 pip install flask-cors
 ```

2. **Initialize Flask-CORS in your Flask application:** You'll need to import `CORS` from `flask_cors` and then initialize it with your Flask `app` object. This is typically done in your main application file (e.g., `app.py` or `run.py`).

 **Option A: Enable CORS for all routes and origins (less secure, good for development):** This is the simplest way to get started.

 ```Python

# Inside your main Flask app file (e.g., run.py or app.py)
 from flask import Flask
 from flask_cors import CORS

# Import CORS

 app = Flask(__name__)
 CORS(app)

# Initialize CORS for your Flask app, allowing all origins
 ```

 **Option B: Enable CORS for specific origins (more secure, recommended for production):** This is better practice as it explicitly whitelists the origins allowed to make requests.

 ```Python

# Inside your main Flask app file (e.g., run.py or app.py)
 from flask import Flask
 from flask_cors import CORS

# Import CORS

 app = Flask(__name__)

# Initialize CORS for your Flask app, specifying allowed origins

# Replace the origins with the actual URLs where your frontend will run
 CORS(app, origins=[".0.1:3000", "])
 ```

3. **Restart your backend:** After making these code changes, you must restart your Flask backend server for the changes to take effect.

 ```bash
 FLASK_PORT=8001 python3 run.py
 ```

 (Remember that `FLASK_PORT=8001` is needed if your `run.py` is configured to use an environment variable for the port.)

# **Related Concepts:**

---
- [[CORS (Cross-Origin Resource Sharing)]]
- [[Flask (web framework)]]
- **Origin:** A combination of scheme (protocol, e.g., `http`, `https`), hostname (domain, e.g., `example.com`), and port (e.g., `80`, `3000`). If any of these differ between the requesting page and the requested resource, it's considered a cross-origin request.
- **Preflight Request (`OPTIONS` method):** Before certain "non-simple" cross-origin requests (like `POST` requests with custom headers), browsers send an `OPTIONS` request to the server. This "preflight" asks the server for permission to send the actual request and checks which methods and headers are allowed. Your backend must respond correctly to this `OPTIONS` request with appropriate CORS headers. Flask-CORS handles this automatically.
- **Same-Origin Policy:** The fundamental browser security model that CORS relaxes. It dictates that web pages can only request resources from the same origin they were loaded from.
- **HTTP Headers:** Specific pieces of information sent with HTTP requests and responses. CORS relies on `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, etc., to communicate permissions between the browser and server.

# **Examples:**

---
```js

# File: InnoBee-Backend/run.py or app.py (your main Flask app file)

import os
from flask import Flask, jsonify, request
from flask_cors import CORS

# 1. Import the CORS extension

app = Flask(__name__)

# 2. Initialize CORS

# Option A: Allow all origins for development (less secure)
CORS(app)

# Option B: Allow specific origins (more secure, recommended for production)

# Make sure to include all origins where your frontend might be hosted

# CORS(app, origins=[".0.1:3000", "])

# Configure the port dynamically from environment variable, defaulting to 8000

# This assumes your Flask app is set up to read from FLASK_PORT
PORT = int(os.environ.get('FLASK_PORT', 8000))

# Ensure your app uses this variable

#
---
Your existing app setup would go here
---
# (Example: Database connection string placeholder)
connection_string = os.environ.get('DATABASE_URL', 'enter-Your_Db-Url')
print(f"connection_string {connection_string}")

# Example route that your frontend might call
@app.route('/api/v1/auth/login', methods=['POST', 'OPTIONS'])
def login:

# In a real app, you'd handle login logic here

# For OPTIONS requests (preflight), Flask-CORS handles the response
 if request.method == 'OPTIONS':
 return jsonify({"message": "CORS preflight check successful"}), 200

 data = request.get_json
 username = data.get('username')
 password = data.get('password')

 if username == "test" and password == "password":
 return jsonify({"message": "Login successful", "token": "fake_jwt_token"}), 200
 else:
 return jsonify({"message": "Invalid credentials"}), 401

# Example health check endpoint
@app.route('/health', methods=['GET'])
def health_check:
 return jsonify({"status": "Backend is healthy!"}), 200

#
---

End of your existing app setup
---
if __name__ == '__main__':

# 3. Ensure your Flask app runs on the correct port

# The INFO messages about Flask running on 8000 might still appear if not overridden

# by the app.run or a custom server runner.

# The key is that the requests will *respond* on the correct port once CORS is applied.
 print(f"Flask is running on .0.1:{PORT} (Debug: True)")
 print(f"🚀 Server starting on .0.1:{PORT}")
 app.run(host='127.0.1', port=PORT, debug=True)
```

# **Flashcards:**

---
What is the purpose of CORS?;; CORS (Cross-Origin Resource Sharing) is a browser security mechanism that prevents web pages from making requests to a different domain (origin) than the one that served the web page, unless explicitly allowed by the server.

What is a CORS "preflight request" and which HTTP method does it use?;; A preflight request is an `OPTIONS` HTTP request sent by the browser before certain "non-simple" cross-origin requests. It asks the server for permission to send the actual request and checks allowed methods and headers.

How do you fix a CORS error in a Flask backend using Flask-CORS?;; 1. `pip install flask-cors`. 2. Import `CORS` from `flask_cors`. 3. Initialize `CORS(app)` for your Flask application (either for all origins or specific ones). 4. Restart the backend.
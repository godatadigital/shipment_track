# 🚀 Ultimate Guide: Deploying a Flask + Selenium API on Ubuntu VPS

This guide walks you through setting up a raw Ubuntu Virtual Private Server (VPS) to host a Python Flask application that uses Selenium for web scraping. We will use **Gunicorn** as the application server, **Nginx** as the web server, and **Cloudflare** for SSL (HTTPS) and security. 

### 📋 Prerequisites
* [A VPS] (Virtual Private Server) running Ubuntu 20.04 or 22.04.
* [A Domain Name] (e.g., `godata.digital`) connected to Cloudflare.
* [Terminal/Command Prompt] to SSH into your server.

---

## Part 1: Initial Server Setup

1.  **Login to your server** via SSH:
    ```bash
    ssh root@YOUR_SERVER_IP
    # Example: ssh root@74.208.146.50
    ```

2.  **Update your system** to ensure all software is fresh:
    ```bash
    apt update && apt upgrade -y
    ```

3.  **Install Essential Tools:**
    We need Python, pip, Nginx, Supervisor (to keep the app running), and Unzip.
    ```bash
    apt install -y python3-pip python3-venv nginx supervisor unzip curl
    ```

4.  **Install Chrome & ChromeDriver:**
    Since your app uses Selenium, the server needs a browser installed to "browse" the web headlessly.
    ```bash
    # Install Chromium Browser
    apt install -y chromium-browser

    # Install Chromium WebDriver (allows Python to control the browser)
    apt install -y chromium-chromedriver
    ```

---

## Part 2: Application Setup

1.  **Create a Project Directory:**
    We will store our code in `/opt/tracking`.
    ```bash
    mkdir -p /opt/tracking
    cd /opt/tracking
    ```

2.  **Create a Python Virtual Environment:**
    This keeps your project libraries separate from the system.
    ```bash
    python3 -m venv venv
    ```

3.  **Activate the Environment:**
    ```bash
    source venv/bin/activate
    ```

4.  **Install Python Libraries:**
    Install Flask, Gunicorn (the production server), CORS support, and Selenium.
    ```bash
    pip install flask flask-cors selenium gunicorn
    ```

5.  **Create the Python Application File:**
    Create the file `api_server.py`.
    ```bash
    nano api_server.py
    ```

6.  **Paste the Code:**
    *Copy the code below and paste it into the editor. Press `Ctrl+X`, then `Y`, then `Enter` to save.*

    ```python
    from flask import Flask, request, jsonify
    from flask_cors import CORS
    from selenium import webdriver
    from selenium.webdriver.chrome.service import Service
    from selenium.webdriver.chrome.options import Options
    from selenium.webdriver.common.by import By
    from selenium.webdriver.support.ui import WebDriverWait
    from selenium.webdriver.support import expected_conditions as EC
    import time

    # Initialize Flask App
    app = Flask(__name__)
    
    # Enable CORS for all domains (Required for GitHub Pages to talk to this server)
    CORS(app)

    def get_tracking_info(tracking_number):
        """
        Launches a headless Chrome browser to scrape tracking data.
        """
        chrome_options = Options()
        chrome_options.add_argument("--headless")  # Run without a GUI (Critical for VPS)
        chrome_options.add_argument("--no-sandbox")
        chrome_options.add_argument("--disable-dev-shm-usage")
        
        # Point to the installed Chromium driver
        service = Service('/usr/bin/chromedriver')
        driver = webdriver.Chrome(service=service, options=chrome_options)

        try:
            # --- REPLACE THIS URL WITH THE ACTUAL TARGET WEBSITE ---
            target_url = "https://example-shipping-carrier.com/track" 
            driver.get(target_url)

            # --- YOUR SCRAPING LOGIC GOES HERE ---
            # Example:
            # search_box = driver.find_element(By.ID, "track-input")
            # search_box.send_keys(tracking_number)
            # driver.find_element(By.ID, "submit-btn").click()
            
            # Wait for results (Example logic)
            # time.sleep(2) 
            
            # Mock Data for demonstration (Replace with actual scraped data)
            result_data = {
                "title": f"Tracking Result for {tracking_number}",
                "header": ["Date", "Location", "Status"],
                "statuses": [
                    {"date_time": "2023-10-27 10:00", "area": "Warehouse", "details": "Parcel Dispatched"},
                    {"date_time": "2023-10-26 14:30", "area": "Sorting Center", "details": "Processed"}
                ]
            }
            
            return result_data

        except Exception as e:
            print(f"Scraping Error: {e}")
            return None
        finally:
            driver.quit()

    @app.route('/track', methods=['POST'])
    def track_shipment():
        data = request.get_json()
        
        # Validation: Check if tracking_number exists
        if not data or 'tracking_number' not in data:
            return jsonify({'error': "Invalid request. 'tracking_number' field is required."}), 400

        tracking_number = data['tracking_number']
        
        # Call the scraping function
        result = get_tracking_info(tracking_number)

        if result:
            return jsonify(result), 200
        else:
            return jsonify({'error': 'Failed to retrieve tracking data or ID not found.'}), 404

    if __name__ == '__main__':
        # Run locally on port 8000
        app.run(host='0.0.0.0', port=8000)
    ```

---

## Part 3: Configure Supervisor (Keep App Running)

Supervisor ensures your app starts automatically when the server turns on and restarts if it crashes.

1.  **Create the config file:**
    ```bash
    nano /etc/supervisor/conf.d/tracking_app.conf
    ```

2.  **Paste the configuration:**
    *Ensure the paths match exactly where we created the files.*

    ```ini
    [program:tracking_app]
    directory=/opt/tracking
    command=/opt/tracking/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 api_server:app
    autostart=true
    autorestart=true
    stderr_logfile=/var/log/tracking_app.err.log
    stdout_logfile=/var/log/tracking_app.out.log
    user=root
    ```

3.  **Start the App:**
    ```bash
    supervisorctl reread
    supervisorctl update
    supervisorctl start tracking_app
    ```

4.  **Check Status:**
    ```bash
    supervisorctl status
    ```
    *You should see `RUNNING`.*

---

## Part 4: Configure Nginx (The Web Server)

Nginx sits in front of your app, handles internet traffic, and talks to Cloudflare.

1.  **Create the Nginx Config:**
    ```bash
    nano /etc/nginx/sites-available/tracking
    ```

2.  **Paste the Configuration:**
    *Replace `api.godata.digital` with your actual domain.*

    ```nginx
    server {
        listen 80;
        listen [::]:80;
        
        # REPLACE WITH YOUR DOMAIN
        server_name api.godata.digital;

        location /track {
            include proxy_params;
            proxy_pass http://127.0.0.1:8000;
            
            # Optimizations for API traffic
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
    ```

3.  **Enable the Site:**
    ```bash
    ln -s /etc/nginx/sites-available/tracking /etc/nginx/sites-enabled/
    ```

4.  **Test Configuration:**
    ```bash
    nginx -t
    ```
    *If it says "syntax is ok", proceed.*

5.  **Restart Nginx:**
    ```bash
    systemctl restart nginx
    ```

---

## Part 5: Firewall Setup

We need to make sure the "door" is open for web traffic.

1.  **Allow Nginx Traffic:**
    ```bash
    ufw allow 'Nginx Full'
    ```

2.  **Enable Firewall (If not already enabled):**
    ```bash
    ufw enable
    ```
    *(Press `y` to confirm).*

---

## Part 6: Cloudflare Setup (Critical for SSL)

This is where we secure the connection without complex certificate management on the server. 

1.  **Log in to Cloudflare.**
2.  **Go to DNS Settings:**
    * [Add an A Record].
    * [Name]: `api` (or whatever subdomain you want).
    * [Content (IPv4)]: Your VPS IP Address (e.g., `74.208.146.50`).
    * [Proxy Status]: **Proxied** (Orange Cloud). This is mandatory.
3.  **Go to SSL/TLS Settings:**
    * [Set the encryption mode] to **Flexible**.
    * *Why?* Cloudflare talks to your users securely (HTTPS), but talks to your server via standard HTTP (Port 80) to simplify configuration.

---

## Part 7: Final Testing

1.  **Wait 1-2 minutes** for Cloudflare DNS to propagate.
2.  **Test from your local computer terminal:**

    ```bash
    curl -X POST https://api.godata.digital/track \
         -H "Content-Type: application/json" \
         -d '{"tracking_number": "TEST12345"}'
    ```

3.  **Expected Output:**
    You should receive a JSON response with the tracking details.

---

## ✏️ Variables You Must Change

When following this guide, find and replace these placeholders with your actual details:

| Variable | Example Value used in Guide | What you should put |
| :--- | :--- | :--- |
| **YOUR_SERVER_IP** | `74.208.146.50` | The actual public IP of your VPS. |
| **Domain Name** | `api.godata.digital` | Your subdomain (e.g., `api.yourdomain.com`). |
| **Target Website** | `example-shipping.com` | The URL your Python script needs to scrape. |
| **Scraping Logic** | *(In python code)* | Your actual Selenium code to find elements and extract text. |

---

This code is Managed by <a href="https://godata.digital">Godata.digital</a>


# Complete OTP System Guide - Reusable for Any Project

## 📚 Required Libraries & Imports

```python
# Core Libraries (Built-in Python)
import smtplib          # SMTP client for sending emails
import ssl             # SSL/TLS encryption for secure email
import random          # Generate random numbers
import string          # String operations
from datetime import datetime, timedelta  # Date/time handling
import sqlite3         # SQLite database
import hashlib         # Password hashing

# Email Libraries (Built-in)
from email.mime.text import MIMEText        # Create text email
from email.mime.multipart import MIMEMultipart  # Create multi-part email

# Flask Libraries (Install with: pip install flask)
from flask import Flask, request, session, flash, redirect, url_for
```

## 🔧 Core OTP Configuration

### 1. Email Settings (Gmail Example)
```python
# Email Configuration - MUST CHANGE THESE
EMAIL_ADDRESS = "your_email@gmail.com"     # Your Gmail address
EMAIL_PASSWORD = "your_16_char_app_password"  # Gmail App Password
SMTP_SERVER = "smtp.gmail.com"             # Gmail SMTP server
SMTP_PORT = 587                            # Gmail SMTP port

# For other email providers:
# Yahoo: smtp.mail.yahoo.com, port 587
# Outlook: smtp-mail.outlook.com, port 587
# Custom SMTP: your_smtp_server, your_port
```

### 2. Database Setup (SQLite)
```python
DATABASE = 'users.db'  # Database file name

def init_database():
    """Creates database tables if they don't exist"""
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    
    # Users table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            email TEXT UNIQUE NOT NULL,
            password TEXT NOT NULL,
            verified INTEGER DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # OTP table with expiration
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS otp_codes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            email TEXT NOT NULL,
            otp_code TEXT NOT NULL,
            purpose TEXT NOT NULL,
            expires_at TIMESTAMP NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    conn.commit()
    conn.close()
```

## 🎲 Step 1: OTP Generation Function

```python
def generate_otp(length=6):
    """
    Generate random OTP code
    
    Parameters:
    length (int): Length of OTP (default: 6)
    
    Returns:
    str: Random OTP code
    """
    # Method 1: Only digits (recommended for SMS/Email)
    return ''.join(random.choices(string.digits, k=length))
    
    # Method 2: Digits + Letters (more secure)
    # return ''.join(random.choices(string.digits + string.ascii_uppercase, k=length))
    
    # Method 3: Custom characters
    # chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    # return ''.join(random.choices(chars, k=length))

# Usage Example:
# otp = generate_otp()      # Returns: "123456"
# otp = generate_otp(4)     # Returns: "7890"
```

## 💾 Step 2: Database Storage Functions

### Save OTP with Expiration
```python
def save_otp_to_database(email, otp, purpose='verification', expiry_minutes=10):
    """
    Save OTP to database with expiration time
    
    Parameters:
    email (str): User's email
    otp (str): Generated OTP
    purpose (str): verification/password_reset/login
    expiry_minutes (int): OTP validity in minutes
    """
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    
    # Delete existing OTP for this email and purpose
    cursor.execute("DELETE FROM otp_codes WHERE email = ? AND purpose = ?", (email, purpose))
    
    # Calculate expiration time
    expires_at = datetime.now() + timedelta(minutes=expiry_minutes)
    
    # Insert new OTP
    cursor.execute("""
        INSERT INTO otp_codes (email, otp_code, purpose, expires_at)
        VALUES (?, ?, ?, ?)
    """, (email, otp, purpose, expires_at))
    
    conn.commit()
    conn.close()
    print(f"OTP {otp} saved for {email} (expires in {expiry_minutes} minutes)")
```

### Verify OTP from Database
```python
def verify_otp_from_database(email, entered_otp, purpose='verification'):
    """
    Verify OTP from database and check expiration
    
    Parameters:
    email (str): User's email
    entered_otp (str): OTP entered by user
    purpose (str): Purpose of verification
    
    Returns:
    bool: True if OTP is valid and not expired
    """
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    
    # Get latest OTP for this email and purpose
    cursor.execute("""
        SELECT otp_code, expires_at FROM otp_codes 
        WHERE email = ? AND purpose = ?
        ORDER BY created_at DESC LIMIT 1
    """, (email, purpose))
    
    result = cursor.fetchone()
    conn.close()
    
    if not result:
        return False, "No OTP found"
    
    stored_otp, expires_at_str = result
    
    # Check expiration
    expires_at = datetime.fromisoformat(expires_at_str)
    if datetime.now() > expires_at:
        # Delete expired OTP
        delete_otp_from_database(email, purpose)
        return False, "OTP has expired"
    
    # Verify OTP
    if stored_otp == entered_otp:
        # Delete OTP after successful verification
        delete_otp_from_database(email, purpose)
        return True, "OTP verified successfully"
    else:
        return False, "Invalid OTP"

def delete_otp_from_database(email, purpose):
    """Delete OTP after use or expiration"""
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute("DELETE FROM otp_codes WHERE email = ? AND purpose = ?", (email, purpose))
    conn.commit()
    conn.close()
```

## 📧 Step 3: Email Sending Function

```python
def send_otp_email(email, otp, purpose='verification'):
    """
    Send OTP via email using SMTP with SSL
    
    Parameters:
    email (str): Recipient email
    otp (str): OTP code to send
    purpose (str): Purpose of OTP
    
    Returns:
    tuple: (success: bool, message: str)
    """
    try:
        # Create email message
        message = MIMEMultipart()
        message["From"] = EMAIL_ADDRESS
        message["To"] = email
        message["Subject"] = f"Your OTP Code - {purpose.title()}"
        
        # Email body based on purpose
        if purpose == 'verification':
            body = f"""
            Welcome!
            
            Your email verification code is: {otp}
            
            This code expires in 10 minutes.
            Please enter this code to verify your account.
            
            If you didn't request this, please ignore this email.
            """
        elif purpose == 'password_reset':
            body = f"""
            Password Reset Request
            
            Your password reset code is: {otp}
            
            This code expires in 10 minutes.
            Use this code to reset your password.
            
            If you didn't request this, please ignore this email.
            """
        else:
            body = f"""
            Your OTP code is: {otp}
            
            This code expires in 10 minutes.
            """
        
        # Attach body to email
        message.attach(MIMEText(body, "plain"))
        
        # Create secure SSL context
        context = ssl.create_default_context()
        
        # Connect and send email
        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.starttls(context=context)    # Enable encryption
            server.login(EMAIL_ADDRESS, EMAIL_PASSWORD)  # Login
            server.send_message(message)        # Send email
        
        return True, f"OTP sent to {email}"
        
    except smtplib.SMTPAuthenticationError:
        return False, "Email authentication failed - check credentials"
    except smtplib.SMTPRecipientsRefused:
        return False, "Invalid recipient email address"
    except smtplib.SMTPServerDisconnected:
        return False, "SMTP server connection failed"
    except Exception as e:
        return False, f"Email sending failed: {str(e)}"
```

## 🔄 Step 4: Complete OTP Workflow Functions

### Registration with OTP
```python
def register_user_with_otp(email, password):
    """
    Complete user registration with OTP verification
    
    Returns:
    dict: {'success': bool, 'message': str, 'otp_sent': bool}
    """
    # Check if user already exists
    if user_exists_in_database(email):
        return {'success': False, 'message': 'Email already registered', 'otp_sent': False}
    
    # Save user (unverified)
    if not save_user_to_database(email, password, verified=False):
        return {'success': False, 'message': 'Failed to create account', 'otp_sent': False}
    
    # Generate and save OTP
    otp = generate_otp()
    save_otp_to_database(email, otp, 'verification', expiry_minutes=10)
    
    # Send OTP email
    email_sent, email_message = send_otp_email(email, otp, 'verification')
    
    if email_sent:
        return {
            'success': True, 
            'message': 'Registration successful! Check email for OTP', 
            'otp_sent': True
        }
    else:
        # Remove user if email failed
        delete_user_from_database(email)
        return {'success': False, 'message': f'Registration failed: {email_message}', 'otp_sent': False}
```

### Login with OTP Verification
```python
def login_user_with_otp_check(email, password):
    """
    Login user with OTP verification if email not verified
    
    Returns:
    dict: {'success': bool, 'verified': bool, 'message': str}
    """
    # Verify credentials
    user_data = verify_user_credentials(email, password)
    
    if not user_data:
        return {'success': False, 'verified': False, 'message': 'Invalid credentials'}
    
    if user_data['verified']:
        return {'success': True, 'verified': True, 'message': 'Login successful'}
    else:
        # Send OTP for verification
        otp = generate_otp()
        save_otp_to_database(email, otp, 'verification')
        send_otp_email(email, otp, 'verification')
        
        return {
            'success': False, 
            'verified': False, 
            'message': 'Please verify your email first. OTP sent!'
        }
```

### Password Reset with OTP
```python
def initiate_password_reset(email):
    """
    Start password reset process with OTP
    
    Returns:
    dict: {'success': bool, 'message': str}
    """
    if not user_exists_in_database(email):
        return {'success': False, 'message': 'Email not found'}
    
    # Generate and save OTP
    otp = generate_otp()
    save_otp_to_database(email, otp, 'password_reset', expiry_minutes=15)  # 15 min for reset
    
    # Send email
    email_sent, email_message = send_otp_email(email, otp, 'password_reset')
    
    if email_sent:
        return {'success': True, 'message': 'Password reset OTP sent to email'}
    else:
        return {'success': False, 'message': f'Failed to send email: {email_message}'}

def complete_password_reset(email, otp, new_password):
    """
    Complete password reset after OTP verification
    
    Returns:
    dict: {'success': bool, 'message': str}
    """
    # Verify OTP
    otp_valid, otp_message = verify_otp_from_database(email, otp, 'password_reset')
    
    if not otp_valid:
        return {'success': False, 'message': otp_message}
    
    # Update password
    if update_user_password(email, new_password):
        return {'success': True, 'message': 'Password reset successful'}
    else:
        return {'success': False, 'message': 'Failed to update password'}
```

## 🧹 Step 5: Maintenance Functions

### Cleanup Expired OTPs
```python
def cleanup_expired_otps():
    """
    Remove expired OTPs from database
    Call this periodically or before OTP operations
    
    Returns:
    int: Number of expired OTPs deleted
    """
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    
    cursor.execute("DELETE FROM otp_codes WHERE expires_at < ?", (datetime.now(),))
    deleted_count = cursor.rowcount
    
    conn.commit()
    conn.close()
    
    if deleted_count > 0:
        print(f"Cleaned up {deleted_count} expired OTPs")
    
    return deleted_count

def get_otp_stats():
    """
    Get OTP statistics for monitoring
    
    Returns:
    dict: OTP statistics
    """
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    
    # Total active OTPs
    cursor.execute("SELECT COUNT(*) FROM otp_codes WHERE expires_at > ?", (datetime.now(),))
    active_otps = cursor.fetchone()[0]
    
    # Expired OTPs
    cursor.execute("SELECT COUNT(*) FROM otp_codes WHERE expires_at <= ?", (datetime.now(),))
    expired_otps = cursor.fetchone()[0]
    
    # OTPs by purpose
    cursor.execute("""
        SELECT purpose, COUNT(*) FROM otp_codes 
        WHERE expires_at > ? 
        GROUP BY purpose
    """, (datetime.now(),))
    by_purpose = dict(cursor.fetchall())
    
    conn.close()
    
    return {
        'active_otps': active_otps,
        'expired_otps': expired_otps,
        'by_purpose': by_purpose
    }
```

## 🚀 Usage Examples for Any Project

### Example 1: Django Project
```python
# In views.py
from .otp_system import generate_otp, save_otp_to_database, send_otp_email

def register_view(request):
    if request.method == 'POST':
        email = request.POST['email']
        password = request.POST['password']
        
        # Generate OTP
        otp = generate_otp()
        save_otp_to_database(email, otp, 'verification')
        
        # Send email
        send_otp_email(email, otp)
        
        return redirect('verify_otp', email=email)
```

### Example 2: FastAPI Project
```python
from fastapi import FastAPI
from .otp_system import generate_otp, verify_otp_from_database

app = FastAPI()

@app.post("/send-otp")
async def send_otp(email: str):
    otp = generate_otp()
    save_otp_to_database(email, otp, 'verification')
    success, message = send_otp_email(email, otp)
    return {"success": success, "message": message}

@app.post("/verify-otp")
async def verify_otp(email: str, otp: str):
    valid, message = verify_otp_from_database(email, otp)
    return {"valid": valid, "message": message}
```

### Example 3: Standalone Script
```python
from otp_system import *

# Initialize database
init_database()

# Register user
email = "user@example.com"
otp = generate_otp()
save_otp_to_database(email, otp, 'verification')
send_otp_email(email, otp)

# Later, verify OTP
user_otp = input("Enter OTP: ")
valid, message = verify_otp_from_database(email, user_otp)
print(f"Verification: {message}")
```

## ⚙️ Configuration for Different Use Cases

### SMS OTP (using Twilio)
```python
import twilio
from twilio.rest import Client

def send_otp_sms(phone_number, otp):
    client = Client(TWILIO_SID, TWILIO_TOKEN)
    message = client.messages.create(
        body=f"Your OTP is: {otp}",
        from_=TWILIO_PHONE,
        to=phone_number
    )
    return message.sid
```

### WhatsApp OTP (using Twilio)
```python
def send_otp_whatsapp(phone_number, otp):
    client = Client(TWILIO_SID, TWILIO_TOKEN)
    message = client.messages.create(
        body=f"Your OTP is: {otp}",
        from_='whatsapp:+14155238886',
        to=f'whatsapp:{phone_number}'
    )
    return message.sid
```

## 🔒 Security Best Practices

1. **Rate Limiting**: Limit OTP requests per email/IP
2. **OTP Complexity**: Use longer OTPs for sensitive operations
3. **Expiry Times**: Shorter expiry for critical operations
4. **Attempt Limits**: Lock after multiple failed attempts
5. **Logging**: Log all OTP activities for monitoring

## 📋 Quick Integration Checklist

- [ ] Install required libraries
- [ ] Configure email settings
- [ ] Set up database tables
- [ ] Copy OTP functions
- [ ] Test OTP generation
- [ ] Test email sending
- [ ] Test database storage
- [ ] Test OTP verification
- [ ] Add cleanup routine
- [ ] Implement in your routes/views

This complete guide can be adapted for any Python project! 🎯

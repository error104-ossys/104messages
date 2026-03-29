# 104messages
linux messanger to mobile
#!/usr/bin/env python3
"""
WhatsApp Messenger - Send and receive WhatsApp messages via Twilio
Green theme terminal app with conversation interface
"""

import os
import threading
import time
from datetime import datetime
from flask import Flask, request, jsonify
import requests
from twilio.rest import Client
from twilio.twiml.messaging_response import MessagingResponse

# ==================== CONFIGURATION ====================
# !!! REPLACE WITH YOUR TWILIO CREDENTIALS !!!
TWILIO_ACCOUNT_SID = "YOUR_ACCOUNT_SID"      # From twilio.com/console
TWILIO_AUTH_TOKEN = "YOUR_AUTH_TOKEN"        # From twilio.com/console
# Twilio WhatsApp Sandbox Number (THIS IS CORRECT - Twilio's official WhatsApp number)
TWILIO_WHATSAPP_NUMBER = "whatsapp:+14155238886"

# For receiving messages - use ngrok for local testing
WEBHOOK_PORT = 5000
WEBHOOK_URL = None  # Will be set when ngrok starts

# ANSI Color codes for green theme
GREEN = '\033[92m'
DARK_GREEN = '\033[32m'
BOLD_GREEN = '\033[1;32m'
LIGHT_GREEN = '\033[38;5;82m'
YELLOW = '\033[93m'
RED = '\033[91m'
BLUE = '\033[94m'
CYAN = '\033[96m'
WHITE = '\033[97m'
RESET = '\033[0m'
CLEAR = '\033[2J\033[H'

# Message storage
messages = []  # List of dicts: {'from': str, 'to': str, 'message': str, 'time': str, 'direction': str}


class WhatsAppMessenger:
    def __init__(self):
        self.client = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)
        self.running = True
        self.my_name = "Me"
        
    def send_whatsapp(self, to_number, message_body):
        """Send a WhatsApp message via Twilio"""
        try:
            # Ensure numbers have whatsapp: prefix
            from_whatsapp = TWILIO_WHATSAPP_NUMBER
            to_whatsapp = to_number if to_number.startswith('whatsapp:') else f"whatsapp:{to_number}"
            
            message = self.client.messages.create(
                body=message_body,
                from_=from_whatsapp,
                to=to_whatsapp
            )
            return True, message.sid
        except Exception as e:
            return False, str(e)
    
    def send_message_interactive(self, to_number, message_body):
        """Send message and store in conversation history"""
        success, result = self.send_whatsapp(to_number, message_body)
        
        if success:
            messages.append({
                'from': self.my_name,
                'to': to_number,
                'message': message_body,
                'time': datetime.now().strftime("%H:%M:%S"),
                'direction': 'outgoing',
                'status': 'sent'
            })
            return True
        else:
            messages.append({
                'from': self.my_name,
                'to': to_number,
                'message': f"[FAILED] {message_body}",
                'time': datetime.now().strftime("%H:%M:%S"),
                'direction': 'outgoing',
                'status': f'error: {result}'
            })
            return False
    
    def add_incoming_message(self, from_number, message_body):
        """Add an incoming message to conversation history"""
        # Clean up the from number (remove whatsapp: prefix if present)
        clean_number = from_number.replace('whatsapp:', '')
        messages.append({
            'from': clean_number,
            'to': TWILIO_WHATSAPP_NUMBER,
            'message': message_body,
            'time': datetime.now().strftime("%H:%M:%S"),
            'direction': 'incoming'
        })


# ==================== FLASK WEBHOOK FOR RECEIVING MESSAGES ====================
flask_app = Flask(__name__)
messenger = None  # Will be set after initialization


@flask_app.route("/whatsapp-webhook", methods=['GET', 'POST'])
def whatsapp_webhook():
    """Handle incoming WhatsApp messages from Twilio"""
    global messenger
    
    if request.method == 'GET':
        # Twilio webhook verification
        return str(request.args.get('hub.challenge', ''))
    
    # POST - incoming message
    from_number = request.form.get('From', '')
    message_body = request.form.get('Body', '')
    
    print(f"\n{LIGHT_GREEN}📩 INCOMING WHATSAPP from {from_number}{RESET}")
    print(f"   Message: {message_body}")
    
    # Add to conversation history
    if messenger:
        messenger.add_incoming_message(from_number, message_body)
    
    # Send auto-reply (optional - remove if you don't want auto-reply)
    resp = MessagingResponse()
    # Uncomment below to send auto-reply:
    # resp.message("✅ Message received! - From WhatsApp Terminal App")
    
    return str(resp)


def run_webhook_server():
    """Run Flask server in a separate thread"""
    flask_app.run(host='0.0.0.0', port=WEBHOOK_PORT, debug=False, use_reloader=False)


# ==================== UI FUNCTIONS ====================
def clear_screen():
    """Clear terminal with green theme"""
    print(CLEAR, end='')
    print(f"{GREEN}{'='*60}{RESET}")
    print(f"{BOLD_GREEN}💬 WHATSAPP MESSENGER - Green Terminal Edition{RESET}")
    print(f"{GREEN}{'='*60}{RESET}")
    print(f"{DARK_GREEN}Twilio WhatsApp API | Real WhatsApp messaging{RESET}")
    print(f"{GREEN}{'='*60}{RESET}\n")


def print_message_box():
    """Display the conversation messages box"""
    if not messages:
        print(f"{DARK_GREEN}┌{'─'*58}┐{RESET}")
        print(f"{DARK_GREEN}│{RESET} {YELLOW}📭 No messages yet. Send a WhatsApp to get started!{RESET} {DARK_GREEN}│{RESET}")
        print(f"{DARK_GREEN}└{'─'*58}┘{RESET}\n")
        return
    
    # Calculate box width
    box_width = 60
    
    print(f"{DARK_GREEN}┌{'─'*box_width}┐{RESET}")
    print(f"{DARK_GREEN}│{RESET} {BOLD_GREEN}📨 WHATSAPP CONVERSATIONS{RESET} {' '*(box_width-27)}{DARK_GREEN}│{RESET}")
    print(f"{DARK_GREEN}├{'─'*box_width}┤{RESET}")
    
    # Show last 20 messages
    for msg in messages[-20:]:
        time_str = msg['time']
        direction = msg['direction']
        
        if direction == 'outgoing':
            prefix = f"{GREEN}➤ TO {msg['to']}{RESET}"
            color = LIGHT_GREEN
        else:
            # Truncate long phone numbers
            from_display = msg['from'][:15] + "..." if len(msg['from']) > 18 else msg['from']
            prefix = f"{CYAN}◀ FROM {from_display}{RESET}"
            color = CYAN
        
        # Truncate long messages
        message_display = msg['message'][:50] + "..." if len(msg['message']) > 50 else msg['message']
        
        # Print message line
        line = f"{DARK_GREEN}│{RESET} {color}{time_str}{RESET} {prefix}: {WHITE}{message_display}{RESET}"
        # Pad to box width
        padding = box_width - (len(time_str) + len(prefix) + len(message_display) + 4)
        if padding < 0:
            padding = 0
        print(f"{line}{' '*padding}{DARK_GREEN}│{RESET}")
    
    print(f"{DARK_GREEN}└{'─'*box_width}┘{RESET}\n")


def print_status_bar():
    """Display status information"""
    print(f"{GREEN}{'─'*60}{RESET}")
    print(f"{DARK_GREEN}📊 Status:{RESET}")
    print(f"   {GREEN}✓{RESET} Twilio WhatsApp Connected")
    print(f"   {GREEN}✓{RESET} WhatsApp Number: {WHITE}{TWILIO_WHATSAPP_NUMBER}{RESET}")
    if WEBHOOK_URL:
        print(f"   {GREEN}✓{RESET} Webhook: {WHITE}{WEBHOOK_URL}/whatsapp-webhook{RESET}")
    print(f"   {YELLOW}ℹ{RESET} Total Messages: {WHITE}{len(messages)}{RESET}")
    print(f"{GREEN}{'─'*60}{RESET}\n")


def print_menu():
    """Print the main menu"""
    print(f"{BOLD_GREEN}┌───────────── MAIN MENU ─────────────┐{RESET}")
    print(f"{GREEN}│{RESET}  {WHITE}1.{RESET} {GREEN}Send a single WhatsApp{RESET}        {GREEN}│{RESET}")
    print(f"{GREEN}│{RESET}  {WHITE}2.{RESET} {GREEN}Send multiple messages{RESET}      {GREEN}│{RESET}")
    print(f"{GREEN}│{RESET}  {WHITE}3.{RESET} {GREEN}View message history{RESET}        {GREEN}│{RESET}")
    print(f"{GREEN}│{RESET}  {WHITE}4.{RESET} {GREEN}Clear message history{RESET}       {GREEN}│{RESET}")
    print(f"{GREEN}│{RESET}  {WHITE}5.{RESET} {GREEN}Set your display name{RESET}       {GREEN}│{RESET}")
    print(f"{GREEN}│{RESET}  {WHITE}6.{RESET} {YELLOW}Exit{RESET}                        {GREEN}│{RESET}")
    print(f"{GREEN}└────────────────────────────────────┘{RESET}")


def get_phone_number(prompt="📞 Enter phone number (with country code, e.g., +14155552671): "):
    """Get and validate phone number"""
    while True:
        number = input(f"{GREEN}{prompt}{RESET}").strip()
        if not number:
            print(f"{RED}❌ Phone number required!{RESET}")
            continue
        if not number.startswith('+'):
            print(f"{YELLOW}⚠️  Warning: Phone number should include country code (e.g., +1 for US){RESET}")
            confirm = input(f"{GREEN}Continue anyway? (y/n): {RESET}").lower()
            if confirm != 'y':
                continue
        return number


def send_single_message():
    """Send a single WhatsApp message"""
    clear_screen()
    print_message_box()
    
    print(f"{BOLD_GREEN}💬 SEND SINGLE WHATSAPP{RESET}\n")
    
    to_number = get_phone_number()
    print(f"{GREEN}✏️  Enter your message (press Enter twice to send):{RESET}")
    
    lines = []
    while True:
        line = input()
        if line == "" and len(lines) > 0:
            break
        elif line == "":
            continue
        lines.append(line)
    
    message = "\n".join(lines)
    
    if not message:
        print(f"{RED}❌ Message cannot be empty!{RESET}")
        input(f"{GREEN}Press Enter to continue...{RESET}")
        return
    
    print(f"\n{YELLOW}📡 Sending WhatsApp message...{RESET}")
    
    if messenger.send_message_interactive(to_number, message):
        print(f"{GREEN}✅ WhatsApp message sent successfully!{RESET}")
    else:
        print(f"{RED}❌ Failed to send message. Check your credentials.{RESET}")
    
    print(f"\n{GREEN}Press Enter to continue...{RESET}")
    input()


def send_multiple_messages():
    """Send multiple WhatsApp messages"""
    clear_screen()
    print_message_box()
    
    print(f"{BOLD_GREEN}💬 SEND MULTIPLE WHATSAPP{RESET}\n")
    
    try:
        count = int(input(f"{GREEN}🔢 How many messages do you want to send? {RESET}"))
        if count <= 0:
            print(f"{RED}❌ Number must be positive!{RESET}")
            input(f"{GREEN}Press Enter...{RESET}")
            return
        if count > 50:
            print(f"{YELLOW}⚠️  Sending {count} messages may take a while. Continue? (y/n): {RESET}")
            if input().lower() != 'y':
                return
    except ValueError:
        print(f"{RED}❌ Invalid number!{RESET}")
        input(f"{GREEN}Press Enter...{RESET}")
        return
    
    to_number = get_phone_number()
    print(f"{GREEN}✏️  Enter your message (press Enter twice to send):{RESET}")
    
    lines = []
    while True:
        line = input()
        if line == "" and len(lines) > 0:
            break
        elif line == "":
            continue
        lines.append(line)
    
    message = "\n".join(lines)
    
    if not message:
        print(f"{RED}❌ Message cannot be empty!{RESET}")
        input(f"{GREEN}Press Enter...{RESET}")
        return
    
    print(f"\n{YELLOW}📡 Sending {count} WhatsApp messages...{RESET}")
    
    success_count = 0
    for i in range(count):
        print(f"   Sending {i+1}/{count}...", end=' ')
        if messenger.send_message_interactive(to_number, f"[{i+1}/{count}] {message}"):
            print(f"{GREEN}✓{RESET}")
            success_count += 1
        else:
            print(f"{RED}✗{RESET}")
        time.sleep(1)  # Rate limiting
    
    print(f"\n{GREEN}✅ Sent {success_count}/{count} messages successfully!{RESET}")
    input(f"{GREEN}Press Enter to continue...{RESET}")


def view_history():
    """View full message history"""
    clear_screen()
    print_message_box()
    print(f"{GREEN}Press Enter to return to menu...{RESET}")
    input()


def clear_history():
    """Clear message history"""
    clear_screen()
    print_message_box()
    confirm = input(f"{YELLOW}⚠️  Are you sure you want to clear all message history? (y/n): {RESET}")
    if confirm.lower() == 'y':
        messages.clear()
        print(f"{GREEN}✅ Message history cleared!{RESET}")
    else:
        print(f"{GREEN}❌ Cancelled.{RESET}")
    input(f"{GREEN}Press Enter to continue...{RESET}")


def set_display_name():
    """Set the display name for outgoing messages"""
    clear_screen()
    print_message_box()
    current = messenger.my_name
    print(f"{BOLD_GREEN}👤 SET DISPLAY NAME{RESET}\n")
    print(f"{DARK_GREEN}Current name: {WHITE}{current}{RESET}")
    new_name = input(f"{GREEN}Enter new name (or press Enter to keep current): {RESET}").strip()
    if new_name:
        messenger.my_name = new_name
        print(f"{GREEN}✅ Display name changed to '{new_name}'{RESET}")
    input(f"{GREEN}Press Enter to continue...{RESET}")


def start_ngrok():
    """Start ngrok tunnel for local webhook testing"""
    try:
        import subprocess
        
        def run_ngrok():
            subprocess.run(["ngrok", "http", str(WEBHOOK_PORT)], capture_output=False)
        
        # Start ngrok in background thread
        ngrok_thread = threading.Thread(target=run_ngrok, daemon=True)
        ngrok_thread.start()
        time.sleep(2)
        
        # Get public URL from ngrok API
        response = requests.get("http://localhost:4040/api/tunnels")
        if response.status_code == 200:
            tunnels = response.json().get('tunnels', [])
            for tunnel in tunnels:
                if tunnel['proto'] == 'https':
                    return tunnel['public_url']
        return None
    except Exception as e:
        print(f"{YELLOW}⚠️  Could not start ngrok automatically. Please run 'ngrok http {WEBHOOK_PORT}' manually.{RESET}")
        return None


# ==================== TWILIO WHATSAPP SETUP GUIDE ====================
def print_twilio_whatsapp_guide():
    """Print setup instructions for Twilio WhatsApp Sandbox"""
    print(f"{BOLD_GREEN}{'='*60}{RESET}")
    print(f"{BOLD_GREEN}📱 TWILIO WHATSAPP SETUP GUIDE{RESET}")
    print(f"{BOLD_GREEN}{'='*60}{RESET}")
    print(f"""
{YELLOW}Step 1: Create a Twilio Account{RESET}
  • Go to {CYAN}https://www.twilio.com/try-twilio{RESET}
  • Sign up for a free account (they give you $15 credit)

{YELLOW}Step 2: Activate WhatsApp Sandbox{RESET}
  • Go to {CYAN}https://console.twilio.com/us1/develop/sms/try-it-out/whatsapp-learn{RESET}
  • Click "Get Started" for WhatsApp Sandbox
  • You'll see a QR code and instructions

{YELLOW}Step 3: Join the Sandbox from YOUR Phone{RESET}
  • On YOUR phone, open WhatsApp
  • Send the code word shown (usually "join <word>") to {CYAN}+14155238886{RESET}
  • Example: Send {WHITE}"join suitable-outcome"{RESET} to +14155238886

{YELLOW}Step 4: Configure Webhook (to receive replies){RESET}
  • Install ngrok: {CYAN}https://ngrok.com/download{RESET}
  • Run: {WHITE}ngrok http 5000{RESET}
  • Copy your HTTPS URL (e.g., https://abc123.ngrok.io)
  • In Twilio Console > WhatsApp Sandbox
  • Set "When a message comes in" to: {WHITE}{YOUR_NGROK_URL}/whatsapp-webhook{RESET}

{YELLOW}Step 5: Add Your Credentials{RESET}
  • Edit this script and add:
  • {WHITE}TWILIO_ACCOUNT_SID{RESET} - from twilio.com/console
  • {WHITE}TWILIO_AUTH_TOKEN{RESET} - from twilio.com/console
  • {WHITE}TWILIO_WHATSAPP_NUMBER{RESET} = "whatsapp:+14155238886" (already set)

{YELLOW}⚠️ IMPORTANT NOTES:{RESET}
  • You can ONLY message numbers that have joined your sandbox first
  • Your recipient must send the join code to Twilio's number ONCE
  • After they join, you can message them freely for 24 hours
  • To message ANY number, you need to upgrade to a production number
  • WhatsApp Business API costs ~$0.005 per message after free credits
""")


# ==================== MAIN ====================
def main():
    global messenger, WEBHOOK_URL
    
    # Clear screen and show header
    clear_screen()
    
    # Show setup guide
    print_twilio_whatsapp_guide()
    
    # Check credentials
    if TWILIO_ACCOUNT_SID == "YOUR_ACCOUNT_SID" or TWILIO_AUTH_TOKEN == "YOUR_AUTH_TOKEN":
        print(f"{RED}{'='*60}{RESET}")
        print(f"{RED}⚠️  TWILIO CREDENTIALS NOT CONFIGURED!{RESET}")
        print(f"{RED}{'='*60}{RESET}")
        print(f"{YELLOW}\nPlease edit the script and add your Twilio credentials:{RESET}")
        print(f"  - TWILIO_ACCOUNT_SID (from twilio.com/console)")
        print(f"  - TWILIO_AUTH_TOKEN (from twilio.com/console)\n")
        input(f"{GREEN}After adding credentials, press Enter to continue...{RESET}")
        return
    
    # Initialize messenger
    messenger = WhatsAppMessenger()
    
    # Start webhook server for receiving messages
    webhook_thread = threading.Thread(target=run_webhook_server, daemon=True)
    webhook_thread.start()
    
    print(f"{GREEN}✅ Webhook server started on port {WEBHOOK_PORT}{RESET}")
    
    # Try to start ngrok automatically
    print(f"{YELLOW}🔄 Attempting to start ngrok tunnel...{RESET}")
    WEBHOOK_URL = start_ngrok()
    
    if WEBHOOK_URL:
        print(f"{GREEN}✅ Ngrok tunnel established!{RESET}")
        print(f"{WHITE}   Public URL: {WEBHOOK_URL}/whatsapp-webhook{RESET}")
        print(f"{YELLOW}   ⚠️  IMPORTANT: Configure this URL in Twilio Console:{RESET}")
        print(f"      1. Go to twilio.com/console/whatsapp/sandbox")
        print(f"      2. Set webhook to: {WEBHOOK_URL}/whatsapp-webhook")
        print(f"      3. Set method to HTTP POST\n")
    else:
        print(f"{YELLOW}⚠️  Ngrok not found or not running. Install ngrok from https://ngrok.com/{RESET}")
        print(f"{WHITE}   Then run in a separate terminal: ngrok http {WEBHOOK_PORT}{RESET}")
        print(f"{WHITE}   And set your Twilio webhook to your ngrok URL/whatsapp-webhook{RESET}\n")
    
    input(f"{GREEN}Press Enter to continue to the main app...{RESET}")
    
    # Main loop
    while True:
        clear_screen()
        print_message_box()
        print_status_bar()
        print_menu()
        
        choice = input(f"\n{GREEN}👉 Enter your choice (1-6): {RESET}").strip()
        
        if choice == '1':
            send_single_message()
        elif choice == '2':
            send_multiple_messages()
        elif choice == '3':
            view_history()
        elif choice == '4':
            clear_history()
        elif choice == '5':
            set_display_name()
        elif choice == '6':
            print(f"\n{GREEN}👋 Goodbye!{RESET}")
            break
        else:
            print(f"{RED}❌ Invalid choice!{RESET}")
            time.sleep(1)


if __name__ == "__main__":
    main()

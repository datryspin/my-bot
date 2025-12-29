{
  "name": "whatsapp-web-bot",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "body-parser": "^1.20.2",
    "whatsapp-web.js": "^1.20.0",
    "qrcode-terminal": "^0.12.0"
  }
}
name=index.js
const express = require('express');
const bodyParser = require('body-parser');
const qrcode = require('qrcode-terminal');
const { Client, LocalAuth, MessageMedia } = require('whatsapp-web.js');

const app = express();
app.use(bodyParser.json());

// Initialize WhatsApp client with LocalAuth (session persisted to .wwebjs_auth)
const client = new Client({
  authStrategy: new LocalAuth()
});

// QR code for initial auth
client.on('qr', qr => {
  console.log('Scan this QR with your phone:');
  qrcode.generate(qr, { small: true });
});

// Ready event
client.on('ready', () => {
  console.log('WhatsApp client is ready');
});

// Incoming messages handler
client.on('message', async msg => {
  console.log('Message from', msg.from, ':', msg.body);
  // simple command: reply 'pong' to 'ping'
  if (msg.body && msg.body.toLowerCase() === 'ping') {
    await msg.reply('pong');
  }

  // echo example
  if (msg.body && msg.body.toLowerCase().startsWith('echo ')) {
    const text = msg.body.slice(5);
    await msg.reply(`You said: ${text}`);
  }
});

// Initialize client
client.initialize();

// HTTP API to send messages from other services
app.post('/send', async (req, res) => {
  const { number, message } = req.body;
  if (!number || !message) return res.status(400).json({ error: 'number and message required' });

  // number format: country code + number, e.g. "15551234567" -> "15551234567@c.us"
  const chatId = number.includes('@') ? number : `${number}@c.us`;

  try {
    const sent = await client.sendMessage(chatId, message);
    return res.json({ id: sent.id._serialized });
  } catch (err) {
    console.error(err);
    return res.status(500).json({ error: err.message });
  }
});

// Send media example (multipart / base64 or local file path)
app.post('/send-media', async (req, res) => {
  // Example expects JSON with { number, caption, base64, filename, mimetype }
  const { number, caption, base64, filename, mimetype } = req.body;
  const chatId = number.includes('@') ? number : `${number}@c.us`;
  try {
    const media = MessageMedia.fromBase64Data(base64, filename);
    const sent = await client.sendMessage(chatId, media, { caption });
    return res.json({ id: sent.id._serialized });
  } catch (err) {
    console.error(err);
    return res.status(500).json({ error: err.message });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server listening on ${PORT}`));
README.md
v1
# WhatsApp Web Bot (whatsapp-web.js)

1. Install:
   - Node 16+ recommended
   - npm install

Option B — Python Flask + Twilio WhatsApp API (official)

Pros: official WhatsApp via Twilio sandbox/production, webhooks from Twilio, reliable and compliant.
Cons: Twilio costs, set up Twilio sandbox or WhatsApp business number.
name=requirements.txt
Flask==2.3.2
twilio==8.5.1
python-dotenv==1.0.0
name=.env.example
# Copy to .env and fill with your Twilio creds
TWILIO_ACCOUNT_SID=ACXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_WHATSAPP_NUMBER=whatsapp:+14155238886
FLASK_ENV=development
PORT=5000
name=app.py
import os
from flask import Flask, request, jsonify
from twilio.twiml.messaging_response import MessagingResponse
from twilio.rest import Client
from dotenv import load_dotenv

load_dotenv()  # loads .env in dev

TWILIO_SID = os.environ.get('TWILIO_ACCOUNT_SID')
TWILIO_TOKEN = os.environ.get('TWILIO_AUTH_TOKEN')
TWILIO_WHATSAPP = os.environ.get('TWILIO_WHATSAPP_NUMBER')  # e.g. "whatsapp:+14155238886"

if not (TWILIO_SID and TWILIO_TOKEN and TWILIO_WHATSAPP):
    raise RuntimeError("TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN and TWILIO_WHATSAPP_NUMBER must be set")

client = Client(TWILIO_SID, TWILIO_TOKEN)
app = Flask(__name__)

# Webhook endpoint for incoming messages from Twilio
@app.route('/webhook', methods=['POST'])
def webhook():
    from_number = request.form.get('From')
    message_body = request.form.get('Body')
    print(f"Incoming from {from_number}: {message_body}")

    resp = MessagingResponse()
    # simple bot logic
    if message_body:
        body_lower = message_body.strip().lower()
        if body_lower == 'hi' or body_lower == 'hello':
            resp.message("Hello! I'm a Twilio WhatsApp bot. Reply 'help' for options.")
        elif body_lower == 'help':
            resp.message("Available commands: hi, help, echo <text>")
        elif body_lower.startswith('echo '):
            resp.message("Echo: " + message_body[5:])
        else:
            resp.message("I received: " + message_body)
    else:
        resp.message("Empty message received")

    return str(resp)

# HTTP API to send messages programmatically
@app.route('/send', methods=['POST'])
def send():
    data = request.json or {}
    to = data.get('to')  # should be in 'whatsapp:+<number>' format
    body = data.get('body')
    if not to or not body:
        return jsonify({'error': 'to and body required'}), 400

    message = client.messages.create(
        body=body,
        from_=TWILIO_WHATSAPP,
        to=to
    )
    return jsonify({'sid': message.sid})

if __name__ == '__main__':
    port = int(os.getenv('PORT', 5000))
    app.run(host='0.0.0.0', port=port)

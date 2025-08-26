import os
import google.generativeai as genai
from flask import Flask, render_template_string, request, jsonify

app = Flask(__name__)

# Configure Gemini (will use environment variable)
genai.configure(api_key=os.environ.get("GEMINI_API_KEY"))
model = genai.GenerativeModel('gemini-1.5-flash')

# HTML Template with CSS and JavaScript - ALL IN ONE PAGE
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gemini Chatbot</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { 
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; 
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            height: 100vh; 
            display: flex; 
            justify-content: center; 
            align-items: center; 
        }
        .chat-container { 
            width: 90%; 
            max-width: 500px; 
            background: white; 
            border-radius: 20px; 
            box-shadow: 0 20px 40px rgba(0,0,0,0.1); 
            overflow: hidden; 
            height: 80vh; 
            display: flex; 
            flex-direction: column; 
        }
        .chat-header { 
            background: linear-gradient(90deg, #4facfe 0%, #00f2fe 100%); 
            color: white; 
            padding: 20px; 
            text-align: center; 
            font-weight: bold; 
            font-size: 1.2em; 
        }
        .chat-box { 
            flex: 1; 
            padding: 20px; 
            overflow-y: auto; 
            background: #f8f9fa; 
        }
        .message { 
            margin: 10px 0; 
            padding: 12px 16px; 
            border-radius: 18px; 
            max-width: 80%; 
            word-wrap: break-word; 
        }
        .user-message { 
            background: #007bff; 
            color: white; 
            margin-left: auto; 
            border-bottom-right-radius: 5px; 
        }
        .bot-message { 
            background: #e9ecef; 
            color: #333; 
            margin-right: auto; 
            border-bottom-left-radius: 5px; 
        }
        .input-container { 
            padding: 15px; 
            background: white; 
            border-top: 1px solid #eee; 
            display: flex; 
            gap: 10px; 
        }
        #user-input { 
            flex: 1; 
            padding: 12px; 
            border: 2px solid #e9ecef; 
            border-radius: 25px; 
            outline: none; 
            font-size: 16px; 
        }
        #user-input:focus { 
            border-color: #007bff; 
        }
        #send-button { 
            background: #007bff; 
            color: white; 
            border: none; 
            border-radius: 50%; 
            width: 50px; 
            height: 50px; 
            cursor: pointer; 
            font-size: 18px; 
            display: flex; 
            align-items: center; 
            justify-content: center; 
        }
        #send-button:hover { 
            background: #0056b3; 
        }
        #send-button:disabled { 
            background: #ccc; 
            cursor: not-allowed; 
        }
    </style>
</head>
<body>
    <div class="chat-container">
        <div class="chat-header">
            🤖 Gemini AI Chatbot
        </div>
        <div class="chat-box" id="chat-box">
            <div class="message bot-message">
                Hello! I'm your AI assistant. How can I help you today?
            </div>
        </div>
        <div class="input-container">
            <input type="text" id="user-input" placeholder="Type your message..." autocomplete="off">
            <button id="send-button" onclick="sendMessage()">➤</button>
        </div>
    </div>

    <script>
        const chatBox = document.getElementById('chat-box');
        const userInput = document.getElementById('user-input');
        const sendButton = document.getElementById('send-button');

        function addMessage(text, isUser) {
            const messageDiv = document.createElement('div');
            messageDiv.className = isUser ? 'message user-message' : 'message bot-message';
            messageDiv.textContent = text;
            chatBox.appendChild(messageDiv);
            chatBox.scrollTop = chatBox.scrollHeight;
        }

        async function sendMessage() {
            const message = userInput.value.trim();
            if (!message) return;

            addMessage(message, true);
            userInput.value = '';
            sendButton.disabled = true;

            const loadingDiv = document.createElement('div');
            loadingDiv.className = 'message bot-message';
            loadingDiv.textContent = 'Thinking...';
            loadingDiv.id = 'loading-message';
            chatBox.appendChild(loadingDiv);

            try {
                const response = await fetch('/chat', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ message: message })
                });

                document.getElementById('loading-message').remove();

                if (!response.ok) throw new Error('Server error');
                const data = await response.json();
                
                if (data.error) {
                    addMessage(`Error: ${data.error}`, false);
                } else {
                    addMessage(data.reply, false);
                }
            } catch (error) {
                if (document.getElementById('loading-message')) {
                    document.getElementById('loading-message').remove();
                }
                addMessage('Sorry, I encountered an error. Please try again.', false);
            } finally {
                sendButton.disabled = false;
            }
        }

        userInput.addEventListener('keypress', (e) => {
            if (e.key === 'Enter') sendMessage();
        });
    </script>
</body>
</html>
'''

@app.route('/')
def home():
    return render_template_string(HTML_TEMPLATE)

@app.route('/chat', methods=['POST'])
def chat():
    try:
        data = request.get_json()
        user_message = data.get('message', '').strip()
        
        if not user_message:
            return jsonify({'error': 'No message provided'}), 400
        
        response = model.generate_content(user_message)
        return jsonify({'reply': response.text})
        
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port)

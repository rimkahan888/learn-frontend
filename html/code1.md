<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ID Text Display Demo</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 600px;
            margin: 50px auto;
            padding: 20px;
            text-align: center;
        }
        #userInput {
            padding: 10px;
            width: 70%;
            font-size: 16px;
            border: 2px solid #ccc;
            border-radius: 5px;
        }
        button {
            padding: 10px 20px;
            background-color: #4285f4;
            color: white;
            border: none;
            border-radius: 5px;
            font-size: 16px;
            cursor: pointer;
            margin-left: 10px;
        }
        button:hover {
            background-color: #3367d6;
        }
        #displayArea {
            margin-top: 30px;
            padding: 20px;
            border: 1px dashed #4285f4;
            border-radius: 5px;
            min-height: 50px;
            font-size: 18px;
        }
        .input-group {
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <h1>Text Display Demo</h1>
    <p>Type something and see it appear below!</p>
    
    <div class="input-group">
        <input 
            type="text" 
            id="userInput" 
            placeholder="Type anything here..."
        >
        <button onclick="displayText()">Display Text</button>
    </div>
    
    <div id="displayArea">
        Your text will appear here...
    </div>

    <script>
        function displayText() {
            // 1. Get the text from the input field using its ID
            const userText = document.getElementById("userInput").value;
            
            // 2. Display it in the display area
            document.getElementById("displayArea").innerHTML = `
                <strong>You typed:</strong> "${userText}"
            `;
            
            // Bonus: Clear the input field after displaying
            document.getElementById("userInput").value = "";
        }
    </script>
</body>
</html>

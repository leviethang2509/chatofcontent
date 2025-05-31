<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8" />
  <title>Chat of Content</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <style>
    body {
      font-family: Arial, sans-serif;
      margin: 0;
      background-color: #f7f7f8;
      display: flex;
      flex-direction: column;
      height: 100vh;
    }

    .chat-header {
      padding: 1rem;
      background-color: #343541;
      color: white;
      text-align: center;
      font-size: 1.2rem;
      font-weight: bold;
    }

    .chat-container {
      flex: 1;
      overflow-y: auto;
      padding: 1rem;
      display: flex;
      flex-direction: column;
      gap: 1rem;
      background-color: white;
    }

    .message {
      max-width: 80%;
      padding: 0.75rem 1rem;
      border-radius: 8px;
      line-height: 1.4;
      white-space: pre-wrap;
      word-wrap: break-word;
    }

    .incoming {
      background-color: #e4e4e7;
      align-self: flex-start;
      color: #000;
    }

    .outgoing {
      background-color: #10a37f;
      color: white;
      align-self: flex-end;
    }

    .chat-input {
      padding: 1rem;
      background-color: white;
      border-top: 1px solid #ccc;
      display: flex;
      gap: 0.5rem;
      align-items: center;
      flex-wrap: wrap;
    }

    .chat-input input[type="text"],
    #fileSelect,
    #txtFileInput {
      flex: 1;
      padding: 0.75rem 1rem;
      border: 1px solid #ccc;
      border-radius: 6px;
      font-size: 1rem;
      min-width: 200px;
    }

    #fileSelect {
      cursor: pointer;
      background-color: #fff;
      font-family: Arial, sans-serif;
    }

    .chat-input button {
      background-color: #10a37f;
      border: none;
      color: white;
      padding: 0.75rem 1rem;
      border-radius: 6px;
      cursor: pointer;
      display: flex;
      align-items: center;
      justify-content: center;
    }

    .chat-input button:hover {
      background-color: #0e8a6b;
    }

    button.clear-history {
      float: right;
      background: none;
      color: #343541;
      border: none;
      cursor: pointer;
      font-size: 1.2rem;
      margin: 0.5rem 1rem 0 0;
    }

    @media (max-width: 600px) {
      .chat-input {
        flex-direction: column;
        align-items: stretch;
      }

      .chat-input input,
      .chat-input select,
      .chat-input input[type="file"],
      .chat-input button {
        width: 100%;
        margin-bottom: 0.5rem;
      }

      .clear-history {
        float: none;
        margin: 0.5rem auto;
        display: block;
      }
    }
  </style>
</head>
<body>

  <div class="chat-header">CHAT OF CONTENT</div>

  <div class="chat-container" id="chatContainer">
    <div class="message incoming">Xin chào! Nhập từ khóa hoặc chọn file để tìm kiếm.</div>
  </div>

  <div class="chat-input">
    <input
      type="text"
      id="userInput"
      placeholder="Nhập từ khóa..."
      onkeypress="if(event.key === 'Enter') sendMessage()"
      autocomplete="off"
    />
    <select id="fileSelect">
      <option value="">-- Chọn file --</option>
    </select>

    <input type="file" id="txtFileInput" accept=".txt" multiple />

    <button onclick="sendMessage()">
      <i class="fa fa-paper-plane" aria-hidden="true"></i>
    </button>

    <button onclick="convertTxtToJson()" title="Chuyển file TXT thành JSON">
      📄➡️📊
    </button>
  </div>

  <button class="clear-history" title="Xóa lịch sử chat" onclick="clearChatHistory()">🗑</button>

  <script>
    let fixesData = [];
    const existingMessages = new Set();

    window.onload = function () {
      restoreHistoryFromLocalStorage();

      fetch('fixes.json')
        .then(response => {
          if (!response.ok) throw new Error('Không thể tải fixes.json');
          return response.json();
        })
        .then(data => {
          fixesData = data;
          populateFileSelect();
        })
        .catch(err => {
          addMessage("Lỗi khi tải file JSON: " + err.message, "incoming");
        });
    };

    function populateFileSelect() {
      const select = document.getElementById('fileSelect');
      select.innerHTML = '<option value="">-- Chọn file --</option>';
      fixesData.forEach((file, idx) => {
        const option = document.createElement('option');
        option.value = idx;
        option.textContent = file.file;
        select.appendChild(option);
      });
    }

    function sendMessage() {
      const input = document.getElementById('userInput');
      const select = document.getElementById('fileSelect');

      let filename = input.value.trim();
      if (!filename && select.value !== "") {
        filename = select.options[select.selectedIndex].text.trim();
      }

      if (!filename) return;

      addMessage(filename, 'outgoing');

      input.value = '';
      select.value = '';

      if (fixesData.length === 0) {
        addMessage("Dữ liệu chưa được tải. Vui lòng thử lại sau.", "incoming");
        return;
      }

      const item = fixesData.find(i => i.file === filename);

      if (!item) {
        addMessage(`Không tìm thấy file "${filename}" trong dữ liệu.`, 'incoming');
      } else {
        const formattedContent = item.content.replace(/\r\n/g, '\n');
        addMessage(`📄 ${item.file}\n\n${formattedContent}`, 'incoming', true);
      }
    }

    function addMessage(text, type, isPreformatted = false) {
      const key = JSON.stringify({ text, type, isPreformatted });
      if (existingMessages.has(key)) return;
      existingMessages.add(key);

      const chatContainer = document.getElementById('chatContainer');
      const msgDiv = document.createElement('div');
      msgDiv.className = `message ${type}`;

      text = text.replace(/\\r\\n/g, '\n').replace(/\\n/g, '\n');

      if (isPreformatted) {
        const pre = document.createElement('pre');
        pre.style.whiteSpace = 'pre-wrap';
        pre.textContent = text;
        msgDiv.appendChild(pre);
      } else {
        msgDiv.innerHTML = text.replace(/\r\n|\n/g, "<br>");
      }

      chatContainer.appendChild(msgDiv);
      chatContainer.scrollTop = chatContainer.scrollHeight;

      saveHistoryToLocalStorage(text, type, isPreformatted);
    }

    function saveHistoryToLocalStorage(text, type, isPreformatted) {
      let history = JSON.parse(localStorage.getItem('chatHistory') || '[]');
      const key = JSON.stringify({ text, type, isPreformatted });
      const exists = history.some(msg => JSON.stringify(msg) === key);
      if (!exists) {
        history.push({ text, type, isPreformatted });
        localStorage.setItem('chatHistory', JSON.stringify(history));
      }
    }

    function restoreHistoryFromLocalStorage() {
      const history = JSON.parse(localStorage.getItem('chatHistory') || '[]');
      for (const msg of history) {
        addMessage(msg.text, msg.type, msg.isPreformatted);
      }
    }

    function clearChatHistory() {
      localStorage.removeItem('chatHistory');
      existingMessages.clear();
      document.getElementById('chatContainer').innerHTML = '';
    }

    function convertTxtToJson() {
      const fileInput = document.getElementById('txtFileInput');
      if (!fileInput.files.length) {
        alert('Vui lòng chọn ít nhất một file TXT để chuyển đổi.');
        return;
      }

      const txtFiles = Array.from(fileInput.files).filter(file => file.name.endsWith('.txt'));
      if (txtFiles.length === 0) {
        alert('Vui lòng chọn file có định dạng .txt');
        return;
      }

      const allFilesData = [];
      let filesProcessed = 0;

      txtFiles.forEach(file => {
        const reader = new FileReader();
        reader.onload = function (e) {
          allFilesData.push({
            file: file.name,
            content: e.target.result
          });

          filesProcessed++;

          if (filesProcessed === txtFiles.length) {
            const jsonString = JSON.stringify(allFilesData, null, 2);
            const blob = new Blob([jsonString], { type: 'application/json' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'merged_files.json';
            document.body.appendChild(a);
            a.click();

            setTimeout(() => {
              document.body.removeChild(a);
              URL.revokeObjectURL(url);
            }, 100);

            addMessage(`📄 Đã chuyển và tải file JSON gộp: merged_files.json`, 'incoming');
          }
        };

        reader.onerror = function () {
          alert(`Lỗi khi đọc file: ${file.name}`);
        };

        reader.readAsText(file);
      });
    }
  </script>

  <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css" rel="stylesheet" />
</body>
</html>

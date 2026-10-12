<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>RBG Batch-1520 Daily Cab Tracker</title>
  <style>
    :root {
      --primary: #1a73e8;
      --success: #34a853;
      --warning: #fbbc04;
      --danger: #ea4335;
      --bg: #f8f9fa;
      --card-bg: #ffffff;
      --text: #202124;
      --gray: #5f6368;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
      background-color: var(--bg);
      color: var(--text);
      margin: 0;
      padding: 16px;
      display: flex;
      flex-direction: column;
      align-items: center;
    }

    .container {
      width: 100%;
      max-width: 480px;
    }

    header {
      text-align: center;
      margin-bottom: 20px;
    }

    header h1 {
      font-size: 20px;
      margin: 0 0 4px 0;
      color: var(--primary);
    }

    header p {
      font-size: 13px;
      color: var(--gray);
      margin: 0;
    }

    .card {
      background: var(--card-bg);
      border-radius: 12px;
      padding: 16px;
      margin-bottom: 12px;
      box-shadow: 0 1px 3px rgba(0,0,0,0.1);
      display: flex;
      flex-direction: column;
      gap: 12px;
    }

    .member-info {
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    .member-name {
      font-weight: 600;
      font-size: 15px;
    }

    .member-meta {
      font-size: 12px;
      color: var(--gray);
    }

    .buttons-group {
      display: grid;
      grid-template-columns: 1fr 1fr 1fr;
      gap: 8px;
    }

    button {
      border: none;
      border-radius: 8px;
      padding: 10px 4px;
      font-size: 12px;
      font-weight: 600;
      cursor: pointer;
      color: white;
      transition: opacity 0.2s;
    }

    button:active {
      opacity: 0.8;
    }

    .btn-ontime { background-color: var(--success); }
    .btn-incab { background-color: var(--warning); color: #202124; }
    .btn-delayed { background-color: var(--danger); }

    .input-row {
      display: flex;
      gap: 8px;
      margin-top: 4px;
    }

    input {
      flex: 1;
      padding: 8px 12px;
      border: 1px solid #dadce0;
      border-radius: 6px;
      font-size: 13px;
      outline: none;
    }

    input:focus {
      border-color: var(--primary);
    }

    .status-badge {
      font-size: 11px;
      padding: 2px 8px;
      border-radius: 10px;
      background: #e8f0fe;
      color: var(--primary);
      font-weight: 500;
    }

    #loading {
      text-align: center;
      color: var(--gray);
      padding: 20px;
    }
  </style>
</head>
<body>

  <div class="container">
    <header>
      <h1>🚐 Daily Cab Check-In</h1>
      <p>RBG Batch-1520 (Shift: 10 AM - 7 PM)</p>
    </header>

    <div id="loading">Loading team members from sheet...</div>
    <div id="member-list"></div>
  </div>

  <script>
    const WEB_APP_URL = "https://script.google.com/a/macros/google.com/s/AKfycbxOeTg9Otc98MKDUFa4x9c3F8xVBi6cL8NTnENTwiCM94WUBjw24YaSaiid-ql8CQdMJg/exec";

    let teamMembers = [];

    async function loadTeamData() {
      const loadingEl = document.getElementById('loading');
      try {
        const response = await fetch(WEB_APP_URL);
        teamMembers = await response.json();
        
        if (teamMembers.error) {
          loadingEl.innerText = "Error: " + teamMembers.error;
          return;
        }
        
        renderUI();
      } catch (err) {
        console.error(err);
        loadingEl.innerText = "Failed to load team members from sheet.";
      }
    }

    function renderUI() {
      const listContainer = document.getElementById('member-list');
      document.getElementById('loading').style.display = 'none';
      
      listContainer.innerHTML = teamMembers.map(m => `
        <div class="card" id="card-${m.ldap}">
          <div class="member-info">
            <div>
              <div class="member-name">${m.name}</div>
              <div class="member-meta">ID: ${m.id} | LDAP: ${m.ldap}</div>
            </div>
            <span class="status-badge" id="badge-${m.ldap}">Pending</span>
          </div>

          <div class="input-row" id="inputs-${m.ldap}" style="display: none;">
            <input type="text" id="extra-${m.ldap}" placeholder="Enter ETA or Route No" />
          </div>

          <div class="buttons-group">
            <button class="btn-ontime" onclick="submitStatus('${m.ldap}', 'On Time / Reached', '')">1️⃣ On Time</button>
            <button class="btn-incab" onclick="promptExtra('${m.ldap}', 'In Cab')">2️⃣ In Cab</button>
            <button class="btn-delayed" onclick="promptExtra('${m.ldap}', 'Delayed')">3️⃣ Delayed</button>
          </div>
        </div>
      `).join('');
    }

    function promptExtra(ldap, statusType) {
      const inputRow = document.getElementById(`inputs-${ldap}`);
      const inputField = document.getElementById(`extra-${ldap}`);
      
      inputRow.style.display = 'flex';
      inputField.focus();
      
      const card = document.getElementById(`card-${ldap}`);
      const buttons = card.querySelectorAll('.buttons-group button');
      
      buttons.forEach(btn => {
        btn.onclick = () => {
          const val = inputField.value.trim();
          if (!val) {
            alert('Please enter ETA or Route Number first!');
            return;
          }
          if (statusType === 'In Cab') {
            submitStatus(ldap, 'In Cab', val, '');
          } else {
            submitStatus(ldap, 'Cab Delayed', '', val);
          }
          inputRow.style.display = 'none';
        };
      });
      inputField.placeholder = statusType === 'In Cab' ? 'Enter ETA (e.g., 5:35 AM)' : 'Enter Route / Cab No';
    }

    function submitStatus(ldap, status, eta = '', route = '') {
      const badge = document.getElementById(`badge-${ldap}`);
      badge.innerText = "Saving...";

      const payload = { ldap: ldap, status: status, eta: eta, route: route };

      fetch(WEB_APP_URL, {
        method: 'POST',
        mode: 'no-cors',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      })
      .then(() => {
        badge.innerText = status;
        badge.style.background = status.includes('On Time') ? '#ceead6' : (status.includes('In Cab') ? '#fef7e0' : '#fce8e6');
        badge.style.color = status.includes('On Time') ? '#137333' : (status.includes('In Cab') ? '#b06000' : '#c5221f');
      })
      .catch(err => {
        console.error(err);
        badge.innerText = "Error saving";
      });
    }

    window.onload = loadTeamData;
  </script>
</body>
</html>

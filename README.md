# Radek-website
<!DOCTYPE html>
<html lang="cs" >
<head>
  <meta charset="UTF-8" />
  <title>Hod mincí | Neal.fun styl</title>
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@600&display=swap');

    * {
      box-sizing: border-box;
    }
    body {
      margin: 0;
      background: linear-gradient(135deg, #667eea, #764ba2);
      font-family: 'Inter', sans-serif;
      color: white;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      height: 100vh;
      text-align: center;
      user-select: none;
    }
    h1 {
      font-size: 5rem;
      margin-bottom: 0.5em;
      letter-spacing: -0.05em;
      text-shadow: 0 4px 8px rgba(0,0,0,0.3);
    }
    #coin-result {
      font-size: 8rem;
      font-weight: 700;
      margin: 0.5em 0 2em;
      text-shadow: 0 5px 10px rgba(0,0,0,0.4);
      min-height: 1em;
      transition: transform 0.3s ease;
    }
    button {
      font-size: 2.5rem;
      padding: 1em 3em;
      border: none;
      border-radius: 12px;
      background: white;
      color: #764ba2;
      font-weight: 700;
      cursor: pointer;
      box-shadow: 0 6px 15px rgba(0,0,0,0.3);
      transition: transform 0.2s ease, box-shadow 0.2s ease;
      user-select: none;
    }
    button:active {
      transform: scale(0.95);
      box-shadow: 0 3px 8px rgba(0,0,0,0.3);
    }
    button:hover {
      background: #f0e9ff;
    }
    /* Animace při změně výsledku */
    .flip {
      animation: flipCoin 0.6s ease forwards;
    }
    @keyframes flipCoin {
      0% { transform: rotateY(0deg); }
      50% { transform: rotateY(180deg); }
      100% { transform: rotateY(0deg); }
    }
  </style>
</head>
<body>
  <h1>Hod mincí</h1>
  <div id="coin-result">?</div>
  <button id="flip-btn">Hoď mincí</button>

  <script>
    const resultEl = document.getElementById('coin-result');
    const btn = document.getElementById('flip-btn');

    btn.addEventListener('click', () => {
      // Přidej třídu flip pro animaci
      resultEl.classList.add('flip');

      // Po půl sekundě (konec animace) změň výsledek
      setTimeout(() => {
        const vysledek = Math.random() < 0.5 ? "🅿️ Panna" : "🦅 Orel";
        resultEl.textContent = vysledek;
        resultEl.classList.remove('flip');
      }, 300); // polovina animace
    });
  </script>
</body>
</html>

# Radek-website
<!DOCTYPE html>
<html lang="cs">
<head>
  <meta charset="UTF-8" />
  <title>Hod mincí</title>
  <style>
    body {
      background-color: #add8e6; /* světle modrá */
      font-family: Arial, sans-serif;
      text-align: center;
      padding: 50px;
      color: #333;
    }
    h1 {
      margin-bottom: 40px;
    }
    #coin-result {
      font-size: 48px;
      margin: 30px 0;
      font-weight: bold;
    }
    button {
      background-color: #0077cc;
      color: white;
      border: none;
      padding: 15px 30px;
      font-size: 24px;
      border-radius: 8px;
      cursor: pointer;
      transition: background-color 0.3s ease;
    }
    button:hover {
      background-color: #005fa3;
    }
  </style>
</head>
<body>
  <h1>Hod mincí</h1>
  <button onclick="hodMince()">Hoď mincí</button>
  <div id="coin-result">?</div>

  <script>
    function hodMince() {
      // Math.random() < 0.5 → Panna, jinak Orel
      const vysledek = Math.random() < 0.5 ? "Panna" : "Orel";
      document.getElementById("coin-result").textContent = vysledek;
    }
  </script>
</body>
</html>
